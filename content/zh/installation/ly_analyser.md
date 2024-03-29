---
title: 分析引擎编译安装
linkTitle: 分析引擎
description: 流影分析引擎模块ly_analyser编译安装说明
date: 2022-12-22
publishdate: 2022-12-22
categories: [installation,compile]
menu:
  docs:
    parent: installation
    weight: 20
toc: true
weight: 20
isCJKLanguage: true
---

ly_analyser是流影的威胁行为分析引擎，读取netflow v9格式的数据作为输入，运行各种威胁行为检测模型，产出威胁事件，并留存相关特征数据用于后续取证分析。包括扫描、DGA、DNS隧道、ICMP隧道、服务器外联、 挖矿、各种注入等威胁行为，涵盖机器学习、威胁情报、数据包检测、经验模型四种识别方式。

## 项目地址

* [Github](https://github.com/Abyssal-Fish-Technology/ly_analyser)
* [Gitee](https://gitee.com/abyssalfish-os/ly_analyser)

## 目录结构

	ly_analyser
	├─── dependence		--依赖环境
	├─── src
		├───common		-- 数据结构与通用工具函数
		├───nfdump		-- 第三方数据接收工具
		├───agent		-- 引擎主体代码
			├───config		
			├───data		--数据库操作模块
			├───dump		--netflow数据解析模块
			├───flow		--检测模型模块
			├───handles		--数据处理模块
			├───indexing	--程序索引模块
			├───model		
			├───utils		
			......



## 环境依赖

操作系统
: CentOS-7-x86_64-Minimal-2009 
	
编译环境
:  gcc、gcc-c++、cmake
	bison、flex
	json-c-devel、
	boost-devel、
	libcurl-devel、
	libpcap-devel、
	mariadb-devel、
	httpd、ntp、
	cgicc-3.2.16、
	cppdb-0.3.1、
	protobuf-3.8.0、
	tensorflow-2.0.4、


## 安装依赖环境

```shell
# 1. 安装依赖组件
	yum install gcc gcc-c++ cmake -y
	yum install bison flex json-c-devel -y
	yum install ntp -y
	yum install httpd -y
	yum install boost-devel -y
	yum install libcurl-devel -y
	yum install mariadb-devel -y
	yum install libpcap-devel -y
		
# 2. 编译安装cgicc 
	tar -zxvf cgicc-3.2.16.tar.gz -C ./
	cd ./cgicc-3.2.16
	./config
	make && make install
	
# 3. 编译安装cppdb
	tar -jxvf cppdb-0.3.1.tar.bz2 -C ./
	cd ./cppdb-0.3.1
	cmake -DCMAKE_INSTALL_PREFIX=/usr -DLIBDIR=lib64 -DMYSQL_LIB=/usr/lib64/mysql/libmysqlclient.so -DMYSQL_PATH=/usr/include/mysql 
	make && make install
	
# 4. 编译安装protobuf-3.8.0
	tar -xzvf protobuf-3.8.0.tar.gz
	./configure
	make && make install
	ln -sf /usr/local/lib/libprotobuf.so.19.0.0 /usr/lib64/libprotobuf.so.19

# 5. tensorflow-2.0.4相关头文件、库安装
	tar -xzvf tf.tar.gz
	cp tf /usr/local/include -r
	tar -xzvf tf_lib.tar.gz 
	cd tf_lib
	cp libtensorflow_framework.so.2.0.4 libtensorflow_cc.so.2.0.4 /usr/local/lib
	ln -sf /usr/local/lib/libtensorflow_framework.so.2.0.4 /usr/local/lib/libtensorflow_framework.so.2
	ln -sf /usr/local/lib/libtensorflow_framework.so.2 /usr/local/lib/libtensorflow_framework.so
	ln -sf /usr/local/lib/libtensorflow_cc.so.2.0.4 /usr/local/lib/libtensorflow_cc.so.2
	ln -sf /usr/local/lib/libtensorflow_cc.so.2 /usr/local/lib/libtensorflow_cc.so
	ln -sf /usr/local/lib/libtensorflow_cc.so.2.0.4 /usr/lib64/libtensorflow_cc.so.2
	ln -sf /usr/local/lib/libtensorflow_framework.so.2.0.4 /usr/lib64/libtensorflow_framework.so.2
```



## 编译部署

```shell
# 6. 创建目录
	mkdir -p /home/Agent
	ln -s /home/Agent /Agent

	mkdir -p /home/data/flow/
	ln -s /home/data /data
	ln -s /data/flow /Agent/flow

# 7. 编译源代码
	cd src/
	# 编译common
	cd common/
	make && make install
	
	# 编译agent
	cd agent/
	make && make instal
	
	# 编译nfdump
	cd nfdump/
	./configure
	make 
	cp bin/nfcapd bin/nfdump /Agent/bin
	
```



## 运行环境配置

```shell
# 8. 配置环境语言及时区
	export LANG=en_US.UTF-8
	ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
	ntpdate cn.pool.ntp.org 
	
# 9. 关闭seliunx，开放本地防⽕墙端口
	#编辑config⽂件
	vi /etc/selinux/config
	#找到配置项
	SELINUX=enforcing
	#修改配置项为
	SELINUX=disabled
	
	#执⾏命令，即时关闭selinux
	setenforce 0 

	#开放本地防⽕墙端口
	systemctl restart firewalld
	firewall-cmd --zone=public --add-port=10081/tcp --permanent
	firewall-cmd --reload

# 10. 配置httpd
	 编辑文件/etc/httpd/conf.d/agent.conf，写入内容：
	 Listen 10081
	 <VirtualHost *:10081>
	     DocumentRoot /Agent/cmd
	     <Directory "/Agent/cmd">
	         Options ExecCGI
	         SetHandler cgi-script
	         AllowOverride None
	         Order allow,deny
	         Allow from all
	         Require all granted
	     </Directory>
	 </VirtualHost>
	 
	 #重启httpd
	 systemctl restart httpd
```



## 运行任务配置

``` sell
# 11. 创建定时任务
	vi /var/spool/cron/apache，加入内容：
	*/5 * * * * /Agent/bin/extractor
	 
# 12. 启动nfcapd接收探针发送的netflow数据
	/Agent/bin/nfcapd -w -D -l /data/flow/3 -p 9995
```

## 开源许可

* GNU GENERAL PUBLIC LICENSE  Version 3

## 联系方式

如果在开发、部署、使用的过程中遇到任何问题，或者技术讨论、产品咨询、商务合作等，都欢迎前来联系我们！

联系邮箱：[opensource@abyssalfish.com.cn](mailto:opensource@abyssalfish.com.cn)



