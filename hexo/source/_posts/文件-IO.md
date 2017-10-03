---
title: 文件 IO
date: 2017-10-03 11:31:04
categories: "Linux"
tags: [Linux, c]
keyworlds: Linux, IO, 文件
description:
---

>文件描述符

类型：一个非负正整数
范围：0 ~ OPEN_MAX - 1

- OPEN_MAX 现在已经被取消
- ulimit -a 查看系统最大打开文件数
- ulimit -n 最大数目  修改最大打开文件数
    
常用惯例：

- 0 -> 标准输入   (STDIN_FILENO)
- 1 -> 标准输出   (STDOUT_FILENO)
- 2 -> 标准错误   (STDERR_FILENO)

> int open(const char* filepath, int oflag, ...)
  int openat(int fd, const char *filepath, int oflag, ... )

返回的fd一定是未使用的文件描述符中最小的

```c
OFLAG:
    /* 有且仅有一个 */
    #define O_RDONLY    0       /* +1 == FREAD */
    #define O_WRONLY    1       /* +1 == FWRITE */
    #define O_RDWR      2       /* +1 == FREAD|FWRITE */
    #define O_EXEC          _FEXECSRCH
    #define O_SEARCH        _FEXECSRCH
    
    /* 可选 */
    #define O_APPEND    _FAPPEND
    #define O_CREAT     _FCREAT
    #define O_TRUNC     _FTRUNC /* 若文件存在，长度截为0 */
    #define O_EXCL      _FEXCL
    #define O_SYNC      _FSYNC

    #define O_NONBLOCK  _FNONBLOCK
    #define O_NOCTTY    _FNOCTTY
    #define O_BINARY    _FBINARY
    #define O_TEXT      _FTEXT
    #define O_CLOEXEC   _FNOINHERIT
    #define O_DIRECT        _FDIRECT
    #define O_NOFOLLOW      _FNOFOLLOW
    #define O_DSYNC         _FSYNC
    #define O_RSYNC         _FSYNC
    #define O_DIRECTORY     _FDIRECTORY
   
     /* O_NDELAY    _FNDELAY    set in include/fcntl.h */
    /*  O_NDELAY    _FNBIO      set in include/fcntl.h */
```

openat 中的fd

1. filepath 是绝对路径， 忽略fd.
2. filepath 是相对路径， fd是相对路径名在文件系统中的开始地址。通过打开目录来获取filepath
3. filepath 是相对路径， fd为AT_FDCWD，路径名在当前工作目录中获取。

> int create(const char *filepath, mode_t mode)

等效于 open(filepath, O_WRONLY|O_CREAT|0_TRUNC, mode);

> int close(int fd)   include<unistd.h>

在一个进程结束时，内核自动关闭这个进程打开的所有文件。

> off_t lseek(int fd, off_t offset, int whence)

```C
OFFSET:
    # define    SEEK_SET    0   /* 距开始 */
    # define    SEEK_CUR    1   /* 距当前 */
    # define    SEEK_END    2   /* 距末尾 */
```

*** 如果 fd 是管道、FIFO、socket lseek返回-1， 置errorno 为ESPIPE。偏移量有可能为负， 因此在检测lseek是否正确时应该使用 ==-1 来判断而不能使用 <0来判断****
当lseek超过文件长度时， 对该文件的下一次写将加长该文件，并在文件中构成一个空洞。空洞都被置为0.空洞不要求在磁盘上占用空间（不分配磁盘块）。


