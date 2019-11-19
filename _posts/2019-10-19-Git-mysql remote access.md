---
layout:     post
title:      Mysql远程访问
subtitle:   mysql配置远程访问
date:       2019-10-19
author:     Wilson
catalog: true
tags:
    - Mysql
---

>mysql remote access


# open remote access

	vi /etc/mysql/mysql.conf.d/mysqld.cnf (ubuntu)
	vi /usr/local/etc/my.cnf (Mac)
	bind-address            = *


# Grant permission in mysql

#### 创建仓库（初始化）
	GRANT ALL ON testdb.* TO testuser@'xx.xx.xx.xx' IDENTIFIED BY 'testpassword';
	FLUSH PRIVILEGES;
	
	#For ex: let root account to be accesable from remote
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'your root password';
	
#### Restart mysql
	#ubuntu
	sudo /etc/init.d/mysql restart
	 
	#mac
	brew services start mysql
	or
	mysql.server start