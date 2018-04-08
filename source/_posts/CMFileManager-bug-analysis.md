---
title: CMFileManager bug analysis
date: 2016-06-13 20:32:38
tags: [CMFileManager, Android]
---
# 0x00 bug现象
----
通过CMFileManager 移动或者复制文件夹的时候，在PC端看到的文件夹图标显示不正确。
![](/imgs/CMFileManager-bug-analysis/before-pc.png)

![](/imgs/CMFileManager-bug-analysis/before-phone1.png)


![](/imgs/CMFileManager-bug-analysis/after-pc.png)

![](/imgs/CMFileManager-bug-analysis/after-phone1.png)

# 0x01 初步分析
----
手机端看上去是正常的，电脑端显示不正常。

手机和PC是通过USB for file transfer 功能连接的。该功能核心就是MTP，[Media Transfer Protocol（媒体传输协议）](https://en.wikipedia.org/wiki/Media_Transfer_Protocol)。

关于Android的MTP介绍，网上这篇[Android之 MTP框架和流程分析](http://www.cnblogs.com/skywang12345/p/3474206.html)可以用来学习。

PC端显示的数据都是从手机端拿到的。其中很重要的一个文件是MtpServer.cpp。

一般情况下，在文件更新的时候，比如新建文件 ，删除文件，移动文件等，MediaProvider数据库会发生变化。之所以叫一般情况，是因为一个进程可以新建文件 但是不更新MediaProvider数据库。数据库不更新，依赖该数据库的应用就不能及时获取文件信息。MTP就需要用到MediaProvider数据库。

当数据更新的时候，MTP就会主动同步手机和PC的信息。比如移动文件之后，数据会更新，PC端就要重新获取相关信息。

CMFileManager文件管理在新建文件 ，删除文件，移动文件等操作之后会主动通知MediaScannerService去扫描变更的目录。MediaScannerService会根据扫描的情况更新数据库。


下面列出出现这个bug的可能原因：
1. PC端获取到了正确数据但是显示不正确，PC端的软件bug `概率极小`
2. PC端没有获取到正确数据  `概率大`
3. 更新数据库之前的数据就不正确 `概率大`
4. MediaScannerService扫描的结果不正确 `概率小`

# 0x02 PC端获取的数据是否正确
手机端的信息都是通过下面这个函数给到PC端的
```cpp
MtpResponseCode MyMtpDatabase::getObjectInfo(MtpObjectHandle handle,MtpObjectInfo& info) {
    MtpString       path;
    int64_t         length;
    MtpObjectFormat format;

    MtpResponseCode result = getObjectFilePath(handle, path, length, format);
    if (result != MTP_RESPONSE_OK) {
        return result;
    }
    info.mCompressedSize = (length > 0xFFFFFFFFLL ? 0xFFFFFFFF : (uint32_t)length);

    JNIEnv* env = AndroidRuntime::getJNIEnv();
    if (!env->CallBooleanMethod(mDatabase, method_getObjectInfo,
                (jint)handle, mIntBuffer, mStringBuffer, mLongBuffer)) {
        return MTP_RESPONSE_INVALID_OBJECT_HANDLE;
    }

    jint* intValues = env->GetIntArrayElements(mIntBuffer, 0);
    info.mStorageID = intValues[0];
    info.mFormat = intValues[1];
    info.mParent = intValues[2];
    ...
```

这个函数Call了一个java的方法method_getObjectInfo

```java
private boolean getObjectInfo(int handle, int[] outStorageFormatParent,char[] outName, long[] outCreatedModified) {
        Cursor c = null;
        try {
            c = mMediaProvider.query(mPackageName, mObjectsUri, OBJECT_INFO_PROJECTION,
                            ID_WHERE, new String[] {  Integer.toString(handle) }, null, null);
            if (c != null && c.moveToNext()) {
                outStorageFormatParent[0] = c.getInt(1);
                outStorageFormatParent[1] = c.getInt(2);
                outStorageFormatParent[2] = c.getInt(3);
```

通过在这些地方打log ，把object的信息打印出来，最后发现文件的格式不对，也就是`outStorageFormatParent[2]`不是期望值。
其实从PC端显示的图标也能猜测下，可能是文件格式不对。名字，日期什么的都正确。

到这里可以知道，给到PC端的数据就不正确。

既然这个数据的来源是数据库，所以直接拿数据库来看。
![](/imgs/CMFileManager-bug-analysis/database.png)
可以看到，文件格式确实不对了。
`0x3000 = 12288`
`0x3001 = 12289`
```java
/** Undefined format code */
public static final int FORMAT_UNDEFINED = 0x3000;

/** Format code for associations (folders and directories) */
public static final int FORMAT_ASSOCIATION = 0x3001;
```

# 0x03 谁的错误
----

数据库的信息是由MediaScannerService插入数据库，相关的代码逻辑应该有文件格式的判断，比如根据后缀名判断，根据文件魔数判断。问题可能出在代码逻辑上，也可能是本来就是文件，MediaScannerService如实的把格式写成文件。
为此直接用shell查看了下文件信息。
![](/imgs/CMFileManager-bug-analysis/shell.png)

可以看到，`DCIM`下的`11` 确实是一个文件夹。

只能说明MediaScannerService判断错误了。但是为什么CMFileManager新建一个文件夹也是正常的呢？

下面是创建一个文件夹的关键代码
```java
public static boolean createDirectory(Context context, String directory, Console console) {
        Console c = ensureConsoleForFile(context, console, directory);
        CreateDirExecutable executable =
                c.getExecutableFactory().newCreator().createCreateDirectoryExecutable(directory);
        writableExecute(context, executable, c);

        // Do media scan
        Bundle args = new Bundle();
        args.putString(VOLUME, EXTERNAL);
        Intent startScan = new Intent();
        startScan.putExtras(args);
        startScan.setComponent(new ComponentName("com.android.providers.media",
                "com.android.providers.media.MediaScannerService"));
        context.startService(startScan);

        return executable.getResult().booleanValue();
    }
```
下面是复制的关键代码
```java
public static boolean copy(Context context, String src, String dst, Console console) {

        Console cSrc = ensureConsoleForFile(context, console, src);
        Console cDst = ensureConsoleForFile(context, console, dst);
        boolean ret = true;
        
            CopyExecutable executable =
                    cSrc.getExecutableFactory().newCreator().createCopyExecutable(src, dst);
            writableExecute(context, executable, cSrc);
            ret = executable.getResult().booleanValue();
        

        // Do media scan (don't scan the file if is virtual file)
        if (ret) {
            if (!VirtualMountPointConsole.isVirtualStorageResource(dst)) {
                recursiveScan(context, null, dst);
            }
        }

        return ret;
    }
```
可以看到，在通知MediaScannerService更新数据库的代码是不一样的。

函数recursiveScan最关键的一句代码如下：
```java
MediaScannerConnection.scanFile(...
```
通过查阅MediaScannerService的源码可以知道，这两种方式是极为不同的，一个是整个`external volume`去scan，一个是直接scanFile。

MediaScannerService对外提供的接口极少，其它应用要么通知整个volume扫描，要么就扫描指定文件。 比如拍照 ，截屏，就用scanFile接口，挂载磁盘就会用scan接口。

- 可以去修改MediaScannerService的代码逻辑，让scanFile接口增加对文件夹的判断。

- 也可以去修改CMFileManager的通知方式。

看起来都有错，但是MediaScannerService作为系统代码还是不改为好。
最终的结果就是修改了CMFileManager的代码

# 0x04 修改
修改已经提交到CM 
http://review.cyanogenmod.org/#/c/149144/
![](/imgs/CMFileManager-bug-analysis/modify.png)

# 0x05 相关链接

http://review.cyanogenmod.org/#/c/149144/
http://www.cnblogs.com/skywang12345/p/3474206.html
