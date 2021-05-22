# Android中的vold分析

本文基于Android R版本的代码分析	

​	Android中的vold是一个native层的用户空间的守护进程，实际上是volume+daemon的缩写，顾名思义是管理系统中的存储卷的守护进程，vold由init在开机阶段根据init.rc文件解析后启动，之后以binder服务的形式驻留在系统中响应外部的请求。

​	vold目前包括两部分的内容：VolumeManager、NetlinkManager，前者主要负责存储卷的管理，后者负责Netlink事件的管理，负责接收内核的相关事件后，根据事件信息进行内部的volume相关的信息管理。



## vold的init启动配置

​	vold通过init.rc的配置启动：

```c++
service vold /system/bin/vold \
        --blkid_context=u:r:blkid:s0 --blkid_untrusted_context=u:r:blkid_untrusted:s0 \
        --fsck_context=u:r:fsck:s0 --fsck_untrusted_context=u:r:fsck_untrusted:s0
    class core
    ioprio be 2
    writepid /dev/cpuset/foreground/tasks
    shutdown critical
    group root reserved_disk
```

在（system\vold\vold.rc）中配置了vold作为服务启动的参数，其中主要就是一些上下文的设置，系统开机后将根据以上配置启动vold运行。



## vold进程的启动

vold进程启动的过程如下所示：

![](images/storage/vold-vold%E8%BF%9B%E7%A8%8B%E5%90%AF%E5%8A%A8.png)

​	init进程启动后，通过init.rc脚本直接拉起vold进程执行，在vold的main函数入口中，进入后，进程主要做以下动作：

1. 创建/dev/block/vold目录。后面的卷会在该目录下建立对应的设备节点，所以在一开始需要将设备节点文件创建出来
2. 通过VolumeManager的单例模式创建出VolumeManager实例
3. 通过NetlinkManager单例创建NetlinkManager实例。NetlinkManager是接收事件处理的入口，volume相关的事件通过NetlinkManager接收后发送给VolumeManager处理。
4. 启动VolumeManager
5. 处理VolumeManager的配置
6. 启动NetLinkManager
7. 通过VoldNativeService将binder服务publish到ServiceManager中，从这个阶段开始，系统可以通过binder服务访问vold
8. 通过coldboot函数执行冷启动的处理。



### VolumeManager启动

volumeManager启动的整体流程如下所示：

![](images/storage/vold-volumanager%E7%9A%84start%E6%95%B4%E4%BD%93%E8%BF%87%E7%A8%8B.png)

volumeManager是一个事件驱动型的服务，首先做清理动作：

1. unmountAll卸载掉所有的挂载点
2. 调用Devmapper的destoryAll函数，删除所有的dm设备
3. 移除挂载的所有的环回设备

然后VM会默认创建出一个内置的emulated的存储卷。最后更新虚拟的磁盘（受配置项控制，不做详细介绍）



#### unmountAll卸载所有挂载点

![](images/storage/vold-volume%E5%8D%B8%E8%BD%BD%E6%89%80%E6%9C%89%E7%9A%84%E6%8C%82%E8%BD%BD.png)

```c++
int VolumeManager::unmountAll() {
    std::lock_guard<std::mutex> lock(mLock);
    ATRACE_NAME("VolumeManager::unmountAll()");

    // First, try gracefully unmounting all known devices
    for (const auto& vol : mInternalEmulatedVolumes) {
        vol->unmount();
    }
    for (const auto& disk : mDisks) {
        disk->unmountAll();
    }

    // Worst case we might have some stale mounts lurking around, so
    // force unmount those just to be safe.
    FILE* fp = setmntent("/proc/mounts", "re");
    if (fp == NULL) {
        PLOG(ERROR) << "Failed to open /proc/mounts";
        return -errno;
    }

    // Some volumes can be stacked on each other, so force unmount in
    // reverse order to give us the best chance of success.
    std::list<std::string> toUnmount;
    mntent* mentry;
    while ((mentry = getmntent(fp)) != NULL) {
        auto test = std::string(mentry->mnt_dir);
        if ((StartsWith(test, "/mnt/") &&
#ifdef __ANDROID_DEBUGGABLE__
             !StartsWith(test, "/mnt/scratch") &&
#endif
             !StartsWith(test, "/mnt/vendor") && !StartsWith(test, "/mnt/product") &&
             !StartsWith(test, "/mnt/installer") && !StartsWith(test, "/mnt/androidwritable")) ||
            StartsWith(test, "/storage/")) {
            toUnmount.push_front(test);
        }
    }
    endmntent(fp);

    for (const auto& path : toUnmount) {
        LOG(DEBUG) << "Tearing down stale mount " << path;
        android::vold::ForceUnmount(path);
    }

    return 0;
}
```



#### 卸载dm设备

dm设备是linux下管理映射设备的列表。vold启动时将重置与vold相关的所有dm设备：

```c++
int Devmapper::destroyAll() {
    ATRACE_NAME("Devmapper::destroyAll");

    auto& dm = DeviceMapper::Instance();
    std::vector<DeviceMapper::DmBlockDevice> devices;
    if (!dm.GetAvailableDevices(&devices)) {
        LOG(ERROR) << "Failed to get dm devices";
        return -1;
    }

    for (const auto& device : devices) {
        if (android::base::StartsWith(device.name(), kVoldPrefix)) {
            LOG(DEBUG) << "Tearing down stale dm device named " << device.name();
            if (!dm.DeleteDevice(device.name())) {
                if (errno != ENXIO) {
                    PLOG(WARNING) << "Failed to destroy dm device named " << device.name();
                }
            }
        } else {
            LOG(DEBUG) << "Found unmanaged dm device named " << device.name();
        }
    }
    return 0;
}
```

读取所有的的dm设备，如果设备的名称为kVoldPrefix（“vold:”）开头，则将请求dm（DeviceMapper）将对应设备删除。



#### 清除环回设备

环回设备是Linux提供的一种机制，可以创建一个块设备，该设备直接映射到文件系统的普通文件上，而不是映射到真正的物理块设备和光盘设备上。这样做的目的是可以通过mount命令把文件系统中的文件挂载到系统中，像块设备一样访问（https://man7.org/linux/man-pages/man4/loop.4.html）。主要用于类似img镜像文件的使用场景。

vold在启动的时候会清除环回设备：

```c++
int Loop::destroyAll() {
    ATRACE_NAME("Loop::destroyAll");

    std::string root = "/dev/block/";
    auto dirp = std::unique_ptr<DIR, int (*)(DIR*)>(opendir(root.c_str()), closedir);
    if (!dirp) {
        PLOG(ERROR) << "Failed to opendir";
        return -1;
    }

    // Poke through all devices looking for loops
    struct dirent* de;
    while ((de = readdir(dirp.get()))) {
        auto test = std::string(de->d_name);
        if (!android::base::StartsWith(test, "loop")) continue;

        auto path = root + de->d_name;
        unique_fd fd(open(path.c_str(), O_RDWR | O_CLOEXEC));
        if (fd.get() == -1) {
            if (errno != ENOENT) {
                PLOG(WARNING) << "Failed to open " << path;
            }
            continue;
        }

        struct loop_info64 li;
        if (ioctl(fd.get(), LOOP_GET_STATUS64, &li) < 0) {
            if (errno != ENXIO) {
                PLOG(WARNING) << "Failed to LOOP_GET_STATUS64 " << path;
            }
            continue;
        }

        auto id = std::string((char*)li.lo_crypt_name);
        if (android::base::StartsWith(id, kVoldPrefix)) {
            LOG(DEBUG) << "Tearing down stale loop device at " << path << " named " << id;

            if (ioctl(fd.get(), LOOP_CLR_FD, 0) < 0) {
                PLOG(WARNING) << "Failed to LOOP_CLR_FD " << path;
            }
        } else {
            LOG(DEBUG) << "Found unmanaged loop device at " << path << " named " << id;
        }
    }

    return 0;
}
```

直接遍历“/dev/block”目录下所有以loop开头的设备，通过LOOP_GET_STATUS64查询环回设备的信息，找到vold相关的环回设备（lo_crypt_name为“vold:”开头），将设备与文件系统断开。



### VolumeManager的配置处理

主要功能是读取系统中的fstab信息，系统中的fstab信息一般保存在系统中的etc目录下，配置处理主要将其中文件读取并解析出来。vold按照以下优先级查找。

首先查询fstab的后缀，依次查询：

- ro.boot.fstab_suffix
- ro.boot.hardware
- ro.boot.hardware.platform

prop项的值，将查询出来的后缀依次与以下路径进行拼接，检查其中的文件是否存在

- /odm/etc/fstab.
- /vendor/etc/fstab.
- /first_stage_ramdisk/fstab.

如果存在则将其内容作为fstab的配置信息解析处理。以pixel为例：其ro.boot.hardware为blueline，但是该后缀下没有任何文件匹配，而ro.boot.hardware.platform值为sdm845，而对应有/vendor/etc/fstab.sdm845的文件存在，因此，pixel将使用该配置文件作为fstab信息。其内容大致如下：

```ini
# Android fstab file.

#<src>                                              <mnt_point>        <type>      <mnt_flags and options>                               <fs_mgr_flags>
system                                              /system            ext4        ro,barrier=1                                          wait,slotselect,avb=vbmeta,logical,first_stage_mount
system_ext                                          /system_ext        ext4        ro,barrier=1                                          wait,slotselect,avb,logical,first_stage_mount
vendor                                              /vendor            ext4        ro,barrier=1                                          wait,slotselect,avb,logical,first_stage_mount
product                                             /product           ext4        ro,barrier=1                                          wait,slotselect,avb,logical,first_stage_mount
/dev/block/by-name/metadata                         /metadata          ext4        noatime,nosuid,nodev,discard,data=journal,commit=1    wait,formattable,first_stage_mount
/dev/block/bootdevice/by-name/userdata              /data              f2fs        noatime,nosuid,nodev,discard,reserve_root=32768,resgid=1065,fsync_mode=nobarrier       latemount,wait,check,fileencryption=ice,keydirectory=/metadata/vold/metadata_encryption,quota,formattable,sysfs_path=/sys/devices/platform/soc/1d84000.ufshc,reservedsize=128M,checkpoint=fs
/dev/block/bootdevice/by-name/misc                  /misc              emmc        defaults                                              defaults
/dev/block/bootdevice/by-name/modem                 /vendor/firmware_mnt          vfat        ro,shortname=lower,uid=1000,gid=1000,dmask=227,fmask=337,context=u:object_r:firmware_file:s0   wait,slotselect
/devices/platform/soc/a600000.ssusb/a600000.dwc3*   auto               vfat        defaults                                              voldmanaged=usb:auto
/dev/block/zram0                                    none               swap        defaults                                              zramsize=2147483648,max_comp_streams=8,zram_backingdev_size=512M


```

以上主要标识了文件中的源设备、挂载点、文件类型、挂载参数和标志为。与vold有关系的主要是fs_mgr_flags，如果在该flag中指定了voldmanaged的设备，在vold中可以动态加载和移除



### NetlinkManager启动

vold内部使用netlink类型的socket与内核进行交互，netlink是一种特殊的socket(https://linux.die.net/man/7/netlink)。整体的过程如下所示：

![](images/storage/vold-NetlinkManager%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.png)

首先创建了一个的socket地址：

```C++
	struct sockaddr_nl nladdr;	
	memset(&nladdr, 0, sizeof(nladdr));
    nladdr.nl_family = AF_NETLINK;
    nladdr.nl_pid = getpid();
    nladdr.nl_groups = 0xffffffff;
```

nl_family指定了协议族为AF_NETLINK，表明vold希望通过socket来与内核进行通信，nl_pid指定了socket的owner，通过getpid获取，所以就是vold的进程号。nl_groups在netlink的设计中，每个bit位用于表示netlink事件中的一个group，在bind socket时，nladdr中的nl_groups将用来指定需要监听的事件的掩码，最多可以监听32个事件，以上全部设置为f，意思是想要监听所有的事件。

```c++
    struct sockaddr_nl nladdr;
    int sz = 64 * 1024;
    int on = 1;

    ...
    if ((mSock = socket(PF_NETLINK, SOCK_DGRAM | SOCK_CLOEXEC, NETLINK_KOBJECT_UEVENT)) < 0) {
        PLOG(ERROR) << "Unable to create uevent socket";
        return -1;
    }

    // When running in a net/user namespace, SO_RCVBUFFORCE will fail because
    // it will check for the CAP_NET_ADMIN capability in the root namespace.
    // Try using SO_RCVBUF if that fails.
    if ((setsockopt(mSock, SOL_SOCKET, SO_RCVBUFFORCE, &sz, sizeof(sz)) < 0) &&
        (setsockopt(mSock, SOL_SOCKET, SO_RCVBUF, &sz, sizeof(sz)) < 0)) {
        PLOG(ERROR) << "Unable to set uevent socket SO_RCVBUF/SO_RCVBUFFORCE option";
        goto out;
    }

    if (setsockopt(mSock, SOL_SOCKET, SO_PASSCRED, &on, sizeof(on)) < 0) {
        PLOG(ERROR) << "Unable to set uevent socket SO_PASSCRED option";
        goto out;
    }
```

以上代码，首先创建一个socket，指定了的最后一个值为NETLINK_KOBJECT_UEVENT（https://linux.die.net/man/7/netlink）表示接收kernel发送到用户空间的事件消息。设置socket的参数接收buffer大小设置为64K，同时设置SO_PASSCRED（消息控制的设置），

紧接着启动将创建出来的socket绑定到netlink的地址上，最后把绑定好的socket传递给NetlinHandler启动监听：

```c++
    if (bind(mSock, (struct sockaddr*)&nladdr, sizeof(nladdr)) < 0) {
        PLOG(ERROR) << "Unable to bind uevent socket";
        goto out;
    }

    mHandler = new NetlinkHandler(mSock);
    if (mHandler->start()) {
        PLOG(ERROR) << "Unable to start NetlinkHandler";
        goto out;
    }
```

相关的类的结构如下：

![](images/storage/vold-NetlinkManager%E7%9A%84%E7%B1%BB%E7%BB%93%E6%9E%84.png)

socket的事件监听，通过Android里面已经包装好的SocketListener启动监听（startListener）后，即可通过onEvent回调接收到内核发送出来的事件消息。其中的事件流程，后面再介绍。



### Vold中的coldboot冷启动

vold进程启动的最后一步是冷启动设置，冷启动设置的核心代码如下所示：

```c++
coldboot("/sys/block");
```



```C++
static void coldboot(const char* path) {
    ATRACE_NAME("coldboot");
    DIR* d = opendir(path);
    if (d) {
        do_coldboot(d, 0);
        closedir(d);
    }
}

static void do_coldboot(DIR* d, int lvl) {
    struct dirent* de;
    int dfd, fd;

    dfd = dirfd(d);

    fd = openat(dfd, "uevent", O_WRONLY | O_CLOEXEC);
    if (fd >= 0) {
        write(fd, "add\n", 4);
        close(fd);
    }

    while ((de = readdir(d))) {
        DIR* d2;

        if (de->d_name[0] == '.') continue;

        if (de->d_type != DT_DIR && lvl > 0) continue;

        fd = openat(dfd, de->d_name, O_RDONLY | O_DIRECTORY | O_CLOEXEC);
        if (fd < 0) continue;

        d2 = fdopendir(fd);
        if (d2 == 0)
            close(fd);
        else {
            do_coldboot(d2, lvl + 1);
            closedir(d2);
        }
    }
}
```

从"/sys/block"目录开始往下搜索，如果找到uevent文件，则直接向该设备上写入“add\n”的字串。原理暂时不清楚，可能会触发设备上的事件。



## NetlinkManager接收内核事件的机制

TODO: