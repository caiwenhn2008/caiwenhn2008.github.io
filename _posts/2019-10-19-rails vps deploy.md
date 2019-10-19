---
layout:     post
title:      使用capistrano部署Rails到vps
subtitle:   使用capistrano部署Rails(nginx+passenger)到ubuntu
date:       2019-10-19
author:     BY
header-img: img/post-bg-debug.png
catalog: true
tags:
    - rails
    - capistrano
    - mysql
    - ruby
---


> 使用capistrano部署Rails(nginx+passenger)到Vultr ubuntu
> - vultr ubuntu 16.06
> - ruby 2.5.5
> - rails 5.1.7
> - mysql 8
> - capistrano 3.11.0

> Sample project using this steps to depoy to vultr ubuntu:
> <https://github.com/caiwenhn2008/myapp/>


## 新增 deploy 用户
	adduser --disabled-password deploy
	su deploy

## SSH 免密登入设置
	#本机
	cd ~
	ssh-keygen -t rsa
	 
	接着复制本机的 ~/.ssh/id_rsa.pub的内容到 服务器的/home/deploy/.ssh/authorized_keys
	这样本机就可以直接 ssh deploy@&lt;主机IP位置&gt;，登入无须密码
	 
	#远程服务器赋予ssh key权限
	#ubuntu
	chmod 644 /home/deploy/.ssh/authorized_keys
	chown deploy:deploy /home/deploy/.ssh/authorized_keys
	 
	#centos
	chmod 700 /home/deploy/.ssh 


## 更新和安装ubuntu系统
 
	apt-get update
	apt-get upgrade -y
	 
	#set system timezone
	dpkg-reconfigure tzdata
	 
	#setup lib requred by rails
	sudo apt-get install -y build-essential git-core bison openssl libreadline6-dev curl zlib1g zlib1g-dev libssl-dev libyaml-dev libsqlite3-0 libsqlite3-dev sqlite3  autoconf libc6-dev libpcre3-dev curl libcurl4-nss-dev libxml2-dev libxslt-dev imagemagick nodejs libffi-dev

## 安装 Ruby 
	apt-get install software-properties-common
	apt-add-repository ppa:brightbox/ruby-ng 
	apt-get update 
	apt-get install ruby2.5 ruby2.5-dev
	gem install bundler

## 安装MySQL数据库
	apt-get install mysql-common mysql-client libmysqlclient-dev mysql-server
	#接着我们进入 mysql console 建立新的数据库：
	mysql -u root -p
	CREATE DATABASE myapp_production CHARACTER SET utf8mb4;

## 安装 Nginx + Passenger 快方法：用套件安装
	apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
	 
	apt-get install -y apt-transport-https ca-certificates
	 
	# Add our APT repository
	sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger xenial main &gt; /etc/apt/sources.list.d/passenger.list'
	 
	apt-get update
	 
	# Install Passenger + Nginx
	apt-get install -y nginx-extras passenger

## Nginx启动和重开用法
	service nginx start
	service nginx stop
	service nginx restart

## capistrano配置

#### 我们在本地端‘Gemfile’中加上：
	gem 'capistrano-rails', :group =&gt; :development
	gem 'capistrano-passenger', :group =&gt; :development

#### 数据库用 MySQL 的话，记得再加上 gem "mysql2"
接着输入bundle install

#### 在capfile中加入Capistrano plugin
在你的Rails专案目录下执行：cap install

Enhance Capistrano with awesome collaboration and automation features? 请输入 no
这样就会产生几个文件，编辑Capfile在中段加入

	require 'capistrano/rails'
	require 'capistrano/passenger'
	require "capistrano/scm/git"
	install_plugin Capistrano::SCM::Git

####  编辑config/deploy.rb
请替换以下的application名称、git repo网址和deploy_to路径：
	`ssh-add` # 注意这是键盘左上角的「 `」不是单引号「 '」
	set :application, "myapp"
	set :repo_url, "git@github.com:caiwenhn2008/myapp.git"
	 
	set :deploy_to, "/home/deploy/myapp"
	 
	# Default value for :linked_files is []
	append :linked_files, 'config/database.yml', 'config/secrets.yml'
	 
	# Default value for linked_dirs is []
	append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system"
	 
	 
	set :keep_releases, 5
	 
	set :passenger_restart_with_touch, true
	# ....

#### 设置服务器的IP
其中的ssh-add可以参考SSH agent forwarding 的应用的说明。
编辑config/deploy/production.rb将example.com换成服务器的IP或网域，例如：

	server '45.32.59.122', user: 'deploy', roles: %w{app db web}, my_property: :my_value

#### 服务器上新建database.yml和secrets.yml
因为有了shared目录之后，我们还需要把配置放上去，请编辑：

- (远端) 新建 shared/config/database.yml
- (远端) 新建 shared/config/secrets.yml

database.yml和secrets.yml档案预先放在服务器的shared/config目录下，自动布署时会复盖过去。

执行rake secret产生的 key 放到远端服务器的shared/config/secrets.yml，范例如下(小心YAML格式的缩排规则，用两个空格)：

	production:
	  secret_key_base: xxxxxxx........

远端服务器设定好shared/config/database.yml，范例如下:
	production:
	  adapter: mysql2
	  encoding: utf8mb4
	  database: your_database_name
	  host: localhost
	  username: root
	  password: your_database_password

## capistrano 部署
本机执行cap production deploy:check，就会自动登入远端的服务器，在登入的帐号下新建releases和shared这两个目录，releases是每次布署的目录，shared目录则是不同布署目录之间会共享的。

到此终于可以部署了，执行cap production deploy就可以了。这会建立current这个目录用symbolic link指向releases目录下最新的版本。

## Nginx 配置

#### enable passenger
编辑 /etc/nginx/nginx.conf，打开以下一行：

	include /etc/nginx/passenger.conf;

在 /etc/nginx/nginx.conf最上方新增一行：

	env PATH;

#### add rails project configuration
新增 /etc/nginx/sites-enabled/myapp.conf

	server {
	  listen 80;
	  server_name 45.32.59.122; # 还没 domain 的话，先填 IP 位置
	 
	  root /home/deploy/myapp/current/public;
	  # 如果是自动化部署，位置在 root /home/deploy/your_project_name/current/public;
	 
	  passenger_enabled on;
	 
	# passenger min process number
	  passenger_min_instances 5;
	 
	  location ~ ^/assets/ {
	    expires 1y;
	    add_header Cache-Control public;
	    add_header ETag "";
	    break;
	   }
	}

####  restart nginx:

	service nginx restart

## Passenger 监控指令

	passenger-status
	passenger-memory-stats
