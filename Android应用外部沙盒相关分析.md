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

以下依次进行创建。



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

![](C:\Users\huangdezhi\Documents\GitHub\android-doc\images\storage\分析-StorageManagerService调用setupAppDir.png)



setupAppDir的过程在vold进程中的volumeManager中完成。在volumeManager中setupAppDir时流程如下：

![](C:\Users\huangdezhi\Documents\GitHub\android-doc\images\storage\分析-volumemanager setupAppDir的入口.png)

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











