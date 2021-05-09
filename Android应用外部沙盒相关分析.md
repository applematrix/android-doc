# Android应用外部沙盒相关分析

基于Android R版本的代码进行分析，R版本开始，外置存储的管理也按照沙盒的方式进行管理



## 应用触发创建沙盒的入口

​	在App中通过Context的接口：

​	`public File getExternalFilesDir(String type) `

可以获取到外部沙盒的文件，从而对外部的文件进行操作。如果应用的沙盒不存在，该操作将会触发系统创建应用沙盒，相关流程如下：

![](https://github.com/applematrix/android-doc/blob/master/images/storage/%E5%BA%94%E7%94%A8%E8%AE%BF%E9%97%AE%E6%B2%99%E7%9B%92%E7%9A%84%E5%85%A5%E5%8F%A3.png)

该接口主要做了以下几件事情：

1. 调用Environment的buildExternalStorageAppFilesDirs，构造沙盒目录路径
2. 调用Environment的buildPaths构建目录路径
3. 调用ensureExternalDirsExistOrFilter创建目录

以下依次进行介绍。



## 构造应用的沙盒路径

应用的沙盒的路径通过Environment的buildExternalStorageAppFilesDirs来构造，构造路径的时序图如下所示：

![](https://github.com/applematrix/android-doc/blob/master/images/storage/%E5%88%86%E6%9E%90-%E6%9E%84%E9%80%A0%E5%A4%96%E9%83%A8%E6%B2%99%E7%9B%92%E8%B7%AF%E5%BE%84.png)

沙盒路径的构造通过Environment来创建，由于支持多用户，所以实际上内部又调用了UserEnvironment的接口进行路径的获取，首先通过getExternalDirs函数从StorageManager来查询对用户的所有可写的volume列表：

```java
	public File[] getExternalDirs() {
            final StorageVolume[] volumes = StorageManager.getVolumeList(mUserId,
                    StorageManager.FLAG_FOR_WRITE);
            final File[] files = new File[volumes.length];
            for (int i = 0; i < volumes.length; i++) {
                files[i] = volumes[i].getPathFile();
            }
            return files;
        }
```

之后就可以从StorageVolume的getPathFile接口中拿到对应的文件路径，再基于已有的目录下通过buildPaths把对应的路径构造出来：

```java
	public File[] buildExternalStorageAppFilesDirs(String packageName) {
            return buildPaths(getExternalDirs(), DIR_ANDROID, DIR_DATA, packageName, DIR_FILES);
        }
```

buildPaths即是将文件的路径依次拼接起来，其中DIR_ANDROID为“Android”、DIR_DATA为“data”、DIR_FILES为“files”，假设应用的包名为：com.example.myapp，那么以上的操作拼接后将创建的外部路径为假设为主用户：

/storage/emulated/0/Android/data/com.example.myapp/files



## 构造应用定义的目录路径

前面开始介绍的，应用调用了getExternalFilesDir接口时，将再次调用buildPaths拼接路径：

```java
    public File[] getExternalFilesDirs(String type) {
        synchronized (mSync) {
            File[] dirs = Environment.buildExternalStorageAppFilesDirs(getPackageName());
            if (type != null) {
                dirs = Environment.buildPaths(dirs, type);
            }
            return ensureExternalDirsExistOrFilter(dirs, true /* tryCreateInProcess */);
        }
    }
```

即将前面创建出来的目录拼接到一起形成最终的目录路径，假设应用创建了一个名为my_data的目录，则按照前面的应用包名举例，最终的路径即为：/storage/emulated/0/Android/data/com.example.myapp/files/my_data

注意，经过buildPaths后，此时的目录还没有创建出来，只是生成了路径位置。



## 创建应用沙盒内的数据目录

上面分析的下一步是通过ensureExternalDirsExistOrFilter创建实际的目录：

```java
    private File[] ensureExternalDirsExistOrFilter(File[] dirs, boolean tryCreateInProcess) {
        final StorageManager sm = getSystemService(StorageManager.class);
        final File[] result = new File[dirs.length];
        for (int i = 0; i < dirs.length; i++) {
            File dir = dirs[i];
            if (!dir.exists()) {
                try {
                    if (!tryCreateInProcess || !dir.mkdirs()) {
                        // recheck existence in case of cross-process race
                        if (!dir.exists()) {
                            // Failing to mkdir() may be okay, since we might not have
                            // enough permissions; ask vold to create on our behalf.
                            sm.mkdirs(dir);
                        }
                    }
                } catch (Exception e) {
                    Log.w(TAG, "Failed to ensure " + dir + ": " + e);
                    dir = null;
                }
            }
            result[i] = dir;
        }
        return result;
    }
```

从以上的代码中可以看到，创建目录分为两阶段进行尝试：

1. 调用jdk的File的mkdirs接口进行创建
2. 如果应用调用File的mkdir创建失败（或者tryCreateInProcess为false，前面的参数传递为true，这种情况不考虑），一般创建失败原因是没权限，此时将通过StorageManager的mkdirs接口创建，该接口将请求vold进行创建

我们重点分析第二种情况



## StorageManager中的mkdirs

StorageManagerService中的mkdirs时的核心逻辑，主要是做参数的检查和校验等操作，其主要逻辑如下：

```java
        final Matcher matcher = KNOWN_APP_DIR_PATHS.matcher(appPath);
        if (matcher.matches()) {
            // And that the package dir matches the calling package
            if (!matcher.group(3).equals(callingPkg)) {
                throw new SecurityException("Invalid mkdirs path: " + appFile
                        + " does not contain calling package " + callingPkg);
            }
            // And that the user id part of the path (if any) matches the calling user id,
            // or if for a public volume (no user id), the user matches the current user
            if ((matcher.group(2) != null && !matcher.group(2).equals(Integer.toString(userId)))
                    || (matcher.group(2) == null && userId != mCurrentUserId)) {
                throw new SecurityException("Invalid mkdirs path: " + appFile
                        + " does not match calling user id " + userId);
            }
            try {
                mVold.setupAppDir(appPath, callingUid);
            } catch (RemoteException e) {
                throw new IllegalStateException("Failed to prepare " + appPath + ": " + e);
            }

            return;
        }
```

上面的代码中，首先要求文件的路径必须满足正则表达式的格式要求。正则表达式：

```java
    /** Matches known application dir paths. The first group contains the generic part of the path,
     * the second group contains the user id (or null if it's a public volume without users), the
     * third group contains the package name, and the fourth group the remainder of the path.
     */
    public static final Pattern KNOWN_APP_DIR_PATHS = Pattern.compile(
            "(?i)(^/storage/[^/]+/(?:([0-9]+)/)?Android/(?:data|media|obb|sandbox)/)([^/]+)(/.*)?");
```

即要求目录的路径，只能是/storage开头的外部存储的/Android/下的data、media、obb或sandbox目录。

然后调用vold的setupAppDir创建目录，所以应用的沙盒目录核心是在vold中创建的。传递给vold的参数为对应的目录，以及callingUid，即API调用的应用对应的uid。



## vold中的setupAppDir流程

前面的分析中StorageManagerService调用vold的setupAppDir的过程如下：

![](https://github.com/applematrix/android-doc/blob/master/images/storage/%E5%88%86%E6%9E%90-StorageManagerService%E8%B0%83%E7%94%A8setupAppDir.png)



setupAppDir的过程在vold进程中的volumeManager中完成。在volumeManager中setupAppDir时流程如下：

![](https://github.com/applematrix/android-doc/blob/master/images/storage/%E5%88%86%E6%9E%90-volumemanager%20setupAppDir%E7%9A%84%E5%85%A5%E5%8F%A3.png)

先找到对应的路径，具体应该是创建到那个volume中，在将外部的路径转化为内部路径，之后调用PrepareAppDirFromRoot接口创建目录。

每个volume对外有一个外部名字和内部名字，在vold中需要将路径的外部名字转化为内部路径，比如一个路径在App中看到的是外部路径，其路径地址形式为：/storage/emulated/0/Android/data/com.example.myapp/files/my_data。

​	假如在vold中，/storage/emulated对应的内部路径为/data/media路径，则替换后的内部路径为/data/media/0/Android/data/com.example.myapp/files/my_data

​	以上的流程中，最关键的是vold中需要对目录的权限进行设置，而如果对应的volume为kPublic类型的，则通过fs_mkdirs创建目录，按照vold中的注释解释，kPublic的volume卷通过fUSE来管理，所以不需要通过vold来设置，直接创建。

```C++
    if (volume->getType() == VolumeBase::Type::kPublic) {
        // On public volumes, we don't need to setup permissions, as everything goes through
        // FUSE; just create the dirs and be done with it.
        return fs_mkdirs(lowerPath.c_str(), 0700);
    }
	// Create the app paths we need from the root
    return PrepareAppDirFromRoot(lowerPath, volumeRoot, appUid, fixupExistingOnly);
```

对于非kPublic类型的volume，则需要通过PrepareAppDirFromRoot来创建对应的目录文件。



### PrepareAppDirFromRoot创建沙盒流程

​	PrepareAppDirFromRoot的主要逻辑如下（system/vold/Utils.cpp）：

 ![](https://github.com/applematrix/android-doc/blob/master/images/storage/%E5%88%86%E6%9E%90-PrepareAppDirFromRoot%E7%9A%84%E6%B5%81%E7%A8%8B%E6%A6%82%E8%A6%81.png)

​	

​	以下依次对以上流程进行说明。



#### 	PrepareAndroidDirs创建Android层级目录

​		对应的代码如下所示：

```C++
    status_t PrepareAndroidDirs(const std::string& volumeRoot) {
    std::string androidDir = volumeRoot + kAndroidDir;
    std::string androidDataDir = volumeRoot + kAppDataDir;
    std::string androidObbDir = volumeRoot + kAppObbDir;
    std::string androidMediaDir = volumeRoot + kAppMediaDir;

    bool useSdcardFs = IsSdcardfsUsed();
    
    // mode 0771 + sticky bit for inheriting GIDs
    mode_t mode = S_IRWXU | S_IRWXG | S_IXOTH | S_ISGID;
    if (fs_prepare_dir(androidDir.c_str(), mode, AID_MEDIA_RW, AID_MEDIA_RW) != 0) {
        PLOG(ERROR) << "Failed to create " << androidDir;
        return -errno;
    }
    
    gid_t dataGid = useSdcardFs ? AID_MEDIA_RW : AID_EXT_DATA_RW;
    if (fs_prepare_dir(androidDataDir.c_str(), mode, AID_MEDIA_RW, dataGid) != 0) {
        PLOG(ERROR) << "Failed to create " << androidDataDir;
        return -errno;
    }
    
    gid_t obbGid = useSdcardFs ? AID_MEDIA_RW : AID_EXT_OBB_RW;
    if (fs_prepare_dir(androidObbDir.c_str(), mode, AID_MEDIA_RW, obbGid) != 0) {
        PLOG(ERROR) << "Failed to create " << androidObbDir;
        return -errno;
    }
    // Some other apps, like installers, have write access to the OBB directory
    // to pre-download them. To make sure newly created folders in this directory
    // have the right permissions, set a default ACL.
    SetDefaultAcl(androidObbDir, mode, AID_MEDIA_RW, obbGid, {});
    
    if (fs_prepare_dir(androidMediaDir.c_str(), mode, AID_MEDIA_RW, AID_MEDIA_RW) != 0) {
        PLOG(ERROR) << "Failed to create " << androidMediaDir;
        return -errno;
    }
    
    return OK;
}
```

逻辑比较简单：

1. 在外卡上创建Android目录，mode为02771（2位粘滞位，即S_ISGID），即rwxrws---，owner和group都为media_rw
2. 在外卡上创建Android/data目录，mode为02771，即rwxrws---，owner为media_rw，群组为ext_data_rw（如果是sdcardfs，则群组为media_rw，在R版本上sdcardfs会废弃，需要考虑Q版本升级到R版本的手机）
3. 在外卡上创建Android/obb目录，mode为02771，即rwxrws---，owner为media_rw，群组为ext_obb_rw
4. 针对obb目录做SetDefaultAcl的处理，该流程后面还会使用，后面再做介绍
5. 在外卡上创建Android/obb目录，mode为02771，即rwxrws---，owner为media_rw，群组为media_rw

经过这一处理后，应用的沙盒外部的目录已经创建好，注意，其中设置了粘滞位，即S_ISGID，会让在其目录下创建的子目录继承父目录的属性，这很关键。



在创建下一层目录（应用的外部沙盒）之前，系统需要进行几个关键的参数的计算，参数将用于设置对应目录的属性。主要包括以下：

1. uid，用于设置目录的owner，需要考虑多用户的场景
2. gid，用于设置目录的group
3. mode，用于设置目录的mode
4. projectId，用于设置目录的配额，该值将通过文件系统的ioctl操作设置到文件的属性中
5. additional groups，额外的group，该信息是文件的额外属性，通过ioctl设置到文件的xattr属性中

以下依次介绍以上各参数的设置策略



#### 目录的UID策略

​	目录的uid即是StorageManagerService中接收到的binder调用的callingUid，即是应用对应的uid

#### 目录的GID策略

​	目录的gid的设置策略如下所示：

```c++
    uid_t uid = appUid;
    gid_t gid = AID_MEDIA_RW;
    std::vector<gid_t> additionalGids;
    std::string appDir;

    // Check that the next part matches one of the allowed Android/ dirs
    if (StartsWith(pathFromRoot, kAppDataDir)) {
        appDir = kAppDataDir;
        if (!sdcardfsSupport) {
            gid = AID_EXT_DATA_RW;
            // Also add the app's own UID as a group; since apps belong to a group
            // that matches their UID, this ensures that they will always have access to
            // the files created in these dirs, even if they are created by other processes
            additionalGids.push_back(uid);
        }
    } else if (StartsWith(pathFromRoot, kAppMediaDir)) {
        appDir = kAppMediaDir;
        if (!sdcardfsSupport) {
            gid = AID_MEDIA_RW;
        }
    } else if (StartsWith(pathFromRoot, kAppObbDir)) {
        appDir = kAppObbDir;
        if (!sdcardfsSupport) {
            gid = AID_EXT_OBB_RW;
            // See comments for kAppDataDir above
            additionalGids.push_back(uid);
        }
    } else {
        LOG(ERROR) << "Invalid application directory: " << path;
        return -EINVAL;
    }
```





![](https://github.com/applematrix/android-doc/blob/master/images/storage/%E5%88%86%E6%9E%90-gid%E8%AE%BE%E7%BD%AE%E7%AD%96%E7%95%A5.png)

​	以上策略对应的是sdcardfs不开启的情况（即R版本后的默认情况），如果sdcard开启，则：

1. 无论在哪个目录下创建目录，其gid都为media_rw

2. 不会将应用的id加入到additional_groups列表中

   

#### 目录的mode策略

应用沙盒的目录，默认的mode，无论在哪个目录下，均为02770（注意设置了粘滞位），即rwxrws---

```c++
    // mode = 770, plus sticky bit on directory to inherit GID when apps
    // create subdirs
    mode_t mode = S_IRWXU | S_IRWXG | S_ISGID;
```



#### 目录的projectId策略

projectId是底层文件系统用于管理配额的参数信息（详细如何管理的没仔细分析），在vold中创建目录的时候，需要计算出projectId的值，后面通过ioctl命令将其下发到底层文件系统中去，projectId是一个long型的数值。取值的方式通过以下

```c++
    // Derive initial project ID
    if (appDir == kAppDataDir || appDir == kAppMediaDir) {
        projectId = uid - AID_APP_START + PROJECT_ID_EXT_DATA_START;
    } else if (appDir == kAppObbDir) {
        projectId = uid - AID_APP_START + PROJECT_ID_EXT_OBB_START;
    }
```

可以看到，实际上系统的id分配系统中保留了号段，专门用于projectId的分配，以上使用到两个号段：

1. 一个从PROJECT_ID_EXT_DATA_START（20000）开始，用于分配给Android/data和Android/media下的目录使用
2. 一个从PROJECT_ID_EXT_OBB_START（40000）开始，用于分配给Android/obb下的目录使用

每个号段10000个ID，对应的号段预留代码可以在system/core/libcutils/include/private/android_projectid_config.h中查询到。projectId的计算方式，将app的id计算出来后，加上号段起始地址即可。假设一个应用的appId为10037，由于AID_APP_START为10000，所以Android/data和Android/media下的目录的projectId即为20037（10037-10000+20000）。

此外，对于应用沙盒目录下的cache目录具有特殊处理，该cache目录的projectId从PROJECT_ID_EXT_CACHE_START（30000）开始分配：

```c++
	if (appDir == kAppDataDir && depth == 1 && component == "cache/") {
            // All dirs use the "app" project ID, except for the cache dirs in
            // Android/data, eg Android/data/com.foo/cache
            // Note that this "sticks" - eg subdirs of this dir need the same
            // project ID.
            projectId = uid - AID_APP_START + PROJECT_ID_EXT_CACHE_START;
        }
```



#### PrepareDirWithProjectId创建目录

PrepareDirWithProjectId的流程很简单，主要就是2步：

1. 使用前面获取到的uid、gid、mode创建指定的目录
2. 对目录设置projectId属性

对应的代码如下：

```C++
int PrepareDirWithProjectId(const std::string& path, mode_t mode, uid_t uid, gid_t gid,
                            long projectId) {
    int ret = fs_prepare_dir(path.c_str(), mode, uid, gid);

    if (ret != 0) {
        return ret;
    }

    if (!IsSdcardfsUsed()) {
        ret = SetQuotaProjectId(path, projectId);
    }

    return ret;
}
```

fs_prepare_dir是AOSP中包装的工具函数，公共的创建文件流程，其中主要的流程即是调用chown、chmod等系统api，相关代码在system/core/libcutils/fs.cpp中，在此不做介绍。

SetQuotaProjectId即涉及到将前面生成的projectId进行设置。

```c++
int SetQuotaProjectId(const std::string& path, long projectId) {
    struct fsxattr fsx;

    android::base::unique_fd fd(TEMP_FAILURE_RETRY(open(path.c_str(), O_RDONLY | O_CLOEXEC)));
    if (fd == -1) {
        PLOG(ERROR) << "Failed to open " << path << " to set project id.";
        return -1;
    }

    int ret = ioctl(fd, FS_IOC_FSGETXATTR, &fsx);
    if (ret == -1) {
        PLOG(ERROR) << "Failed to get extended attributes for " << path << " to get project id.";
        return ret;
    }

    fsx.fsx_projid = projectId;
    ret = ioctl(fd, FS_IOC_FSSETXATTR, &fsx);
    if (ret == -1) {
        PLOG(ERROR) << "Failed to set project id on " << path;
        return ret;
    }
    return 0;
}
```

简而言之：

1. 打开文件
2. 通过ioctl的FS_IOC_FSGETXATTR获取文件系统的扩展属性
3. 将projectId设置到扩展属性中
4. 通过ioctl的FS_IOC_FSSETXATTR将projectId设置到文件系统中对应的文件上



#### 顶层沙盒目录的特殊处理

前面提到，对于顶层的沙盒目录，系统中会进行特殊处理，什么叫顶层沙盒目录，即应用包名这一级，假设一个目录的地址为/storage/emulated/0/Android/data/com.example.myapp/files/my_data，那么从Android/data目录开始的下一级目录com.example.myapp即为顶层沙盒目录（Android/media目录下同理）。该目录需要额外进行以下两个步骤：

```c++
        if (depth == 0) {
            // Set the default ACL on the top-level application-specific directories,
            // to ensure that even if applications run with a umask of 0077,
            // new directories within these directories will allow the GID
            // specified here to write; this is necessary for apps like
            // installers and MTP, that require access here.
            //
            // See man (5) acl for more details.
            ret = SetDefaultAcl(pathToCreate, mode, uid, gid, additionalGids);
            if (ret != 0) {
                return ret;
            }

            if (!sdcardfsSupport) {
                // Set project ID inheritance, so that future subdirectories inherit the
                // same project ID
                ret = SetQuotaInherit(pathToCreate);
                if (ret != 0) {
                    return ret;
                }
            }
        }
```



![](https://github.com/applematrix/android-doc/blob/master/images/storage/%E5%88%86%E6%9E%90-%E9%A1%B6%E5%B1%82%E6%B2%99%E7%9B%92%E7%9A%84%E7%9B%AE%E5%BD%95.png)

##### SetDefaultAcl

​	根据函数名字可以看出，该函数主要是设置权限相关的属性，其代码如下：

```C++
status_t SetDefaultAcl(const std::string& path, mode_t mode, uid_t uid, gid_t gid,
                       std::vector<gid_t> additionalGids) {
    if (IsSdcardfsUsed()) {
        // sdcardfs magically takes care of this
        return OK;
    }

    size_t num_entries = 3 + (additionalGids.size() > 0 ? additionalGids.size() + 1 : 0);
    size_t size = sizeof(posix_acl_xattr_header) + num_entries * sizeof(posix_acl_xattr_entry);
    auto buf = std::make_unique<uint8_t[]>(size);

    posix_acl_xattr_header* acl_header = reinterpret_cast<posix_acl_xattr_header*>(buf.get());
    acl_header->a_version = POSIX_ACL_XATTR_VERSION;

    posix_acl_xattr_entry* entry =
            reinterpret_cast<posix_acl_xattr_entry*>(buf.get() + sizeof(posix_acl_xattr_header));

    int tag_index = 0;

    entry[tag_index].e_tag = ACL_USER_OBJ;
    // The existing mode_t mask has the ACL in the lower 9 bits:
    // the lowest 3 for "other", the next 3 the group, the next 3 for the owner
    // Use the mode_t masks to get these bits out, and shift them to get the
    // correct value per entity.
    //
    // Eg if mode_t = 0700, rwx for the owner, then & S_IRWXU >> 6 results in 7
    entry[tag_index].e_perm = (mode & S_IRWXU) >> 6;
    entry[tag_index].e_id = uid;
    tag_index++;

    entry[tag_index].e_tag = ACL_GROUP_OBJ;
    entry[tag_index].e_perm = (mode & S_IRWXG) >> 3;
    entry[tag_index].e_id = gid;
    tag_index++;

    if (additionalGids.size() > 0) {
        for (gid_t additional_gid : additionalGids) {
            entry[tag_index].e_tag = ACL_GROUP;
            entry[tag_index].e_perm = (mode & S_IRWXG) >> 3;
            entry[tag_index].e_id = additional_gid;
            tag_index++;
        }

        entry[tag_index].e_tag = ACL_MASK;
        entry[tag_index].e_perm = (mode & S_IRWXG) >> 3;
        entry[tag_index].e_id = 0;
        tag_index++;
    }

    entry[tag_index].e_tag = ACL_OTHER;
    entry[tag_index].e_perm = mode & S_IRWXO;
    entry[tag_index].e_id = 0;

    int ret = setxattr(path.c_str(), XATTR_NAME_POSIX_ACL_DEFAULT, acl_header, size, 0);

    if (ret != 0) {
        PLOG(ERROR) << "Failed to set default ACL on " << path;
    }

    return ret;
}
```

![](https://github.com/applematrix/android-doc/blob/master/images/storage/%E5%88%86%E6%9E%90-%E9%A1%B6%E5%B1%82%E6%B2%99%E7%9B%92%E7%9B%AE%E5%BD%95%E7%9A%84ACL%E5%B1%9E%E6%80%A7%E8%AE%BE%E7%BD%AE.png)

简单的说明就是，系统根据uid、gid、mod信息、additional group的信息，将其分别填写到对应的posix_acl_attr_entry的字段中，在通过setxattr函数将整个acl信息设置到文件目录中去。其中：

1. owner的entry，主要设置uid信息和mode中owner的S_IRWXU属性，因为前面已经分析过了，mode为02770，所以设置为rwx
2. group的entry，主要设置gid信息（ext_data_rw或media_rw）和mode中group的S_IRWXU属性，设置为rwx
3. 额外的group的entry，上面的逻辑支持多个额外group的entry，在目录的GID策略中分析过，data数据沙盒默认会将应用的uid加入到额外的group中，其他未添加任何group，所以额外的group只有一个，即对应应用的uid。额外的group的entry中，id设置为uid，而mode与group的mode一致，都设置为rwx
4. other的entry，id设置为0，mode也为0

setxattr函数同样也是AOSP包装的公共的文件操作函数（bionic/libc/include/sys/xattr.h）最终设置到文件系统中，不做详细介绍。



##### setQuotaInherit

setQuotaInherit从字面意思上看是需要将对应的文件的配额信息继承，其代码也比较简单

```c++
int SetQuotaInherit(const std::string& path) {
    unsigned int flags;

    android::base::unique_fd fd(TEMP_FAILURE_RETRY(open(path.c_str(), O_RDONLY | O_CLOEXEC)));
    if (fd == -1) {
        PLOG(ERROR) << "Failed to open " << path << " to set project id inheritance.";
        return -1;
    }

    int ret = ioctl(fd, FS_IOC_GETFLAGS, &flags);
    if (ret == -1) {
        PLOG(ERROR) << "Failed to get flags for " << path << " to set project id inheritance.";
        return ret;
    }

    flags |= FS_PROJINHERIT_FL;

    ret = ioctl(fd, FS_IOC_SETFLAGS, &flags);
    if (ret == -1) {
        PLOG(ERROR) << "Failed to set flags for " << path << " to set project id inheritance.";
        return ret;
    }

    return 0;
}

```

流程总结：

1. 打开文件
2. ioctl读取flag
3. 更新flag
4. ioctl将更新的flag发送回文件系统



## 总结

​	应用外部沙盒，在vold中创建时，不是简单的创建了一个目录，设置给对应的应用own和mode即可，其中还设置了额外的acl属性已经配额管理等信息，所以外部沙盒的数据，如果不是经过以上流程创建的，没有经过对应的属性设置，可能会存在潜在的权限问题，以及可能底层文件系统配额限制（如projectId的具体用途）等方面，导致奇怪的问题。所以非应用进程，不要轻易的代替应用来创建对应的沙盒，否则容易出现三方兼容性问题。

​	此外，如果非要代替应用创建外部沙盒，建议通过vold的setupAppDir来创建，而不是自行通过mkdir，chown，chmod等操作来山寨一个看起来一样，但是存在潜在风险的沙盒目录。

​	如果应用的沙盒存在问题，vold中提供了一个fixupAppDir的函数可以进行修复，该方法未详细分析，但是大致流程与上面的分析类似，应该可以解决问题。
