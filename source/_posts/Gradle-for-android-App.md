---
title: 'Gradle for android App '
date: 2016-08-11 17:35:46
tags: Gradle
toc: false
---
# 1、gradle 支持sdkman安装

>curl -s https://get.sdkman.io | bash
>sdk install gradle 2.14.1


构建工具:
- 构建功能是基础
- 依赖管理是核心
- IDE集成是本分
- 扩展任务是前景
- 环境管理是情怀


**Ant -> Maven -> Gradle**


# 2、Gradle

[Gradle Wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html)实现了工具自管理，是有情怀的表现。

> gradlew
> gradlew.bat
>  gradle/wrapper/
>  gradle-wrapper.jar
>  gradle-wrapper.properties

上述文件建议用版本管理工具管理起来。
- 方便没有安装Gradle的机器使用
- wrapper可以定制，尽管一般情况下没有做定制
- gradle-wrapper.properties中记录了Gradle的版本信息

可以通过命令生成：
>gradle wrapper --gradle-version XXX



涉及的配置文件
1. settings.gradle
2. build.gradle
3. gradle.properties
4. local.properties


**settings.gradle**
>The **settings.gradle** file, located in the root project directory, tells Gradle which modules it should include when building your app. 

**build.gradle**
>The **top-level build.gradle** file, located in the root project directory, defines build configurations that apply to all modules in your project. By default, the top-level build file uses the buildscript {} block to define the Gradle repositories and dependencies that are common to all modules in the project. 

>The **module-level build.gradle** file, located in each  < project >/< module >/ directory, allows you to configure build settings for the specific module it is located in. 

**gradle.properties**
>This is where you can configure project-wide Gradle settings, such as the Gradle daemon's maximum heap size. For more information, see [Configuring the build environment](https://docs.gradle.org/current/userguide/build_environment.html).

**local.properties**
>Configures local environment properties for the build system, such as the path to the SDK installation.

上面4个文件，除了local.properties，都应该提交到版本管理仓库。

**.gitignore**的配置可以直接在github上拿一个Android项目的默认配置。不过默认没把**.idea/***去掉，需要添加。


# 3、github上新建一个Android项目

- Create a new repository
在github上新建一个项目，添加好.gitignore ，选择好license。

- Android studio 新建工程
找到AS新建的工程目录，执行**git init**，然后**git remote add origin  XXXXX && git remote update**
最后 **git checkout master**
配置好.gitignore, 提交到github，初始化工作就算完成了。

