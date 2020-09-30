---
title: 中华杯 - Linux模块样题详解
tags: 中华杯,搭建,比赛
grammar_cjkRuby: true
---

# 任务配置清单
| 虚拟主机名称| 服务角色| 用户名和密码| 域名信息| IP 主机名|
|:-:|:-:|:-:|:-:|:-:|
| Debian       | DNS、WWW、Mail等服务器 | root/123456<br>test/123456 | w1.skills2021.sh<br>w2.skills2021.sh<br>mail.skills2021.sh |  192.168.101.100/24<br>linux.skills2021.sh |

# 任务配置描述
## 根据配置清单配置主机名与IP地址
```bash?linenums
root@debian:~# # 查看端口名称
root@debian:~# ip ad
	...
	3: ens37: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
		link/ether 00:0c:29:8c:6d:5a brd ff:ff:ff:ff:ff:ff
root@debian:~# # 直接修改配置文件来配置IP
root@debian:~# vim /etc/network/interfaces
	auto ens37
	iface ens37 inet static
	address 192.168.101.100
	netmask 255.255.255.0
root@debian:~# # 修改Hostname
root@debian:~# hostnamectl set-hostname linux.skills2021.sh
root@debian:~# 
```
## DNS (bind9)
1. 安装 bind9 服务
	```bash?linenums
	root@linux:~# # 安装Bind9和调试工具
	root@linux:~# apt install -y bind9 bind9utils
		...
		Done
	root@linux:~# # 进入Bind9配置文件夹
	root@linux:~# cd /etc/bind
	root@linux:/etc/bind#
	```
2. 为两个网站 w1.skills2021.sh、w2.skills2021.sh 以及 mail.skills2021.sh 提供解析，解析 IP 地址为 DEBIAN 的 IP 地址
	```bash?linenums
	root@linux:/etc/bind# # 在根配置中添加自定区域配置，里面的文件名叫啥都行，前后一样就行
	root@linux:/etc/bind# vim named.conf
		...
		include "/etc/bind/zone.skills2021";
	root@linux:/etc/bind# # 创建区域配置文件
	root@linux:/etc/bind# touch zone.skills2021
	root@linux:/etc/bind# # 编辑区域配置文件
	root@linux:/etc/bind# vim zone.skills2021
		zone "skills2021.sh." {
			type master;
			file "/etc/bind/db.skills2021";
		};
	root@linux:/etc/bind# # 检查配置文件是否正确,没有返回就说明没有错误
	root@linux:/etc/bind# named-checkconf
	root@linux:/etc/bind# # 创建记录文件
	root@linux:/etc/bind# cp db.local db.skills2021
	root@linux:/etc/bind# # 编辑记录文件
	root@linux:/etc/bind# vim db.skills2021
		;
		; BIND data file for local loopback interface
		;
		; SOA后第一个域名必须写NS记录的地址
		; SOA后第二个域名是管理员邮箱，这个写啥都行
		; 下面的一堆参数复制过来之后不用修改，默认即可
		$TTL    604800
		@       IN      SOA     skills2021.sh.  root.skills2021.sh. (
									  2         ; Serial
								 604800         ; Refresh
								  86400         ; Retry
								2419200         ; Expire
								 604800 )       ; Negative Cache TTL
		;
		@       IN      NS      skills2021.sh.
		@       IN      A       192.168.101.100
		w1      IN      A       192.168.101.100
		w2      IN      A       192.168.101.100
		mail    IN      A       192.168.101.100
	root@linux:/etc/bind# # 检查记录文件是否正确
	root@linux:/etc/bind# # named-checkzone <域名> <记录文件位置>
	root@linux:/etc/bind# named-checkzone skills2021.sh. /etc/bind/db.skills2021
		zone skills2021.sh/IN: loaded serial 2
		OK
	root@linux:/etc/bind# 
	```
3. 为 skills2021.sh 添加 MX 记录，优先级为 20，对应 IP 地址为 DEBIAN 的 IP 地址
	```bash?linenums
	root@linux:/etc/bind# # 编辑记录文件
	root@linux:/etc/bind# vim db.skills2021
		...
		@       IN      MX 20   mail
	root@linux:/etc/bind# # 检查记录文件是否正确
	root@linux:/etc/bind# # named-checkzone <域名> <记录文件位置>
	root@linux:/etc/bind# named-checkzone skills2021.sh. /etc/bind/db.skills2021
		zone skills2021.sh/IN: loaded serial 2
		OK
	root@linux:/etc/bind# 
	```
4. 将域名 server150.skills2021.sh 到 server200.skills2021.sh 对应解析到 192.168.101.150 到 192.168.101.200
	```bash?linenums
	root@linux:/etc/bind# # 在比赛过程中时间很短，所以我这里写了个python脚本，如果会别的语言可以用别的
	root@linux:/etc/bind# vim test.py
			#!/usr/bin/python3
			file=open("db.skills2021","a")
			for i in range(150,201):
				file.write("server%s    IN  A   192.168.101.%s\n"%(str(i),str(i)))
			file.close()
	root@linux:/etc/bind# python3 test.py
	root@linux:/etc/bind# cat db.skills2021
		...
		server150    IN  A   192.168.101.150
		server151    IN  A   192.168.101.151
		server152    IN  A   192.168.101.152
		server153    IN  A   192.168.101.153
		...
	root@linux:/etc/bind# # 检查记录文件是否正确
	root@linux:/etc/bind# # named-checkzone <域名> <记录文件位置>
	root@linux:/etc/bind# named-checkzone skills2021.sh. /etc/bind/db.skills2021
		zone skills2021.sh/IN: loaded serial 2
		OK
	root@linux:/etc/bind# 
	```
5. 将 DNS 解析服务（bind9）设置为开机自启动
	```bash?linenums
	root@linux:/etc/bind# update-rc.d bind9 enable
	root@linux:/etc/bind#
	```

## Apache2
1. 安装 Apache2 服务
	```bash?linenums
	root@linux:~# # 安装Apache2及网页测试工具
	root@linux:~# apt install -y apache2 curl
		...
		Done
	root@linux:~# 
	```
2. 设置 apache2 提供 PHP 脚本的解析服务
	```bash?linenums
	root@linux:~# # 安装Apache2的PHP模块
	root@linux:~# apt install -y libapache2-mod-php
		...
		Done
	root@linux:~# 
	```
3. 在 apache2 的默认网站文件目录中创建一个test.php页面用于测试php模块是否正确安装(内容为phpinfo)
	```bash?linenums
	root@linux:~# # 转到默认网页存放处
	root@linux:~# cd /var/www/html/
	root@linux:/var/www/html# # 创建并编辑test.php文件
	root@linux:/var/www/html# vim test.php
		<?php phpinfo();?>
	root@linux:/var/www/html# # 使用curl测试php
	root@linux:/var/www/html# curl localhost/test.php
		......
		一堆
		......
	root@linux:/var/www/html# # 测试成功
	```
4. 添加两个虚拟站点，当使用 w1.skills2021.sh、w2.skills2021.sh 访问时分别显示为“This is w1 website”和“This is w2 website”
	```bash?linenums
	root@linux:/var/www/html# # 转到配置文件文件夹
	root@linux:/var/www/html# cd /etc/apache2/sites-available/
	root@linux:/etc/apache2/sites-available# # 从模板(000-default.conf)创建出新配置
	root@linux:/etc/apache2/sites-available# cp 000-default.conf w1.conf
	root@linux:/etc/apache2/sites-available# cp 000-default.conf w2.conf
	root@linux:/etc/apache2/sites-available# # 修改w1站点
	root@linux:/etc/apache2/sites-available# vim w1.conf
		# 将虚拟主机名改成要求的
		<VirtualHost w1.skills2021.sh:80>
		...
			# 将网站跟目录改成别的地方
			DocumentRoot /var/www/w1
		...
		# 别的就没啥好改的了
	root@linux:/etc/apache2/sites-available# # 修改w2站点
	root@linux:/etc/apache2/sites-available# vim w2.conf
		# 将虚拟主机名改成要求的
		<VirtualHost w2.skills2021.sh:80>
		...
			# 将网站跟目录改成别的地方
			DocumentRoot /var/www/w2
		...
		# 别的就没啥好改的了
	root@linux:/etc/apache2/sites-available# # 转到网站存放处
	root@linux:/etc/apache2/sites-available# cd /var/www/
	root@linux:/var/www# # 创建对应文件夹
	root@linux:/var/www# mkdir w1 w2
	root@linux:/var/www# # 创建w1的index文件并写入内容
	root@linux:/var/www# echo "This is w1 website" > w1/index.html
	root@linux:/var/www# # 创建w2的index文件并写入内容
	root@linux:/var/www# echo "This is w2 website" > index.html
	root@linux:/var/www# # 修改DNS,这个只是临时修改，可能重启或某段时间后会自动恢复
	root@linux:/var/www# # 每当访问不了都可以重新设置下nameserver
	root@linux:/var/www# # 永久修改比较耗时间，懒得做了，毕竟比赛环境，如果不放心可以写个死循环
	root@linux:/var/www# echo "nameserver 192.168.101.100" > /etc/resolv.conf
	root@linux:/var/www# # 启动w1和w2站点
	root@linux:/var/www# a2ensite w1 w2
		Enabling site w1.
		Enabling site w2.
		To activate the new configuration, you need to run:
		  systemctl reload apache2
	root@linux:/var/www# # 重新加载Apache2服务
	root@linux:/var/www# systemctl restart apache2
	root@linux:/var/www# # 测试w1的访问与内容
	root@linux:/var/www# curl w1.skills2021.sh
		This is w1 website
	root@linux:/var/www# # 测试w2的访问与内容
	root@linux:/var/www# curl w2.skills2021.sh
		This is w2 website
	```
5. 将 apache2 服务设置为开机自启动
	```bash?linenums
	root@linux:/var/www# update-rc.d apache2 enable
	root@linux:/var/www#
	```
