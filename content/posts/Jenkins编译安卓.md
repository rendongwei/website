---
title: "Jenkins 编译安卓"
date: "2022-10-26"
categories:
    - "技术"
tags:
    - "Jenkins"
    - "Android"
toc: false
indent: false
original: true
draft: true
---

## 更新记录

| 时间       | 内容 |
| ---------- | ---- |
| 2022-10-26 | 初稿 |

## 软件版本

| soft    | Version   |
| ------- | --------- |
| Jenkins | 2.346.2   |
| SDK     | 3859397   |
| JDK     | 1.8.0_151 |

## 一、配置编译环境

①、SDK

``` zsh
➜  cd /usr/local

# 下载 && 安装
➜  wget https://dl.google.com/android/repository/sdk-tools-linux-3859397.zip
➜  unzip sdk-tools-linux-3859397.zip -d android-sdk

# SDK 安装
➜  cd android-sdk/tools/bin
➜  ./sdkmanager --list
➜  ./sdkmanager "build-tools;30.0.3" "platforms;android-32" "platform-tools"
➜  ./sdkmanager --licenses
```

②、gradle

``` zsh
➜  wget https://downloads.gradle-dn.com/distributions/gradle-7.3.3-all.zip
➜  unzip gradle-7.3.3-all.zip
```

③、配置环境变量

``` zsh
➜  vim /etc/profile
export ANDROID_HOME=/usr/local/android-sdk/
export PATH=$PATH:$ANDROID_HOME:/usr/local/gradle-7.3.3/bin

➜  source /etc/profile
```

## 二、配置自动化流水线

①、Jenkinsfile

配置 Jenkins Pipeline

``` zsh
➜  vim android-nrmm.groovy
pipeline{
    agent any

    options {
        // 保存最近历史构建记录的数量，只能在pipeline下使用
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // 禁止当前pipeline同时多次执行
        disableConcurrentBuilds()
    }

    environment {
        workdir = "$WORKSPACE"

        ProjectName = "非道路安卓(Android)"
        DownloadURL = "dlapk.gongjiangren.net"
    }

    parameters {
        choice choices: ['origin/master'], description: '选择分支', name: 'branch'
    }

    stages{
        stage('钉钉推送'){
            steps{
                sh "sh -x /script/test-service/jenkins-tools/dingding-test/nrmm_start.sh"
            }
        }
        stage("拉取代码"){
            steps{
                git credentialsId: '', url: 'http://172.31.229.139:6088/nrmm/android.git'
            }
        }
        stage("版本控制"){
            steps{
                sh "sh -e /script/test-service/jenkins-tools/clone_code.sh"
            }
        }
        stage("打包 && 发布 Nginx"){
            steps{
                sh "sh -x /script/test-service/nrmm/android/build_apk.sh"
            }
        }
    }

    post {
        success {
            sh "sh -x /script/test-service/jenkins-tools/dingding-test/nrmm_android_end.sh"
        }
        failure {
            sh "sh -x /script/test-service/jenkins-tools/dingding-test/nrmm_failed.sh"
        }
    }

}
```

②、编译脚本

``` zsh
# 编译 apk 脚本
➜  vim build_apk.sh
#!/bin/bash

# 失败退出
set -e
source /etc/profile

remote_dir="/var/www/dlapk"

# 编译
sed -i 's/^#org.gradle.java.home/org.gradle.java.home/g' $workdir/gradle.properties

cd $workdir/app

    /usr/local/gradle-7.3.3/bin/gradle assembleRelease

if [ $? -ne 0 ];
then
    echo -e  "\t编译 apk 失败! \n\t请查看日志信息或联系运维人员处理"
    exit -1
fi

#-------------------------------------------------------------------
# 传输

# 远程主机路径检测
ssh root@172.31.229.139 "if [ ! -d $remote_dir ]; then echo "$remote_dir: directory does not exist!"; echo "mkdir -p $remote_dir"; mkdir -p $remote_dir; fi"
scp $workdir/app/build/outputs/apk/release/*.apk root@172.31.229.139:$remote_dir

if [ $? -ne 0 ];
then
    echo -e  "\t传输 apk 失败! \n\t请查看日志信息或联系运维人员处理"
    exit -1
fi

# 清理 apk
rm -f $workdir/app/build/outputs/apk/release/*.apk

```

③、Nginx config

配置 Nginx 为文件服务器的配置文件

``` zsh
➜  cat base__dlapk.gongjiangren.net__.conf
server {
        listen 80;
        server_name dlapk.gongjiangren.net;

        # 打开目录浏览功能
        autoindex on;

        # 设置为on时, 显示文件的确切大小, 单位是bytes
        # 设置为off时, 显示文件的大概大小
        autoindex_exact_size off;

        # 设置为on时, 显示文件的服务器时间
        # 设置为off时, 显示文件的时间为GMT时间
        autoindex_localtime on;

        root /var/www/dlapk;

}
```

## 三、问题处理

①、gradle 与 JDK 版本不兼容

``` zsh
➜  gradle assembleRelease

FAILURE: Build failed with an exception.

* Where:
Build file '/root/miaocunfa/android/app/build.gradle' line: 2

* What went wrong:
An exception occurred applying plugin request [id: 'com.android.application']
> Failed to apply plugin 'com.android.internal.application'.
   > Android Gradle plugin requires Java 11 to run. You are currently using Java 1.8.      # 需要使用 Java 11
     Your current JDK is located in  /usr/local/jdk1.8.0_151/jre
     You can try some of the following options:
       - changing the IDE settings.
       - changing the JAVA_HOME environment variable.
       - changing `org.gradle.java.home` in `gradle.properties`.

* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 961ms
```

问题处理

``` zsh
# 修改项目下的 gradle.properties
➜  vim gradle.properties
org.gradle.java.home=/usr/local/jdk-11.0.2/
```

②、Failed to install the following Android SDK packages as some licences have not been accepted

``` zsh
Checking the license for package Android SDK Build-Tools 30.0.3 in /usr/local/android-sdk/tools/licenses
Warning: License for package Android SDK Build-Tools 30.0.3 not accepted.
Checking the license for package Android SDK Platform 32 in /usr/local/android-sdk/tools/licenses
Warning: License for package Android SDK Platform 32 not accepted.

FAILURE: Build failed with an exception.

* What went wrong:
Could not determine the dependencies of task ':app:processReleaseResources'.
> Failed to install the following Android SDK packages as some licences have not been accepted.
     platforms;android-32 Android SDK Platform 32
     build-tools;30.0.3 Android SDK Build-Tools 30.0.3
  To build this project, accept the SDK license agreements and install the missing components using the Android Studio SDK Manager.
  Alternatively, to transfer the license agreements from one workstation to another, see http://d.android.com/r/studio-ui/export-licenses.html
  
  Using Android SDK: /usr/local/android-sdk/tools

* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --debug option to get more log output.
> Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 2s
```

问题处理

``` zsh
# 由 tools 下没有 licenses 文件夹, 而上一层级下有 licenses 文件夹, 检查出环境变量 $ANDROID_HOME 设置有问题
Checking the license for package Android SDK Platform 32 in /usr/local/android-sdk/tools/licenses         

➜  vim /etc/profile
export ANDROID_HOME=/usr/local/android-sdk/
```

> 参考列表:  
> 
> - [SDK license更新](https://blog.csdn.net/watson2017/article/details/120527084)  
> - [Nginx 文件服务器](https://dandelioncloud.cn/article/details/1487563088084979714)  
>
