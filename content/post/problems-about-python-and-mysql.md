+++
title = "problems about python and mysql"
date = "2015-01-19"
slug = "2015/01/19/problems-about-python-and-mysql"
Categories = []
+++

记录一下这几天项目上遇到的问题

## Python 2.7的编码问题
Python 2.x内部处理编码默认还是ascii，在处理utf8编码的数据时会报UnicodeEncodeError: ‘ascii’ codec can’t encode异常错误。这个问题之前就遇到过。

	import sys
	print sys.getdefaultencoding()  # 获取系统默认编码
	reload(sys)
	sys.setdefaultencoding('utf-8')  # 重新设置为utf-8

## MySQL 乱码解决
创建数据库  `CREATE DATABASE test CHARACTER SET utf8 COLLATE utf8_general_ci;`     
创建表    `CREATE TABLE test_table (........) ENGINE=InnoDB DEFAULT CHARSET=utf8;`

Python连接数据库时 `conn = MySQLdb.connect(host="localhost", user="root", passwd="root", db="db",  charset="utf8")`

## git
- 基于某个分支新建分支
	git branch muchunyu origin/dev  # 假设dev分支不在本地，存在于远程服务器
	git checkout muchunyu  # 切换到这个分支
或者可以直接：     
	git checkout -b muchunyu origin/dev
- 只clone某个分支到本地
	git clone -b <branch> <remote_repo>

## 常用数据库操作
	create database xxx; #创建数据库
	use xxx;   # 转到某个数据库
	create table xxx (....); # 创建表
	show tables;  # 显示数据库中的表
	describe xxxx; # 显示表结构
	drop xxx  # 可以是表名或者数据库名 表的结构 属性 索引都会被删掉
	truncate xxx  # 只用于表且只会清空表中数据，不会删除表结构等

## 参考
[在GitHub上管理项目](http://www.cnblogs.com/mengdd/p/3447464.html)
[Mysql乱码问题解决历程](http://www.cnblogs.com/fantiantian/p/3468454.html)
[Mysql中文乱码问题完美解决方案](http://www.2cto.com/database/201108/101151.html)
