---
title: maven学习笔记
author: essviv
date: 2016-10-04 10:20:54+0800
---

# Maven学习笔记

## 构件的坐标

groupId, artifactId, version

## 管理依赖 
    
### 通过构件的坐标来唯一指定

````java
	<dependency>
          <groupId></groupId>
          <artifactId></artifactId>
          <version></version>  //快照版本： SNAPSHOT
          <scope></scope> //可取值包括: compile, runtime, provided, test, system(不常用)
     </dependency>
````

### 间接依赖 ===> 依赖冲突

* 路径最短
	````     
	A  ==>  B  ==>  C(V1.0)（selected)        A  ==>  D  ==>  E  ==> C(V2.0)
	````
* 声明优先
	````
	A  ==>  B  ==>  C(V1.0) (selected)       A  ==>  D  ==>  C(V2.0)
	````

## 生命周期和阶段（抽象概念， 具体实现由指定的插件目标决定）
     
### 生命周期
* clean: pre-clean, clean, post-clean
* default: process-resources, compile, process-test-resources, test-compile, test, package, install, deploy
* site: pre-site, site, post-site, site-deploy

### 阶段
 
同一个周期内的阶段按顺序执行，不同周期的阶段没有先后顺序关系

* mvn clean: 执行clean周期的pre-clean, clean两个阶段
* mvn clean test: 执行clean周期的pre-clean， clean， 然后再执行default周期的process-resources, compile……test
* mvn clean package:  执行clean周期的pre-clean, clean，然后再执行default周期的process-resources ... package

### 插件

生命周期定义的阶段均为抽象的概念，具体的操作由插件目标来实现，maven为一些关键的阶段定义了默认的插件目标

* 默认绑定： maven-clean-plugin: clean  ===> clean

* 自定义绑定： 将指定坐标的构件的A目标绑定到test阶段

	````
	 <plugin>
	      <groupId></groupId>
	      <artifactId></artifactId>
	      <version></version>
	      
	      <executions>
	           <execution>
	                <id></id>
	                <phase>test</phase> //指定绑定的阶段
	                <goals>
	                     <goal>A</goal>  //定义插件的目标
	                </goals>
	           </execution>
	      </excutions>
	 </plugin>
	````

## 聚合和继承
     
### 聚合: 一次性构建多个项目
     
````
 <groupId />
 <artifactId />
 <version />
 <packaging>POM</packaging> //聚合项目的打包方式必须为POM
 
 <modules>
      <module /> //第一个被聚合的项目
      <module /> //第二个被聚合的项目
 </modules>
````

### 继承

继承的目的是为了减少重复的配置，子项目会从父项目中继承相应的配置，如果有必要，子项目也可以重写一些配置属性

```
父POM：
  <groupId />
  <artifactId />
  <version />
  <packaging>POM<packaging> //父工程的打包方式也必须为POM

子POM
  <artifactId />
  <parent>
       <groupId />
       <artifactId />
       <version />
       <relativePath></relativePath> //父POM的路径
  </parent>
```

## 多环境构建： 基于属性, filter和profile实现, 通过${}访问
### 属性

* 自定义属性: <helloWorld>JAVA</helloWorld>

* POM属性: ${project.baseDir}, ${project.artifactId}, ${project.build.sourceDirectory}

### filter

资源过滤： 解析资源文件中的Maven属性

````
<resources>
  <resource>
       <directory />
       <filtering>true</filtering>
  </resource>
</resources>
````

### profile

针对不同的环境采用不用的属性, 命令行激活， 如： -Pdev

````
<profiles>
  <profile>
       <id>dev</id>
       <properties>
            <msg>hello world</msg>
       </properties>

       <activation>
            <activateByDefault>true</activateByDefault> //默认启用的profile
       </activation>
  </profile>

  <profile>
       <id>prod</id>
       <properties>
            <msg>Holy god!</msg>
       </properties>
  </profile>
</profiles>
````

## 约定大于配置（Convention Over Configuration）
````
src/main/java, 
src/main/resources, 
src/test/java, 
src/test/resources
````

## 参考文献

1. maven实战  机械工业出版社  许晓斌
