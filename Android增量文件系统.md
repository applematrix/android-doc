# Android增量文件系统

基于Android R版本的AOSP代码分析

## 增量文件系统的代码路径

增量文件在Android中用户空间态的代码位置在/system/core/incremental_delivery

## 系统是否支持增量文件系统

通过filesystem来判断。在/proc/filesystems中，系统会列出所有的文件系统，如果其中具有incremental-fs的文件，则证明系统支持增量文件系统

## 文件系统的feature

通过是否判断具有/sys/fs/incremental-fs目录。在/sys/fs/incremental-fs/features中保存有关于增量文件系统的feature。一般在/sys/fs/incremental-fs/features目录下具有一个corefs的文件，标明incfs为核心的文件系统。

## 增量文件系统的数据结构

### IncFsControl

```C++
struct IncFsControl final {
    IncFsFd cmd;
    IncFsFd pendingReads;
    IncFsFd logs;
    constexpr IncFsControl(IncFsFd cmd, IncFsFd pendingReads, IncFsFd logs)
          : cmd(cmd), pendingReads(pendingReads), logs(logs) {}
};
```

IncFsControl数据结构中包含cmd：pendingReads：logs：

### IncFsSpan

```C++
typedef struct {
    const char* data;
    IncFsSize size;
} IncFsSpan;
```

增量文件系统读取时使用的数据结构，包括数据指针和size大小。

### IncFsFileId

```C++
typedef struct {
    union {
        char data[16];
        int64_t for_alignment;
    };
} IncFsFileId;
```

用于表示Fs文件的id，id为16个字节的ascii码。如果所有ascii码均为-1，则表示非法的FileId

### IncFsFeature

```c++
typedef enum {

  INCFS_FEATURE_NONE = 0,

  INCFS_FEATURE_CORE = 1,

} IncFsFeatures;
```

incfs的特性，主要就是2个，core类型和none类型



### IncFsDataBlock

```C++
typedef struct {

  IncFsFd fileFd;//文件句柄

  IncFsBlockIndex pageIndex;//block的下标

  IncFsCompressionKind compression; //压缩的方式

  IncFsBlockKind kind;// 数据块的类型

  uint32_t dataSize; // 数据的大小

  const char* data; // 数据的指针

} IncFsDataBlock;
```

IncFsDataBlock代表数据块。



## cmd到root的转换

在IncFsControl中存储了一个cmd的fd，系统拿到fd时需要进行fd到文件目录的转化。转化的过程如下：

1. 首先通过fd获取到对应的cmd文件。直接去读取/proc/self/fd/下对应的文件fd的值，由于该路径下的文件是link文件所以直接读取该目录的连接后，可以获取到对应的cmd目录

   ```shell
   sargo:/proc/self/fd $ ls -la
   total 0
   dr-x------ 2 shell shell  0 2021-04-14 18:47 .
   dr-xr-xr-x 9 shell shell  0 2021-04-14 17:23 ..
   lrwx------ 1 shell shell 64 2021-04-14 18:47 0 -> /dev/pts/0
   lrwx------ 1 shell shell 64 2021-04-14 18:47 1 -> /dev/pts/0
   lrwx------ 1 shell shell 64 2021-04-14 18:47 10 -> /dev/tty
   lrwx------ 1 shell shell 64 2021-04-14 18:47 2 -> /dev/pts/0
   ```

2. 将获取到的cmd文件截取文件后取到最后一级目录，如上图中fd为10的cmd转换后即为/dev

3. 在incfs的fd文件，必须是以".pending_reads"结尾的文件，因此只cmd到root的转换实际上就是找到对应的文件所在的父目录。

## 判断一个文件是否是incFs的文件

获取文件的属性后，在文件的属性的type字段将标识一个文件是否为incFs的文件（目录）：

```C++
bool isIncFsFd(int fd) {
  struct statfs fs = {};
  if (::fstatfs(fd, &fs) != 0) {
    PLOG(WARNING) << __func__ << "(): could not fstatfs fd " << fd;
    return false;
  }
  return fs.f_type == (decltype(fs.f_type))INCFS_MAGIC_NUMBER;
}
```

如果文件的类型为incFs的魔数定义，则该文件即为incFs的文件，其中魔数的定义如下：

\#define INCFS_MAGIC_NUMBER (0x5346434e49ul)



## 文件的metadata

## 增量文件的挂载（IncFs_Mount）

增量文件系统的挂载，通过IncFs_Mount将源路径挂载到一个目标路径下，核心的处理逻辑如下：

```C++
    const auto opts = makeMountOptionsString(options);
    if (::mount(backingPath, targetDir, INCFS_NAME, MS_NOSUID | MS_NODEV | MS_NOATIME,
                opts.c_str())) {
        PLOG(ERROR) << "[incfs] Failed to mount IncFS filesystem: " << targetDir
                    << " errno: " << errno;
        return nullptr;
    }

    if (!restoreconControlFiles(targetDir)) {
        return nullptr;
    }

    auto control = makeControl(targetDir);
    if (control == nullptr) {
        return nullptr;
    }
    return control;
```

其中backingPath为源路径，targetDir为目标路径，INCFS_NAME为：incremental-fs，挂载选项



## 增量文件的打开(IncFs_Open)

打开增量文件系统时，传递一个文件路径给增量文件系统的IncFs_Open函数，该函数将打开一个IncFsControl的对象，通过该对象可以操作增量文件系统。

```C++
IncFsControl* IncFs_Open(const char* dir) {
    auto root = registry().rootFor(dir);
    if (root.empty()) {
        errno = EINVAL;
        return nullptr;
    }
    return makeControl(android::incfs::details::c_str(root));
}
```

任何增量文件的相关数据，都在根目录下，所以在open时，首先通过rootFor找到对应目录的root，再在root中通过对应的配置文件创建出IncFsControl的对象供后续操作。IncFsControl的数据，IncFsControl中将包含以下数据：

1、cmd。指向root下的.pending_reads文件的句柄

2、pendingReads。同样是指向root下的.pending_reads文件的句柄

3、log。指向root下的.log



## 增量文件的读取

