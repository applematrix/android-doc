# Android中的PMS服务

PMS是Android中的核心服务，负责管理系统中的所有应用，包括应用的组件，系统中如果启动应用，需要通过PMS来查询其所管理的所有应用的信息。PMS的主要功能包括以下：
- 应用安装
- 应用卸载
- 应用的数据管理
- 应用的组件管理

## PMS在系统中的层级
## PMS的启动
## PMS中预装应用的扫描过程
## PMS中安装应用的过程
## PMS中应用的数据沙盒隔离
## 应用APK组成方式
## APEX应用
## 应用的UID
系统中安装的应用，每个应用都具有一个唯一的UID，uid由PMS分配，应用的uid用户数据沙盒目录的隔离等，进程的权限等场景。是系统中非常重要的信息。在PMS中的每个应用的uid为整数，PMS将uid分为几个号段，应用从对应的号段中分配。在Process.java中：
```java
    /**
     * Defines the start of a range of UIDs (and GIDs), going from this
     * number to {@link #LAST_APPLICATION_UID} that are reserved for assigning
     * to applications.
     */
    public static final int FIRST_APPLICATION_UID = 10000;

    /**
     * Last of application-specific UIDs starting at
     * {@link #FIRST_APPLICATION_UID}.
     */
    public static final int LAST_APPLICATION_UID = 19999;
```
为每个应用分配id从10000到19999，总共10000个id，所以理论上系统中最多能安装的应用上限为10000个。除了以上uid，在系统中还会保留一些特定uid给特殊的应用，这些uid都是小于10000的id，都在Process.java中进行定义：
|UID|值|用途|
|-|-|-|
|ROOT_UID|0|root进程的uid|
|SYSTEM_UID|1000|system进程，一般system_server进程的uid，和sharedUid为andorid.system.uid的进程|
|PHONE_UID|1001|phone进程，通话相关的进程|
|SHELL_UID|2000|shell进程|
|LOG_UID|1007|在log group群组中的进程|
|WIFI_UID|1010|wifi相关的进程|
|MEDIA_UID|1013|mediaserver相关的进程|
|DRM_UID|1019|DRM相关的进程|
|SDCARD_RW_GID|1015|用于控制是否可以写存储区的groupid|
|VPN_UID|1016|vpn的uid|
|KEYSTORE_UID|1017|keystore的uid|
|CREDSTORE_UID|1076|credstore的uid|
|...|...|...|

以上定义的uid，部分可能直接用与native进程的uid，部分可用于给应用apk进行分配，还有部分用于group id定义。本节主要关注应用的uid，其他细节暂不描述。

### UID的分配算法
默认情况下，当系统安装应用时，应用的uid从10000开始往后依次分配，所以先安装的应用具有较小的uid，后安装的应用有较大的uid。但是由于应用可能会发生卸载的情况，所以可能部分uid会被空闲出来。
Uid的分配示意图如下所示：
假设一开始系统安装了3个应用，那么他们的uid依次从10000开始往后分配：

![](images/pms/uid1.png)

如果这时再安装一个新应用App4，那么将依次往后找未被占用的uid来进行分配：
![](images/pms/uid2.png)

如果这时将应用2卸载，那么uid的分配占用情况将变成：
![](images/pms/uid3.png)

原来app2占用的uid将会被释放，在系统内部注册表中，对应的位置会填充一个null指针，如果要搜索uid时，对应的的位置为null的uid可以被分配，但是此时有个特殊的处理。如果再安装应用App5时，并不是直接使用App2的uid。而是在后面尾部分配：
![](images/pms/uid4.png)

为什么不能直接使用App2的uid，原因可能是系统中可能还有内存的数据可能通过App2的uid来管控，如果直接复用App2的uid给其他的应用，可能会导致新安装的应用具有App2的权限出现安全问题。卸载应用的uid只能在系统重启后可以继续使用。系统中通过设置一个可用的uid的分配起点来避免刚卸载的应用的uid被复用。如果任何一个应用被卸载，那么可用的的uid分配起点只能从该uid的后面开始。如果没有空闲的，那么在最尾部追加。假设重启后再安装App6，那么uid的分配状态可能如下：

![](images/pms/uid5.png)

如果用户批量卸载了大量的应用，那么可用的的uid分配起点的状态将如下所示：

![](images/pms/uid6.png)

可用uid的分配起点总是指向被卸载的应用中的最大的uid的后一个uid。


## 应用的