# Linux内核new_sync_write函数深入分析

## 1. 函数概述

`new_sync_write`是Linux内核VFS（虚拟文件系统）层的核心写入函数，位于`fs/read_write.c`文件中。该函数是同步写操作的统一入口，将传统的write系统调用转换为基于迭代器的写操作。

```c
static ssize_t new_sync_write(struct file *filp, const char __user *buf, 
                             size_t len, loff_t *ppos)
```

## 2. 函数执行流程详解

### 2.1 初始化阶段

#### 2.1.1 初始化同步kiocb结构
```c
init_sync_kiocb(&kiocb, filp);
```

`init_sync_kiocb`函数的实现（include/linux/fs.h）：
```c
static inline void init_sync_kiocb(struct kiocb *kiocb, struct file *filp)
{
    *kiocb = (struct kiocb) {
        .ki_filp = filp,
        .ki_flags = filp->f_iocb_flags,
        .ki_ioprio = get_current_ioprio(),
    };
}
```

关键点：
- **ki_complete未设置**：这是判断同步IO的关键标志（`is_sync_kiocb`函数通过检查`ki_complete == NULL`来判断）
- **ki_flags继承文件标志**：可能包含`IOCB_DSYNC`、`IOCB_SYNC`等标志
- **ki_ioprio设置IO优先级**

#### 2.1.2 设置写入位置
```c
kiocb.ki_pos = (ppos ? *ppos : 0);
```

#### 2.1.3 初始化IO迭代器
```c
iov_iter_ubuf(&iter, ITER_SOURCE, (void __user *)buf, len);
```

该函数创建一个用户缓冲区迭代器：
- `ITER_SOURCE`：表示数据源（写操作）
- 设置单个段（nr_segs = 1）
- 直接指向用户空间缓冲区

### 2.2 核心写入操作

```c
ret = filp->f_op->write_iter(&kiocb, &iter);
```

直接调用文件系统的`write_iter`方法，以ext4为例：

#### 2.2.1 ext4_file_write_iter流程

1. **路径选择**：
   ```c
   if (iocb->ki_flags & IOCB_DIRECT)
       return ext4_dio_write_iter(iocb, from);  // Direct I/O
   else
       return ext4_buffered_write_iter(iocb, from);  // Buffered I/O
   ```

2. **Buffered I/O路径**（最常见）：
   
   `ext4_buffered_write_iter`的核心流程：
   ```c
   inode_lock(inode);
   ret = generic_perform_write(iocb, from);
   inode_unlock(inode);
   return generic_write_sync(iocb, ret);
   ```

#### 2.2.2 generic_perform_write详解

这是实际的页缓存写入函数（mm/filemap.c）：

```c
ssize_t generic_perform_write(struct kiocb *iocb, struct iov_iter *i)
{
    do {
        // 1. 调用文件系统的write_begin准备页面
        status = a_ops->write_begin(iocb, mapping, pos, bytes,
                                   &folio, &fsdata);
        
        // 2. 从用户空间复制数据到页缓存
        copied = copy_folio_from_iter_atomic(folio, offset, bytes, i);
        
        // 3. 调用文件系统的write_end完成写入
        status = a_ops->write_end(iocb, mapping, pos, bytes, copied,
                                 folio, fsdata);
        
        // 4. 平衡脏页
        balance_dirty_pages_ratelimited(mapping);
    } while (iov_iter_count(i));
}
```

**关键点：数据只写入页缓存，不会立即写入磁盘！**

#### 2.2.3 generic_write_sync的同步机制

```c
static inline ssize_t generic_write_sync(struct kiocb *iocb, ssize_t count)
{
    if (iocb_is_dsync(iocb)) {
        // 需要同步时调用fsync
        int ret = vfs_fsync_range(iocb->ki_filp,
                iocb->ki_pos - count, iocb->ki_pos - 1,
                (iocb->ki_flags & IOCB_SYNC) ? 0 : 1);
        if (ret)
            return ret;
    } else if (iocb->ki_flags & IOCB_DONTCACHE) {
        // 异步刷新脏页
        filemap_fdatawrite_range_kick(mapping, iocb->ki_pos - count,
                                     iocb->ki_pos - 1);
    }
    return count;
}
```

### 2.3 同步判断条件

`iocb_is_dsync`的判断逻辑：
```c
static inline bool iocb_is_dsync(const struct kiocb *iocb)
{
    return (iocb->ki_flags & IOCB_DSYNC) ||
           IS_SYNC(iocb->ki_filp->f_mapping->host);
}
```

触发同步的条件：
1. **文件以O_DSYNC/O_SYNC打开**：设置了IOCB_DSYNC标志
2. **文件系统挂载为sync模式**：mount -o sync
3. **inode设置了S_SYNC标志**：通过chattr +S设置

### 2.4 错误处理

```c
BUG_ON(ret == -EIOCBQUEUED);
```

同步写入不应该返回`-EIOCBQUEUED`（表示异步IO已排队），如果返回则触发内核panic。

## 3. 数据写入层次分析

### 3.1 正常的Buffered I/O写入路径

1. **用户空间** → **页缓存（Page Cache）**
   - 数据通过`copy_folio_from_iter_atomic`复制到页缓存
   - 页面被标记为脏页（dirty）
   - **此时write系统调用即可返回**

2. **页缓存** → **磁盘**（异步）
   - 由内核的writeback机制控制
   - 触发条件：
     - 脏页达到阈值（dirty_ratio）
     - 定期刷新（dirty_writeback_centisecs，默认5秒）
     - 内存压力
     - 显式sync/fsync调用

### 3.2 同步写入路径（O_SYNC/O_DSYNC）

1. **用户空间** → **页缓存**
   - 同上，先写入页缓存

2. **页缓存** → **磁盘**（同步）
   - `generic_write_sync`调用`vfs_fsync_range`
   - ext4_sync_file执行：
     ```c
     // 等待数据写入磁盘
     file_write_and_wait_range(file, start, end);
     
     // 等待元数据日志提交
     ext4_fsync_journal(inode, datasync, &needs_barrier);
     
     // 必要时发出磁盘刷新命令
     if (needs_barrier)
         blkdev_issue_flush(inode->i_sb->s_bdev);
     ```
   - **write系统调用等待数据真正写入磁盘后才返回**

## 4. 关键结论

### 4.1 new_sync_write的同步特性

**默认情况下，new_sync_write只是将数据写入页缓存就返回，并不等待数据写入磁盘。**

"sync"的含义是指：
- **同步API调用**：相对于异步IO（AIO），调用会等待操作完成
- **不是指数据同步到磁盘**：除非显式指定O_SYNC/O_DSYNC

### 4.2 数据持久化保证

| 场景 | 写入目标 | 系统调用返回时机 | 数据持久化保证 |
|-----|---------|----------------|---------------|
| 普通write | 页缓存 | 数据复制到页缓存后 | 无（可能丢失） |
| O_SYNC write | 磁盘 | 数据和元数据写入磁盘后 | 完全持久化 |
| O_DSYNC write | 磁盘 | 数据写入磁盘后 | 数据持久化 |
| write + fsync | 磁盘 | write：页缓存<br>fsync：磁盘 | 完全持久化 |

### 4.3 性能影响

1. **普通写入（默认）**：
   - 高性能：只需内存操作
   - 低延迟：微秒级
   - 风险：系统崩溃可能丢失数据

2. **同步写入（O_SYNC）**：
   - 低性能：需要磁盘IO
   - 高延迟：毫秒级
   - 安全：数据不会丢失

### 4.4 实际应用建议

1. **普通应用**：使用默认的buffered I/O，依赖操作系统的writeback机制
2. **数据库等关键应用**：
   - 使用O_DIRECT绕过页缓存
   - 或使用write + fsync组合
   - 或使用O_SYNC/O_DSYNC
3. **日志文件**：可以使用O_APPEND + 定期fsync

## 5. 源码追踪路径总结

```
new_sync_write (fs/read_write.c)
  ├── init_sync_kiocb (include/linux/fs.h)
  ├── iov_iter_ubuf (include/linux/uio.h)
  ├── filp->f_op->write_iter
  │   └── ext4_file_write_iter (fs/ext4/file.c)
  │       └── ext4_buffered_write_iter
  │           ├── generic_perform_write (mm/filemap.c)
  │           │   ├── a_ops->write_begin
  │           │   ├── copy_folio_from_iter_atomic
  │           │   └── a_ops->write_end
  │           └── generic_write_sync (include/linux/fs.h)
  │               └── vfs_fsync_range (fs/sync.c)
  │                   └── file->f_op->fsync
  │                       └── ext4_sync_file (fs/ext4/fsync.c)
  │                           ├── file_write_and_wait_range
  │                           └── blkdev_issue_flush
  └── 更新文件位置 (*ppos = kiocb.ki_pos)
```

## 总结

`new_sync_write`函数是Linux内核VFS层的核心写入接口，它将传统的write系统调用转换为基于迭代器的现代IO模型。**在默认情况下，该函数仅将数据写入页缓存就返回，不会等待数据写入磁盘**。只有在文件以O_SYNC/O_DSYNC模式打开，或文件系统以sync模式挂载时，才会等待数据真正持久化到磁盘。

这种设计在性能和数据安全性之间取得了平衡：
- 默认的异步写入提供了优秀的性能
- 需要数据安全保证的应用可以选择同步模式
- 操作系统的writeback机制确保数据最终会写入磁盘

理解这一机制对于开发高性能、高可靠性的存储系统至关重要。