---
title: maven
author: essviv
date: 2016-10-04 10:20:54+0800
---

# Maven

## 生命周期

maven中将整个项目构建抽象成一系列的生命周期，具体每个周期的实现交由插件实现，这点可以参照设计模式中“模板方法”的实现.

maven的生命周期分为三套：clean， default以及site， 分别对应于清理、构建以及建立项目站点三个环节.

不同的生命周期又可以进一步划分为不同的阶段(phase)，在同一个生命周期内，后面的阶段依赖于前面的阶段. 不同的生命周期不会相互影响.

e.g. clean周期包括pre-clean, clean和post-clean三个阶段，如果调用了clean:clean阶段，则clean:pre-clean和clean:clean都会被调用.

 

## 生命周期的不同阶段

maven中三个不同的生命周期又可以进一步划分为不同的阶段.

* clean: pre-clean, clean, post-clean

* default: 校验，初始化、编译、测试、打包、集成测试、安装、部署

	* validate, initialize,

	* generate-source, process-source, generate-resource, process-resource, compile, process-classes, 

	* generate-test-source, process-test-source, generate-test-resource, process-test-resource, test-compile, process-test-classes, test,

	* prepare-package, package,

	* pre-integration-test, integration-test, post-integration-test,

	* verify, install, deploy

* site: pre-site, site, post-site, site-deploy

 

## 常见的插件

1. http://www.infoq.com/cn/news/2011/04/xxb-maven-7-plugin/

    