---
title: Android ABI issue analysis
date: 2016-10-14 15:45:01
tags: [Android,ABI]
---

# 什么是ABI

ABI 全称 application binary interface，是一个机器语言级别的接口，描述的是二进制代码之间的兼容关系，这也意味着一起工作的二进制组件是ABI兼容的。一个SO库想要调用另一个SO库的函数，就要求它们的ABI兼容。
Stack overflow上有一个以API为类比来说明什么是ABI的回答[What is Application Binary Interface](http://stackoverflow.com/questions/2171177/what-is-application-binary-interface-abi)。


# Process的位数
下图是app启动的大致流程，图片来自http://gityuan.com/2016/03/26/app-process-create/。
![](/imgs/Android-ABI-issue-analysis/start_app_process.jpg)

**1. 创建出来的进程究竟是32位还是64位呢？**

```java
  /**
     * Start a new process.
     * 
     * <p>If processes are enabled, a new process is created and the
     * static main() function of a <var>processClass</var> is executed there.
     * The process will continue running after this function returns.
     * 
     * <p>If processes are not enabled, a new thread in the caller's
     * process is created and main() of <var>processClass</var> called there.
     * 
     * <p>The niceName parameter, if not an empty string, is a custom name to
     * give to the process instead of using processClass.  This allows you to
     * make easily identifyable processes even if you are using the same base
     * <var>processClass</var> to start them.
     * 
     * @param processClass The class to use as the process's main entry
     *                     point.
     * @param niceName A more readable name to use for the process.
     * @param uid The user-id under which the process will run.
     * @param gid The group-id under which the process will run.
     * @param gids Additional group-ids associated with the process.
     * @param debugFlags Additional flags.
     * @param targetSdkVersion The target SDK version for the app.
     * @param seInfo null-ok SELinux information for the new process.
     * @param abi non-null the ABI this app should be started with.
     * @param instructionSet null-ok the instruction set to use.
     * @param appDataDir null-ok the data directory of the app.
     * @param zygoteArgs Additional arguments to supply to the zygote process.
     * 
     * @return An object that describes the result of the attempt to start the process.
     * @throws RuntimeException on fatal start failure
     * 
     * {@hide}
     */
    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) 
```

函数**start**必须要有一个参数**String abi**。这个参数决定了启动的进程是64位还是32位。


**2、这个参数从哪里来呢？**


函数**start**的唯一调用者就是**startProcessLocked**, 下面看看这个函数的删减版。
```java
private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
            ...

            String requiredAbi = (abiOverride != null) ? abiOverride : app.info.primaryCpuAbi;
            if (requiredAbi == null) {
                requiredAbi = Build.SUPPORTED_ABIS[0];
            }

            String instructionSet = null;
            if (app.info.primaryCpuAbi != null) {
                instructionSet = VMRuntime.getInstructionSet(app.info.primaryCpuAbi);
            }

            app.gids = gids;
            app.requiredAbi = requiredAbi;
            app.instructionSet = instructionSet;

            // Start the process.  It will either succeed and return a result containing
            // the PID of the new process, or else throw a RuntimeException.
            boolean isActivityProcess = (entryPoint == null);
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);
            checkTime(startTime, "startProcess: asking zygote to start proc");
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
           ...
    }
```
正常运行的时候，代码中的**abiOverride**几乎总是为空，也就是说**start**函数的**abi**参数由**app.info.primaryCpuAbi**决定。如果**app.info.primaryCpuAbi**为空，就用**Build.SUPPORTED_ABIS[0]**,也就是**ro.product.cpu.abilist**中第一个值。


**3、app.info.primaryCpuAbi是哪里来的呢？**
系统运行起来之后，会将packages的信息保存在**/data/system/packages.xml**文件中，所以查看该文件可用知道对应App的ABI，在遇到问题时，可用查看该文件，但是也不能回答app.info.primaryCpuAbi是哪里来的。

这个值是在解析apk的时候确定下来的，如果确定不了，app.info.primaryCpuAbi就为空。

不论是安装应用，还是开机，**scanPackageLI**总是要被执行的一个函数。从Android L开始，引入了新函数**scanPackageDirtyLI** 。本文关注的ABI就是在scanPackageDirtyLI中被判断出来的。

在**PackageManagerService**构造函数中会对系统里所有的App执行**scanPackageDirtyLI**。


```java
private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg, int parseFlags,
            int scanFlags, long currentTime, UserHandle user)
```
在进入**scanPackageDirtyLI**的时候，**pkg.applicationInfo.primaryCpuAbi**值为空。因为pkg才实例化不久，primaryCpuAbi值还没有初始化。

函数**scanPackageDirtyLI**很长，经过各种判断，一些**pkg.applicationInfo.primaryCpuAbi**被确定下来。比如有so的App就根据so确定下来了。

对应高通项目，要注意**derivePackageAbi**函数。这个函数会调到一个native方法**findSupportedAbi**。这个方法在文件com_android_internal_content_NativeLibraryHelper.cpp中，被高通修改过。


```cpp
     if(status <= 0) {
        // Scan the 'assets' folder only if
        // the abi (after scanning the lib folder)
        // is not already set to 32-bit (i.e '1' or '2').

        int asset_status = NO_NATIVE_LIBRARIES;
        int rc = initAssetsVerifierLib();

        if (rc == LIB_INITED_AND_SUCCESS) {
            asset_status = GetAssetsStatusFunc(zipFile, supportedAbis, numAbis);
        } else {
            ALOGE("Failed to load assets verifier: %d", rc);
        }
        if(asset_status >= 0) {
            // Override the ABI only if
            // 'asset_status' is a valid ABI (64-bit or 32-bit).
            // This is to prevent cases where 'lib' folder
            // has native libraries, but
            // 'assets' folder has none.
            status = asset_status;
        }
```

意思就是如果apk的assets文件夹下有so库，系统会根据so库的ABI去决定app的ABI。


还有一个函数需要注意**adjustCpuAbisForSharedUserLPw**

```java
 /**
     * Adjusts ABIs for a set of packages belonging to a shared user so that they all match.
     * i.e, so that all packages can be run inside a single process if required.
     *
     * Optionally, callers can pass in a parsed package via {@code newPackage} in which case
     * this function will either try and make the ABI for all packages in {@code packagesForUser}
     * match {@code scannedPackage} or will update the ABI of {@code scannedPackage} to match
     * the ABI selected for {@code packagesForUser}. This variant is used when installing or
     * updating a package that belongs to a shared user.
     *
     * NOTE: We currently only match for the primary CPU abi string. Matching the secondary
     * adds unnecessary complexity.
     */
    private void adjustCpuAbisForSharedUserLPw(Set<PackageSetting> packagesForUser,
            PackageParser.Package scannedPackage, boolean forceDexOpt, boolean deferDexOpt,
            boolean bootComplete) 
```
这个函数其实是把有相同**android:sharedUserId**的packages分成一个组，把组里找到的第一个已经确定ABI的package的ABI作为组里其它还没确定ABI的packages的ABI。 

在**PackageManagerService**构造函数执行完毕之后，系统中绝大部分packages的ABI都确定了，还有极少数没有确定的，相关信息都写入文件**/data/system/packages.xml**。

#  问题描述
手机重启之后会有应用反复的crash，导致系统不能使用。
如果再次重启，问题可能就消失了。但是再次重启，可能又出现了。
现象看上去很诡异！

![](/imgs/Android-ABI-issue-analysis/cneserver_crash.png)



# 问题分析
crash的问题，一般都会有明显的log。

> 03-01 00:02:40.792  4133  4133 E AndroidRuntime: FATAL EXCEPTION: main
03-01 00:02:40.792  4133  4133 E AndroidRuntime: Process: com.quicinc.cne.CNEService, PID: 4133
03-01 00:02:40.792  4133  4133 E AndroidRuntime: java.lang.RuntimeException: Unable to instantiate application com.quicinc.cne.CNEService.CNEServiceApp: java.lang.ClassNotFoundException: Not find class "com.quicinc.cne.CNEService.CNEServiceApp" on path: DexPathList[[zip file "/system/framework/com.quicinc.cne.jar", zip file "/system/priv-app/CNEService/CNEService.apk"],nativeLibraryDirectories=[/system/priv-app/CNEService/lib/arm64, /system/priv-app/CNEService/CNEService.apk!/lib/armeabi-v7a, /vendor/lib, /system/lib]]
03-01 00:02:40.792  4133  4133 E AndroidRuntime: 	at android.app.LoadedApk.makeApplication(LoadedApk.java:578)
...

从log看，在apk中找不到class文件。

CNEService是一个系统内置应用，编译的是64位系统，user版本，有做odex，apk中的dex文件被删除。

![](/imgs/Android-ABI-issue-analysis/CNEService.jpg)


> 03-01 00:02:36.367  1284  1284 I PackageManager: Adjusting ABI for : com.quicinc.cne.CNEService to armeabi-v7a 

只有arm64文件夹且被认为是armeabi-v7a，系统找不到odex，到apk中去找，但是dex已经删除了，故进程崩溃。

**正常情况下，CNEService 的primaryCpuAbi应该arm64-v8a，为什么这里有时候就变成了armeabi-v7a？**

上面一节已经描述过primaryCpuAbi的来源了。
从log中可以知道CNEService 的ABI在函数adjustCpuAbisForSharedUserLPw中被确定下来的。
还发现有如下log：
>03-01 00:02:36.364  1284  1284 W PackageManager: Instruction set mismatch, PackageSetting{43d890 com.qti.smq.qualcommFeedback/1000} requires arm whereas PackageSetting{e6e3c66 com.caf.fmradio/1000} requires arm64

在函数adjustCpuAbisForSharedUserLPw被执行之前，一些package的ABI已经确定出来了。比如log中提到的qualcommFeedback和fmradio。

也就是说**qualcommFeedback，fmradio，CNEService **的**android:sharedUserId**是一样的，且在执行**adjustCpuAbisForSharedUserLPw**之前**qualcommFeedback**的ABI已经确定，是armeabi-v7a，而**fmradio**的ABI是arm64-v8a。但是因为**qualcommFeedback**是第一个被找到，故认为其它的没有确定ABI的packages的ABI和qualcommFeedback一样，即**CNEService**的ABI是armeabi-v7a。如果fmradio先被找到，就会认为其它的没有确定ABI的packages的ABI和fmradio一样，即**CNEService**的ABI是arm64-v8a。

至此，能解释上面诡异现象了。

- 当CNEService的ABI是arm64-v8a时，由于存在 arm64/CNEService.odex ，程序正常运行。

- 当CNEService的ABI是armeabi-v7a时，不存在 arm/CNEService.odex ，CNEService.apk 没有dex，程序崩溃。

修改方法有多种：
1. 同时生成arm/CNEService.odex 和arm64/CNEService.odex 
2. 保留apk中的dex文件
3. 修改**android:sharedUserId**
4. 把和CNEService的 **android:sharedUserId**一样的app都变成arm64-v8a。
......

通过对多份log的分析发现，qualcommFeedback的ABI始终是armeabi-v7a。

之前有提到一个native方法** findSupportedAbi**，这个函数经过高通修改之后会根据assets文件夹下的so决定得ABI。
打开APK之后，确实在assets下发现了一个32位的so库。

再没了所有疑惑之后，解决问题就不写了。


# REF
https://read01.com/BDyndB.html
https://my.oschina.net/bugly/blog/738360
https://my.oschina.net/liucundong/blog/653128
http://gityuan.com/2016/03/26/app-process-create/
https://en.wikipedia.org/wiki/Application_binary_interface
https://mssun.me/blog/android-art-runtime-2-dex2oat.html
http://www.iloveandroid.net/2016/06/27/Android_PackageManagerService-9/
