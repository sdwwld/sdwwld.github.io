---
layout: post
title: "Error(72) error unknown element user-permission found"
subtitle: 'Error(72) error unknown element user-permission found'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “山中有直树，世上无直人。”

## 正文

android studio升级之后会出现这样一个问题，Error:(72) error: unknown element <user-permission> found.

解决方法是在项目的gradle.properties文件中添加

android.enableAapt2=false
如果项目下没有gradle.properties文件可以新建个

```
# Project-wide Gradle settings.
 
# IDE (e.g. Android Studio) users:
# Gradle settings configured through the IDE *will override*
# any settings specified in this file.
 
# For more details on how to configure your build environment visit
# http://www.gradle.org/docs/current/userguide/build_environment.html
 
# Specifies the JVM arguments used for the daemon process.
# The setting is particularly useful for tweaking memory settings.
org.gradle.jvmargs=-Xmx1536m
 
# When configured, Gradle will run in incubating parallel mode.
# This option should only be used with decoupled projects. More details, visit
# http://www.gradle.org/docs/current/userguide/multi_project_builds.html#sec:decoupled_projects
# org.gradle.parallel=true
android.enableAapt2=false
```

