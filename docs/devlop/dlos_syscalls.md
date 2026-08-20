# DoglinkOS-2nd 系统调用文档

`DoglinkOS-2nd` 使用 `int $0x80` 进行系统调用。

- 系统调用号通过 `rax` 传递
- 参数主要通过 `rdi`、`rsi`、`rdx`、`rcx`、`r8`、`r9`、`r10` 传递
- 返回值位置并不完全统一，需要按各 syscall 的约定读取

## 系统调用列表

`NUM_SYSCALLS` 当前为 `23`。

### sys_test (0)

参数：无

返回值：无

由内核输出 `test syscall`。

### sys_write (1)

参数：3 个

`rdi` `fd` 目标文件描述符

`rsi` `buf` 待写入数据指针

`rcx` `len` 数据长度

返回值：无

### sys_fork (2)

参数：无

返回值（`rcx`）：父进程中为子进程 `pid`，子进程中为 `0`

### sys_exec (3)

参数：2 个

`rdi` `path` 目标程序路径指针

`rcx` `len` 路径长度

返回值：`noreturn` 或无

### sys_exit (4)

参数：无

返回值：`noreturn`

### sys_read (5)

参数：无

返回值（`rcx`）：读取到的 1 字节值；若输入缓冲区为空则为 `0xff`

从标准输入读取一个字节。

### sys_setfsbase (6)

参数：1 个

`rdi` `fs_base` 要设置的 `FS base`

返回值：无

### sys_brk (7)

参数：1 个

`rdi` `brk` 新值；为 `0` 时仅查询不修改

返回值（`rsi`）：原来的 `brk`

### sys_waitpid (8)

参数：1 个

`rdi` `pid` 目标进程 ID

返回值：无

阻塞当前进程，直到目标进程结束。

### sys_getpid (9)

参数：无

返回值（`rcx`）：当前进程 `pid`

### sys_getticks (10)

参数：无

返回值（`rcx`）：当前内核 tick 计数

### sys_info (11)

参数：1 个

`rdi` `type` 信息类型

```c
#define INFO_CONSOLE_COLS 0
#define INFO_CONSOLE_ROWS 1
#define INFO_PID 2
#define INFO_TICKS 3
#define INFO_CONSOLE_ECHO_OFF 4
#define INFO_CONSOLE_ECHO_ON 5
#define INFO_FRAMEBUFFER_WIDTH 6
#define INFO_FRAMEBUFFER_HEIGHT 7
#define INFO_FRAMEBUFFER_ADDR 8
#define INFO_FRAMEBUFFER_PITCH 9
#define INFO_RETURN_TO_RING0 10
```

返回值（`rcx`）：

- 普通查询返回对应值
- `INFO_CONSOLE_ECHO_OFF` / `INFO_CONSOLE_ECHO_ON` 仅设置回显，返回 `0`
- `INFO_RETURN_TO_RING0` 仅允许 `pid == 0` 的进程使用，成功返回 `0`，失败返回 `u64::MAX`
- 未知 `type` 返回 `u64::MAX`

`INFO_RETURN_TO_RING0` 成功时会把返回现场的 `cs` / `ss` 切换到 Ring 0。

### sys_open (12)

参数：3 个

`rdi` `path` 文件路径指针

`rcx` `len` 路径长度

`r10` `do_create` 非零表示不存在时创建

返回值（`rsi`）：文件描述符，失败时为 `u64::MAX`

### sys_read2 (13)

参数：3 个

`rdi` `buf` 目标缓冲区指针

`rsi` `fd` 文件描述符

`rcx` `len` 读取长度

返回值：无

按 `read_exact` 语义读取。

### sys_seek (14)

参数：3 个

`rdi` `from` 起始位置类型

```c
#define SEEK_CUR 0
#define SEEK_END 1
#define SEEK_SET 2
```

`rsi` `fd` 文件描述符

`rcx` `offset` 偏移量

返回值（`r10`）：新的绝对位置

其中 `SEEK_CUR` 与 `SEEK_END` 会把 `rcx` 作为有符号偏移处理。

### sys_close (15)

参数：1 个

`rsi` `fd` 文件描述符

返回值：无

### sys_remove (16)

参数：2 个

`rdi` `path` 文件路径指针

`rcx` `len` 路径长度

返回值：无

### sys_mount (17)

参数：5 个

`rdi` `mountpoint` 挂载点路径指针

`rcx` `len` 挂载点路径长度

`rsi` `device_type` 设备类型

`rdx` `device_index` 设备索引

`r9` `partition_index` 分区索引

设备类型：

```c
#define MOUNT_AHCI 0
#define MOUNT_NVME 1
#define MOUNT_USB 2
```

返回值（`r10`）：成功为 `0`，`device_type` 非法或设备/分区不存在时为 `u64::MAX`

当前挂载逻辑使用 FAT 文件系统。

### sys_opendir (18)

参数：2 个

`rdi` `path` 目录路径指针

`rcx` `len` 路径长度

返回值（`rsi`）：目录句柄，失败时为 `u64::MAX`

### sys_getdents (19)

参数：3 个

`rdi` `buf` `DirEntry` 缓冲区指针

`rsi` `dirfd` 目录句柄

`rcx` `len` 缓冲区可容纳的 `DirEntry` 数量

返回值（`r10`）：写入的目录项数量，失败时为 `u64::MAX`

`DirEntry` 的具体布局如下：

```c
#define DIRENT_NAME_CAP 255

typedef struct DirEntry {
    uint8_t is_dir;
    uint8_t name[DIRENT_NAME_CAP];
} DirEntry;
```

字段说明：

- `is_dir`：`0` 表示普通文件，`1` 表示目录
- `name`：固定长度 `255` 字节的文件名缓冲区

关于 `name` 的约定：

- 内核会把文件名按字节拷贝进 `name`
- 最多写入 `254` 个字节，最后至少保留一个 `0` 字节
- 因此可以按 C 风格字符串读取
- 文件名若超过 `254` 字节，会被截断

使用方式：

- 用户缓冲区应按 `DirEntry[len]` 分配
- `sys_getdents` 每次最多写入 `len` 个目录项
- 返回的数量是本次实际写入的 `DirEntry` 个数，不是字节数

### sys_closedir (20)

参数：1 个

`rsi` `dirfd` 目录句柄

返回值：无

### sys_ipc (21)

参数：

`rdi` `cmd` IPC 子命令

返回值（`rax`）：`isize` 风格返回码；成功时通常返回非负值，失败时返回负 errno

```c
#define IPC_CMD_CREATE  0
#define IPC_CMD_SEND    1
#define IPC_CMD_RECV    2
#define IPC_CMD_CLOSE   3
#define IPC_CMD_DUP     4
#define IPC_CMD_BIND    5
#define IPC_CMD_CONNECT 6
#define IPC_CMD_ACCEPT  7
```

各子命令约定如下。

#### IPC_CMD_CREATE

创建一对 channel handle。

返回值（`rax`）：第一个 handle

附加返回值（`rdx`）：第二个 handle

失败返回负 errno，如 `-24`（`EMFILE`）。

#### IPC_CMD_SEND

参数：

`rsi` `handle`

`rdx` `buf`

`rcx` `len`

返回值（`rax`）：成功时返回发送字节数

可能的错误：`-9`、`-11`、`-32`、`-90`

#### IPC_CMD_RECV

参数：

`rsi` `handle`

`rdx` `buf`

`rcx` `len`

返回值（`rax`）：

- 大于等于 `0`：实际接收字节数
- `0`：对端已关闭且当前无消息
- `-11`：当前无消息但对端未关闭

#### IPC_CMD_CLOSE

参数：

`rsi` `handle`

返回值（`rax`）：`0` 成功，否则负 errno

#### IPC_CMD_DUP

参数：

`rsi` `handle`

返回值（`rax`）：新 handle，失败时为负 errno

#### IPC_CMD_BIND

参数：

`rsi` `name_ptr`

`rdx` `name_len`

返回值（`rax`）：listener handle，失败时为负 errno

可能的错误：`-17`、`-22`、`-24`

#### IPC_CMD_CONNECT

参数：

`rsi` `name_ptr`

`rdx` `name_len`

返回值（`rax`）：客户端 handle，失败时为负 errno

可能的错误：`-2`、`-22`、`-24`

#### IPC_CMD_ACCEPT

参数：

`rsi` `listener_handle`

返回值（`rax`）：服务端 channel handle，失败时为负 errno

可能的错误：`-9`、`-11`、`-24`

### sys_read3 (22)

参数：3 个

`rdi` `buf` 目标缓冲区指针

`rsi` `fd` 文件描述符

`rcx` `len` 最大读取长度

返回值（`r10`）：实际读取字节数

与 `sys_read2` 的区别是该接口按普通 `read` 语义工作，不要求填满整个缓冲区。
