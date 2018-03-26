---
layout: post
title: ubuntu默认安装openjdk没有tools.jar
date: 2018-03-26 11:53:00 +0800
category: 学习记录

---

## 首先检查java版本

```java
java -version
```

结果：

![1]({{site.baseurl}}/assets/img/notes/20180326-jdk.img/1.png)

## 根据相应版本安装openjdk

```java
sudo apt-get install openjdk-8-jdk
```

如果系统中有多个jdk，可以利用如下命令来查看和选择

```java
sudo update-alternatives --config java
```

此时，tools.jar在

```java
/usr/lib/jvm/java-8-openjdk-amd64/lib/tools.jar
```

