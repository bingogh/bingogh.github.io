---
category: Skills-2016
published: true
layout: post
title: 利用 OpenLDAP 实现 MQ Client 的连接认证
description: 咦，网上怎么没有介绍它俩连接的文章，虽然我写的也不详细，就先这样吧
---

## 1. 实验环境
[Ubuntu 16.4 LTS - 64 bit](http://www.ubuntu.org.cn/download/desktop)  
[OpenLDAP - 2.4.44](http://www.openldap.org/software/download/)  

---  

WebSphere MQ Advanced for Developers (Free ^_^)  
[mqadv\_dev80\_linux_x86-64.tar.gz](https://www.ibm.com/developerworks/community/blogs/messaging/entry/develop_on_websphere_mq_advanced_at_no_charge?lang=en)  
[Installing IBM MQ server on Linux](http://www.ibm.com/support/knowledgecenter/SSFKSJ_8.0.0/com.ibm.mq.ins.doc/q008640_.htm)  
[Uninstalling IBM MQ on Linux](http://www.ibm.com/support/knowledgecenter/SSFKSJ_8.0.0/com.ibm.mq.ins.doc/q009440_.htm)

---   


## 2. OpenLDAP  

### 2.1 安装  
Dependence：libdb ( apt-get install libdb5.3-dev )  
Install: configure -> make depend -> make install

### 2.2 配置  

#### 2.2.1 sldap.conf  
OpenLDAP 配置文件在 /usr/local/etc/openldap 目录下，slapd.conf 是slapd 服务配置文件。  
  
slapd.conf 文件：  
	
	# Modified by bingo	# date 2016.7.23
	#
	# See slapd.conf (5) for details on configuration options.
	#
	include         /usr/local/etc/openldap/schema/core.schema
	include         /usr/local/etc/openldap/schema/cosine.schema
	include         /usr/local/etc/openldap/schema/inetorgperson.schema
	include         /usr/local/etc/openldap/schema/nis.schema

	loglevel        256
	pidfile         /usr/local/var/run/slapd.pid
	argsfile        /usr/local/var/run/slapd.args

	# The next three lines allow use of TLS for encrypting connections.
	# TLSCipherSuite  HIGH:MEDIUM:+SSLv2
	# TLSCACertificateFile    /usr/local/etc/openldap/cacerts/cacert.pem
	# TLSCertificateFile      /usr/local/etc/openldap/slapdcert.pem
	# TLSCertificateKeyFile   /usr/local/etc/openldap/slapdkey.pem

	# access control policy:
	# Restrict password access to change by owner and authentication.
	# Allow read access by everyone to all other attributes.

	access to attrs=shadowLastChange,userPassword
	   by self write
	   by * auth
	
	access to *
	   by * read
	
	#######################################################################
	# database definition
	#######################################################################
	
	database        bdb
	suffix          "dc=bingo,dc=com"
	
	rootdn          "cn=Manager,dc=bingo,dc=com"
	rootpw          secret
	
	directory       /usr/local/var/openldap-data
	
	# Indices to maintain for this database
	index objectClass                       eq,pres
	index ou,cn,mail,surname,givenname      eq,pres,sub
	index uidNumber,gidNumber,loginShell    eq,pres
	index uid,memberUid                     eq,pres,sub
	index nisMapName,nisMapEntry            eq,pres,sub


#### 2.2.2 log 配置  
loglevel 行设置的是日志选项。  
在系统日志配置中加入 ldap 日志路径  
>vi /etc/rsyslog.conf  
	
	local4.*	/var/log/ldap.log

重启日志服务  
>service rsyslog restart  

#### 2.2.3 启动 ldap 服务  
启动 ldap 服务并查看  
>/usr/local/libexec/slapd -f slapd.conf  
>ps -ef | grep slapd 

#### 2.2.4 添加 inetOrgPerson 信息  

addmqm.ldif 文件  

	# written by bingo	# date 2016.7.23	# usage: ldapadd -x -D 'cn=Manager,dc=bingo,dc=com' -w secret -f mqm.ldif	dn: dc=bingo,dc=com	objectClass: dcObject	objectClass: organization	o: bingo	dc: bingo	description: this is my domain	dn: uid=mqm,dc=bingo,dc=com	objectClass: Person	objectClass: inetOrgPerson	uid: mqm	sn: bingo	cn: yuan	userPassword: teamsun	telephoneNumber: 10010	mail: bingobox@qq.com	dn: uid=root,dc=bingo,dc=com	objectClass: Person	objectClass: inetOrgPerson	uid: root	sn: root	cn: root	userPassword: teamsun	telephoneNumber: 10086	mail: root@qq.com  


## 3. IBM MQ  

### 3.1 QM 配置  
Client 是通过 server-connection 来连接到 QM 的，需要定义的参数如下：  

	***Create QM1***  
	crtmqm -p 1414 QM1   
	strmqm QM1 

	***runmqsc QM1***  
	DEFINE CHANNEL(CLIENT.CHL) CHLTYPE(SVRCONN) REPLACE  
	DEFINE AUTHINFO(LDAP) AUTHTYPE(IDPWLDAP) CONNAME('127.0.0.1(389)') BASEDNU('dc=ibm,dc=com') SHORTUSR('uid') REPLACE  
	ALTER QMGR CONNAUTH(LDAP)  
	ALTER QMGR CHLAUTH(DISABLED)  
	REFRESH SECURITY  

### 3.2 Client 连接测试  
使用 amqscnxc 来连接 QM  
	
	amqscnxc -x '127.0.0.1' -c 'CLIENT.CHL' -u mqm

![amqscnxc result](/images/OpenLDAP-1.jpg)  
### 参考  
1. [使用 OpenLDAP 集中管理用户帐号](http://www.ibm.com/developerworks/cn/linux/l-openldap/)  