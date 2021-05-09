# Android应用外部沙盒相关分析

基于Android R版本的代码进行分析，R版本开始，外置存储的管理也按照沙盒的方式进行管理



## 应用触发创建沙盒的入口

​	在App中通过Context的接口：

​	`public File getExternalFilesDir(String type) `

可以获取到外部沙盒的文件，从而对外部的文件进行操作。如果应用的沙盒不存在，该操作将会触发系统创建应用沙盒，相关流程如下：

![](images\storage\应用访问沙盒的入口.png)

该接口主要做了以下几件事情：

1. 调用Environment的buildExternalStorageAppFilesDirs，构造沙盒目录路径
2. 调用Environment的buildPaths构建目录路径
3. 调用ensureExternalDirsExistOrFilter创建目录

以下依次进行创建。



## 构造应用的沙盒路径

应用的沙盒的路径通过Environment的buildExternalStorageAppFilesDirs来构造，构造路径的时序图如下所示：

![](images\storage\分析-构造外部沙盒路径.png)

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

前面开始介绍的，应用







