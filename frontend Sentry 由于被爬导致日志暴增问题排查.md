# frontend Sentry 由于被爬导致日志暴增问题排查

## 系统表现

为 frontend 部署的 Sentry 服务一直运行稳定，但最近突然出现磁盘耗尽问题；

## 问题排查

首先确认磁盘消耗位置

```
root@sentry-frontend-prod:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            3.7G     0  3.7G   0% /dev
tmpfs           748M   73M  675M  10% /run
/dev/xvda1       20G   19G     0 100% /    -- 耗尽
tmpfs           3.7G  1.7M  3.7G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.7G     0  3.7G   0% /sys/fs/cgroup
/dev/xvdb        30G  431M   28G   2% /mnt
none             20G   19G     0 100% /var/lib/docker/aufs/mnt/bcf072126d60f7a1e0c37010d5d2d75af245588acf3329514c645956e9aa8bb3
shm              64M     0   64M   0% /var/lib/docker/containers/ddf0f283873268174c3b9a219fcf5352d176883af3545562a973d1dd9ff16b72/shm
tmpfs           748M     0  748M   0% /run/user/1001
none             20G   19G     0 100% /var/lib/docker/aufs/mnt/809a4c4add2206226106b49976a633badf1e7695b4cb41e988cbb44aed3aad4d
shm              64M     0   64M   0% /var/lib/docker/containers/1cae6524b913945c3ec03064fa69c769b571b5f3a8630a6f63f12a562cfa2211/shm
none             20G   19G     0 100% /var/lib/docker/aufs/mnt/f9c8bcb73ea75339306db5226efb798baaa3302b934388d7f026e28a6dd87928
shm              64M     0   64M   0% /var/lib/docker/containers/a7fd0da26221473071eecc7e21880d2d6d1711a673f72b672bff1bbf3c006e21/shm
none             20G   19G     0 100% /var/lib/docker/aufs/mnt/76c917402c1aee312ddb4858c43c8fcf934d86a874c8059315646eff9fd34ef5
shm              64M     0   64M   0% /var/lib/docker/containers/1aefee7212229654bcf1e986c7dff51c14fdbf5fe294140ec037158395ceb83f/shm
root@sentry-frontend-prod:~#
```

经确认，磁盘消耗主要是由于 sentry_worker 这个容器的日志暴涨导致；

```
root@sentry-frontend-prod:/var/lib/docker# du -sh *
3.3G	aufs
9.0G	containers
5.4M	image
64K	network
20K	plugins
4.0K	swarm
4.0K	tmp
4.0K	trust
164K	volumes
root@sentry-frontend-prod:/var/lib/docker#
root@sentry-frontend-prod:/var/lib/docker# cd containers/
root@sentry-frontend-prod:/var/lib/docker/containers# ll
total 24
drwx------  6 root root 4096 Apr 18 09:28 ./
drwx--x--x 11 root root 4096 Apr  6  2017 ../
drwx------  4 root root 4096 Apr 18 09:28 1aefee7212229654bcf1e986c7dff51c14fdbf5fe294140ec037158395ceb83f/
drwx------  4 root root 4096 May 28 03:07 1cae6524b913945c3ec03064fa69c769b571b5f3a8630a6f63f12a562cfa2211/
drwx------  4 root root 4096 Apr 18 09:28 a7fd0da26221473071eecc7e21880d2d6d1711a673f72b672bff1bbf3c006e21/
drwx------  4 root root 4096 Nov 13  2017 ddf0f283873268174c3b9a219fcf5352d176883af3545562a973d1dd9ff16b72/
root@sentry-frontend-prod:/var/lib/docker/containers#
root@sentry-frontend-prod:/var/lib/docker/containers# cd 1cae6524b913945c3ec03064fa69c769b571b5f3a8630a6f63f12a562cfa2211/
root@sentry-frontend-prod:/var/lib/docker/containers/1cae6524b913945c3ec03064fa69c769b571b5f3a8630a6f63f12a562cfa2211# ll -h
total 8.3G
drwx------ 4 root root 4.0K May 28 03:07 ./
drwx------ 6 root root 4.0K Apr 18 09:28 ../
-rw-r----- 1 root root 8.3G May 29 05:22 1cae6524b913945c3ec03064fa69c769b571b5f3a8630a6f63f12a562cfa2211-json.log
drwx------ 2 root root 4.0K Apr 18 09:28 checkpoints/
-rw-r--r-- 1 root root 3.9K Apr 18 09:28 config.v2.json
-rw-r--r-- 1 root root 1.3K Apr 18 09:28 hostconfig.json
-rw-r--r-- 1 root root   13 Apr 18 09:28 hostname
-rw-r--r-- 1 root root  174 Apr 18 09:28 hosts
-rw-r--r-- 1 root root  208 Apr 18 09:28 resolv.conf
-rw-r--r-- 1 root root   71 Apr 18 09:28 resolv.conf.hash
drwxrwxrwt 2 root root   40 Apr 18 09:28 shm/
```

简单查看，可以看到日志出现了很多奇怪日志

```
root@sentry-frontend-prod:/var/lib/docker/containers/1cae6524b913945c3ec03064fa69c769b571b5f3a8630a6f63f12a562cfa2211# tail -f 1cae6524b913945c3ec03064fa69c769b571b5f3a8630a6f63f12a562cfa2211-json.log
{"log":"172.17.0.1 - - [28/May/2018:06:32:09 +0000] \"GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket\u0026requestData=%5B%7B%22train_date%22%3A%2220180610%22%2C%22purpose_codes%22%3A%2200%22%2C%22from_station%22%3A%22SJP%22%2C%22to_station%22%3A%22BXP%22%2C%22station_train_code%22%3A%22%22%2C%22start_time_begin%22%3A%220000%22%2C%22start_time_end%22%3A%222400%22%2C%22train_headers%22%3A%22QB%23%22%2C%22train_flag%22%3A%22%22%2C%22seat_type%22%3A%22%22%2C%22seatBack_Type%22%3A%22%22%2C%22ticket_num%22%3A%22%22%2C%22dfpStr%22%3A%22%22%2C%22secret_str%22%3A%22%22%2C%22baseDTO%22%3A%7B%22check_code%22%3A%2222b849db2400d6daaea7e244764e8274%22%2C%22device_no%22%3A%220854303abe8e2ed9822aecbd190a0941%22%2C%22mobile_no%22%3A%22%22%2C%22os_type%22%3A%22a%22%2C%22time_str%22%3A%2220180528143209%22%2C%22user_name%22%3A%22%22%2C%22version_no%22%3A%223.0.1.01221000%22%7D%7D%5D\u0026ts=1527489129077\u0026sign=05dfd6fdaab191266169ba2e8f9833a0 HTTP/1.0\" 301 1166 \"-\" \"Go-http-client/1.1\"\n","stream":"stderr","time":"2018-05-28T06:32:09.498045921Z"}
{"log":"172.17.0.1 - - [28/May/2018:06:32:09 +0000] \"GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket\u0026requestData=%5B%7B%22train_date%22%3A%2220180531%22%2C%22purpose_codes%22%3A%2200%22%2C%22from_station%22%3A%22YZK%22%2C%22to_station%22%3A%22ZZQ%22%2C%22station_train_code%22%3A%22%22%2C%22start_time_begin%22%3A%220000%22%2C%22start_time_end%22%3A%222400%22%2C%22train_headers%22%3A%22QB%23%22%2C%22train_flag%22%3A%22%22%2C%22seat_type%22%3A%22%22%2C%22seatBack_Type%22%3A%22%22%2C%22ticket_num%22%3A%22%22%2C%22dfpStr%22%3A%22%22%2C%22secret_str%22%3A%22%22%2C%22baseDTO%22%3A%7B%22check_code%22%3A%22cf38b9bb193d44c10b13f7f17e149a94%22%2C%22device_no%22%3A%22WHxl5Mq48swDAHVf%2BxyZ%2Bfbm%22%2C%22mobile_no%22%3A%22%22%2C%22os_type%22%3A%22a%22%2C%22time_str%22%3A%2220180528143209%22%2C%22user_name%22%3A%22%22%2C%22version_no%22%3A%223.0.1.01221000%22%7D%7D%5D\u0026ts=1527489129228\u0026sign=3f79ad75ab7c06953d56065f35d1cb69 HTTP/1.0\" 301 1162 \"-\" \"Go-http-client/1.1\"\n","stream":"stderr","time":"2018-05-28T06:32:09.500997117Z"}
{"log":"172.17.0.1 - - [28/May/2018:06:32:09 +0000] \"GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket\u0026requestData=%5B%7B%22train_date%22%3A%2220180609%22%2C%22purpose_codes%22%3A%2200%22%2C%22from_station%22%3A%22RYV%22%2C%22to_station%22%3A%22DTV%22%2C%22station_train_code%22%3A%22%22%2C%22start_time_begin%22%3A%220000%22%2C%22start_time_end%22%3A%222400%22%2C%22train_headers%22%3A%22QB%23%22%2C%22train_flag%22%3A%22%22%2C%22seat_type%22%3A%22%22%2C%22seatBack_Type%22%3A%22%22%2C%22ticket_num%22%3A%22%22%2C%22dfpStr%22%3A%22%22%2C%22secret_str%22%3A%22%22%2C%22baseDTO%22%3A%7B%22check_code%22%3A%22a4aa9b7b3fb971b9e808ef02405299f4%22%2C%22device_no%22%3A%2294480b6fa8b5d9defe0ff3b956526359%22%2C%22mobile_no%22%3A%22%22%2C%22os_type%22%3A%22a%22%2C%22time_str%22%3A%2220180528143209%22%2C%22user_name%22%3A%22%22%2C%22version_no%22%3A%223.0.1.01221000%22%7D%7D%5D\u0026ts=1527489129701\u0026sign=2bb7b7aedba05f1eb1b73a798c1af754 HTTP/1.0\" 301 1166 \"-\" \"Go-http-client/1.1\"\n","stream":"stderr","time":"2018-05-28T06:32:09.865311581Z"}
{"log":"172.17.0.1 - - [28/May/2018:06:32:09 +0000] \"GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket\u0026requestData=%5B%7B%22train_date%22%3A%2220180528%22%2C%22purpose_codes%22%3A%2200%22%2C%22from_station%22%3A%22HKN%22%2C%22to_station%22%3A%22SNN%22%2C%22station_train_code%22%3A%22%22%2C%22start_time_begin%22%3A%220000%22%2C%22start_time_end%22%3A%222400%22%2C%22train_headers%22%3A%22QB%23%22%2C%22train_flag%22%3A%22%22%2C%22seat_type%22%3A%22%22%2C%22seatBack_Type%22%3A%22%22%2C%22ticket_num%22%3A%22%22%2C%22dfpStr%22%3A%22%22%2C%22secret_str%22%3A%22%22%2C%22baseDTO%22%3A%7B%22check_code%22%3A%22cb61f349012068fb404ca376f92247c6%22%2C%22device_no%22%3A%224d1be535746ee52042d6eaeccda537be%22%2C%22mobile_no%22%3A%22%22%2C%22os_type%22%3A%22a%22%2C%22time_str%22%3A%2220180528143209%22%2C%22user_name%22%3A%22%22%2C%22version_no%22%3A%223.0.1.01221000%22%7D%7D%5D\u0026ts=1527489129817\u0026sign=4a0c18d91412384ad62d3bc4e82b3518 HTTP/1.0\" 301 1166 \"-\" \"Go-http-client/1.1\"\n","stream":"stderr","time":"2018-05-28T06:32:09.994817825Z"}
{"log":"172.17.0.1 - - [28/May/2018:06:32:10 +0000] \"GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket\u0026requestData=%5B%7B%22train_date%22%3A%2220180625%22%2C%22purpose_codes%22%3A%2200%22%2C%22from_station%22%3A%22XVF%22%2C%22to_station%22%3A%22ZAF%22%2C%22station_train_code%22%3A%22%22%2C%22start_time_begin%22%3A%220000%22%2C%22start_time_end%22%3A%222400%22%2C%22train_headers%22%3A%22QB%23%22%2C%22train_flag%22%3A%22%22%2C%22seat_type%22%3A%22%22%2C%22seatBack_Type%22%3A%22%22%2C%22ticket_num%22%3A%22%22%2C%22dfpStr%22%3A%22%22%2C%22secret_str%22%3A%22%22%2C%22baseDTO%22%3A%7B%22check_code%22%3A%22bb6c4f4defb6d2a88695673febbd1afd%22%2C%22device_no%22%3A%2264557355bbe66e935e719935366c225d%22%2C%22mobile_no%22%3A%2213582079683%22%2C%22os_type%22%3A%22a%22%2C%22time_str%22%3A%2220180528143209%22%2C%22user_name%22%3A%22%22%2C%22version_no%22%3A%223.0.1.01221000%22%7D%7D%5D\u0026ts=1527489129977\u0026sign=4f7e40465fac46e318a4cc04c42eb615 HTTP/1.0\" 301 1177 \"-\" \"Go-http-client/1.1\"\n","stream":"stderr","time":"2018-05-28T06:32:10.222748087Z"}
...
```


消息解析举例

```
{"log":"172.17.0.1 - - [28/May/2018:06:32:10 +0000] \"GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket\u0026requestData=%5B%7B%22train_date%22%3A%2220180625%22%2C%22purpose_codes%22%3A%2200%22%2C%22from_station%22%3A%22XVF%22%2C%22to_station%22%3A%22ZAF%22%2C%22station_train_code%22%3A%22%22%2C%22start_time_begin%22%3A%220000%22%2C%22start_time_end%22%3A%222400%22%2C%22train_headers%22%3A%22QB%23%22%2C%22train_flag%22%3A%22%22%2C%22seat_type%22%3A%22%22%2C%22seatBack_Type%22%3A%22%22%2C%22ticket_num%22%3A%22%22%2C%22dfpStr%22%3A%22%22%2C%22secret_str%22%3A%22%22%2C%22baseDTO%22%3A%7B%22check_code%22%3A%22bb6c4f4defb6d2a88695673febbd1afd%22%2C%22device_no%22%3A%2264557355bbe66e935e719935366c225d%22%2C%22mobile_no%22%3A%2213582079683%22%2C%22os_type%22%3A%22a%22%2C%22time_str%22%3A%2220180528143209%22%2C%22user_name%22%3A%22%22%2C%22version_no%22%3A%223.0.1.01221000%22%7D%7D%5D\u0026ts=1527489129977\u0026sign=4f7e40465fac46e318a4cc04c42eb615 HTTP/1.0\" 301 1177 \"-\" \"Go-http-client/1.1\"\n","stream":"stderr","time":"2018-05-28T06:32:10.222748087Z"}
```

转换后有

```
{"log":"172.17.0.1 - - [28/May/2018:06:32:10  0000] \"GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket&requestData=[{"train_date":"20180625","purpose_codes":"00","from_station":"XVF","to_station":"ZAF","station_train_code":"","start_time_begin":"0000","start_time_end":"2400","train_headers":"QB#","train_flag":"","seat_type":"","seatBack_Type":"","ticket_num":"","dfpStr":"","secret_str":"","baseDTO":{"check_code":"bb6c4f4defb6d2a88695673febbd1afd","device_no":"64557355bbe66e935e719935366c225d","mobile_no":"13582079683","os_type":"a","time_str":"20180528143209","user_name":"","version_no":"3.0.1.01221000"}}]&ts=1527489129977&sign=4f7e40465fac46e318a4cc04c42eb615 HTTP/1.0\" 301 1177 \"-\" \"Go-http-client/1.1\"\n","stream":"stderr","time":"2018-05-28T06:32:10.222748087Z"}
```

经过确认，发现上述日志应该是由于某种爬虫应用造成的；

![网上随处可见的爬虫代码](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E7%BD%91%E4%B8%8A%E9%9A%8F%E5%A4%84%E8%AF%BE%E4%BB%B6%E7%9A%84%E7%88%AC%E8%99%AB%E4%BB%A3%E7%A0%81.png)

关键信息：

- 请求时间：28/May/2018:06:32:10
- 请求 URL：GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket
- 请求内容 requestData=[{"train_date":"20180625","purpose_codes":"00","from_station":"XVF","to_station":"ZAF","station_train_code":"","start_time_begin":"0000","start_time_end":"2400","train_headers":"QB#","train_flag":"","seat_type":"","seatBack_Type":"","ticket_num":"","dfpStr":"","secret_str":"","baseDTO":{"check_code":"bb6c4f4defb6d2a88695673febbd1afd","device_no":"64557355bbe66e935e719935366c225d","mobile_no":"13582079683","os_type":"a","time_str":"20180528143209","user_name":"","version_no":"3.0.1.01221000"}}]


时间线梳理：

- 4月18日，启动 sentry 服务
- 4月27日，首次发现被爬日志
- 5月1日，开始出现频繁、大量的被爬日志

![prom 监控 sentry 的磁盘曲线 - 1](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/prom%20%E7%9B%91%E6%8E%A7%20sentry%20%E7%9A%84%E7%A3%81%E7%9B%98%E6%9B%B2%E7%BA%BF%20-%201.png)

![prom 监控 sentry 的磁盘曲线 - 2](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/prom%20%E7%9B%91%E6%8E%A7%20sentry%20%E7%9A%84%E7%A3%81%E7%9B%98%E6%9B%B2%E7%BA%BF%20-%202.png)

相关日志截取

> 以下时间戳计算需要 +8

- Sentry 服务开始运行（2018-04-18T09:28:20）

```
{"log":"09:28:20 [INFO] sentry.plugins.github: apps-not-configured\n","stream":"stdout","time":"2018-04-18T09:28:20.72630072Z"}
```

- 第一次出现爬虫日志（2018-04-27T02:30:41）

```
{"log":"172.17.0.1 - - [27/Apr/2018:02:30:41 +0000] \"GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket\u0026requestData=%5B%7B%22train_date%22%3A%2220180507%22%2C%22purpose_codes%22%3A%2200%22%2C%22from_station%22%3A%22PIJ%22%2C%22to_station%22%3A%22POJ%22%2C%22station_train_code%22%3A%22%22%2C%22start_time_begin%22%3A%220000%22%2C%22start_time_end%22%3A%222400%22%2C%22train_headers%22%3A%22QB%23%22%2C%22train_flag%22%3A%22%22%2C%22seat_type%22%3A%22%22%2C%22seatBack_Type%22%3A%22%22%2C%22ticket_num%22%3A%22%22%2C%22dfpStr%22%3A%22%22%2C%22secret_str%22%3A%22%E5%90%8E%E5%8F%B0%E5%BC%80%E5%85%B3%E8%8E%B7%E5%8F%96%E5%A4%B1%E8%B4%A5%EF%BC%8C%E6%88%96%E9%85%8D%E7%BD%AE%E5%BC%80%E5%85%B3%E4%B8%BAfalse.null%22%2C%22baseDTO%22%3A%7B%22check_code%22%3A%22a1b6a51c291e24ce75a8403f864dd25f%22%2C%22device_no%22%3A%220d38f9f9e797d93a%22%2C%22mobile_no%22%3A%22%22%2C%22os_type%22%3A%22a%22%2C%22time_str%22%3A%2220180427103041%22%2C%22user_name%22%3A%22%22%2C%22version_no%22%3A%223.0.1.01221000%22%7D%7D%5D\u0026ts=1524796241867\u0026sign= HTTP/1.0\" 301 1263 \"-\" \"Go-http-client/1.1\"\n","stream":"stderr","time":"2018-04-27T02:30:41.969188684Z"}
```

- 开始出现大量爬虫日志（2018-05-01T12:07:48）

```
{"log":"172.17.0.1 - - [01/May/2018:12:07:48 +0000] \"GET /otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket\u0026requestData=%5B%7B%22train_date%22%3A%2220180511%22%2C%22purpose_codes%22%3A%2200%22%2C%22from_station%22%3A%22PIJ%22%2C%22to_station%22%3A%22POJ%22%2C%22station_train_code%22%3A%22%22%2C%22start_time_begin%22%3A%220000%22%2C%22start_time_end%22%3A%222400%22%2C%22train_headers%22%3A%22QB%23%22%2C%22train_flag%22%3A%22%22%2C%22seat_type%22%3A%22%22%2C%22seatBack_Type%22%3A%22%22%2C%22ticket_num%22%3A%22%22%2C%22dfpStr%22%3A%22%22%2C%22secret_str%22%3A%22%E5%90%8E%E5%8F%B0%E5%BC%80%E5%85%B3%E8%8E%B7%E5%8F%96%E5%A4%B1%E8%B4%A5%EF%BC%8C%E6%88%96%E9%85%8D%E7%BD%AE%E5%BC%80%E5%85%B3%E4%B8%BAfalse.null%22%2C%22baseDTO%22%3A%7B%22check_code%22%3A%229ac643f3a46e796a9e8037cfb7c85c12%22%2C%22device_no%22%3A%2225bff8496cc2f4d7%22%2C%22mobile_no%22%3A%22%22%2C%22os_type%22%3A%22a%22%2C%22time_str%22%3A%2220180501200747%22%2C%22user_name%22%3A%22%22%2C%22version_no%22%3A%223.0.1.01221000%22%7D%7D%5D\u0026ts=1525176467930\u0026sign= HTTP/1.0\" 301 1263 \"-\" \"Go-http-client/1.1\"\n","stream":"stderr","time":"2018-05-01T12:07:48.080518827Z"}
...
```

一句话来说：有“人”在不停的爬取火车票（上述信息为获取从许昌东（XVF）到郑州东（ZAF）的 20180625 火车票，电话号码为 13582079683）

> 全国车站代码详见：[这里](http://www.cnblogs.com/zengxiangzhan/p/3514917.html)

现在的问题是：**这个爬虫程序是公司内部员工所为还是外部人干的**？

> 背景知识：
>
> - sentry_web 监听 9000 端口（0.0.0.0:9000->9000/tcp）
> - sentry_worker 和 sentry_cron 使用 9000 端口在内部提供服务；

可以看到 TIME-WAIT 的数量较多（和历史值相比）；且几乎都是 9000 端口相关的；

```
root@sentry-frontend-prod:~# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
LISTEN 7
ESTAB 28
FIN-WAIT-1 1
FIN-WAIT-2 5
TIME-WAIT 1154
root@sentry-frontend-prod:~#
root@sentry-frontend-prod:~# netstat -natp | grep  "TIME_WAIT" | grep "9000" | wc -l
1078
root@sentry-frontend-prod:~#
```

抓包分析网络包特征：

```
root@sentry-frontend-prod:~# tcpdump -i any tcp port 9000 -s 0 -w crawler.pcap
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
^C9072 packets captured
12304 packets received by filter
0 packets dropped by kernel
root@sentry-frontend-prod:~#
```

抓包分析如下

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E6%8A%93%E5%8C%85%E5%88%86%E6%9E%90%20-%201.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E6%8A%93%E5%8C%85%E5%88%86%E6%9E%90%20-%202.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E6%8A%93%E5%8C%85%E5%88%86%E6%9E%90%20-%203.png)



截图中的完整 URL ：

```
http://mobile.12306.cn/otsmobile/app/mgs/mgw.htm?operationType=com.cars.otsmobile.queryLeftTicket&requestData=%5B%7B%22train_date%22%3A%2220180608%22%2C%22purpose_codes%22%3A%2200%22%2C%22from_station%22%3A%22XUN%22%2C%22to_station%22%3A%22ZZF%22%2C%22station_train_code%22%3A%22%22%2C%22start_time_begin%22%3A%220000%22%2C%22start_time_end%22%3A%222400%22%2C%22train_headers%22%3A%22QB%23%22%2C%22train_flag%22%3A%22%22%2C%22seat_type%22%3A%22%22%2C%22seatBack_Type%22%3A%22%22%2C%22ticket_num%22%3A%22%22%2C%22dfpStr%22%3A%22%22%2C%22secret_str%22%3A%22%22%2C%22baseDTO%22%3A%7B%22check_code%22%3A%22573a0e763991d33907a478cc5f73d988%22%2C%22device_no%22%3A%22WHxl5Mq48swDAHVf%2BxyZ%2Bfbm%22%2C%22mobile_no%22%3A%22%22%2C%22os_type%22%3A%22a%22%2C%22time_str%22%3A%2220180529142125%22%2C%22user_name%22%3A%22%22%2C%22version_no%22%3A%223.0.1.01221000%22%7D%7D%5D&ts=1527574885119&sign=31fdf6151135112f68ecf724a1871ff2
```

分析包中包含的 `X-Forwarded-For` 地址来源（仅整理了部分数据）：

- 111.231.12.246  -- 上海市 腾讯云
- 115.159.197.148 -- 上海市 腾讯云
- 115.159.56.38   -- 上海市 腾讯云
- 115.159.82.162  -- 上海市 腾讯云
- 115.159.192.32  -- 上海市 腾讯云
- 111.231.17.35   -- 上海市 腾讯云
- 123.207.215.239 -- 广东省广州市 腾讯云
- 123.207.187.137 -- 广东省广州市 腾讯云
- 47.96.159.245   -- 浙江省杭州市 阿里云
- 47.97.96.243    -- 浙江省杭州市 阿里云
- 47.96.176.180   -- 浙江省杭州市 阿里云
- 47.97.174.148   -- 浙江省杭州市 阿里云
- 47.96.159.245   -- 浙江省杭州市 阿里云
- 47.96.176.77    -- 浙江省杭州市 阿里云
- 118.89.162.100  -- 天津市 腾讯云华北数据中心
- 118.89.161.26   -- 天津市 腾讯云华北数据中心
- 118.89.176.229  -- 天津市 腾讯云华北数据中心
- 118.89.189.215  -- 天津市 腾讯云华北数据中心
- 121.236.239.219 -- 江苏省苏州市 电信

部分截图信息如下：

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/111.231.12.246.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/118.89.162.100.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/121.236.239.219.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/123.207.215.239.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/47.96.159.245.png)

到这里基本上可以确定不是内部员工所为，问题变成了：**为什么爬火车票的请求会发送到我们的 sentry 服务上**？

推理如下：我们 frontend Sentry 服务由于需要接收来自客户端的异常信息上报，因此开放了公网访问地址；因为考虑了安全因素，我们在 Sentry 服务前使用 Nginx 启用了 HTTPS 进行了防护，因此，**来自外部的请求一定是通过 Nginx 的 443 端口进入的**；由于一直以来认为基于 HTTPS 的交互是安全的，因此 Nginx 上并没有设置复杂的规则进行“防护”处理；所以，现实情况变成了（猜测）：爬虫程序运行在用户手机或云服务商的服务器上，**利用我们 frontend Sentry 的 HTTPS 连接进行了爬虫行为**；综合上面 `X-Forwarded-For` 的地址来源大多来自云服务商，故后者更加可疑；



目前 nginx 上针对 HTTPS 的设置如下

```
        server {
                listen 443 ssl;
                server_name prod.ft.llsops.com;

                proxy_set_header   Host                 $http_host;
                proxy_set_header   X-Forwarded-Proto    $scheme;
                proxy_set_header   X-Forwarded-For      $remote_addr;
                proxy_redirect     off;

                keepalive_timeout 0;

                proxy_read_timeout 5s;
                proxy_send_timeout 5s;
                send_timeout 5s;
                resolver_timeout 5s;
                client_body_timeout 5s;

                # buffer larger messages
                client_max_body_size 5m;
                client_body_buffer_size 100k;

                root /var/www/html;
                index index.html index.htm index.nginx-debian.html;

                location / {
                        proxy_pass http://localhost:9000;
                        #proxy_redirect     default;
                        add_header Strict-Transport-Security "max-age=31536000";
                }
        }
```



## 问题解决

问题解决分如下几个方面：

### 解决磁盘满问题

```
root@sentry-frontend-prod:/var/lib/docker/containers/1cae6524b913945c3ec03064fa69c769b571b5f3a8630a6f63f12a562cfa2211# truncate -s 3G 1cae6524b913945c3ec03064fa69c769b571b5f3a8630a6f63f12a562cfa2211-json.log
```

效果为直接将该日志文件从 8.3G 减到 3.0G（鉴于该问题中日志本身价值不大，所以直接进行了 shrink 操作）


### Sentry 日志配置调整

全局配置：默认位置 `/etc/docker/daemon.json` ，通过 dockerd 启动参数 `--config-file` 指定

若想将 `json-file` driver 作为默认 logging driver ，则需要在 daemon.json 文件中对 log-driver 和 log-opt 进行设置，该文件在 Linux 上位于 `/etc/docker/` 下；

```
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m"
  }
}
```

为指定 container 设置 logging driver 配置

```
docker run --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 alpine echo hello world
```

> 参考：[JSON File logging driver](https://docs.docker.com/config/containers/logging/json-file/)

docker-compose 配置文件

```
nginx:
  image: nginx:1.12.1
  restart: always
  logging:
    driver: "json-file"
    options:
      max-size: "5g"
```

调整配置（之后要重启容器）


```
root@sentry-frontend-prod:/opt/docker/deploy# ll
total 24
drwxr-xr-x 5 deployer users 4096 May 29 12:03 ./
drwxr-xr-x 3 root     root  4096 Apr 13 05:45 ../
drwxr-xr-x 2 deployer users 4096 May 29 12:04 sentry_cron/
drwxr-xr-x 2 deployer users 4096 May 29 12:04 sentry_web/
drwxr-xr-x 2 deployer users 4096 May 29 12:04 sentry_worker/
-rw-r--r-- 1 deployer users   96 Nov 13  2017 start_email.sh
root@sentry-frontend-prod:/opt/docker/deploy#
root@sentry-frontend-prod:/opt/docker/deploy# cat sentry_cron/start_8.22.sh
#!/bin/bash

docker run -d \
     \
    --log-driver json-file \   -- 增加
    --log-opt max-size=2g \    -- 增加
    --restart=on-failure:3 \
    -e "DEPLOY_ENV=production" \
    -e "HOST_IP=ens3" \
    -e "HOST_NAME=sentry-frontend-prod" \
    -e "SENTRY_DB_NAME=sentryfrontend" \
    -e "SENTRY_DB_USER=root" \
    -e "SENTRY_DB_PASSWORD=&tgyg>2jP4Z97(HH3(3Jx]3{7&s(i*bPr29QAqx^" \
    -e "SENTRY_SECRET_KEY=y1=x)zfo+oclm7nn%z)+x%#l6pd(f03bet-ghvif*al&)o6umg"  \
    -e "SENTRY_REDIS_HOST=sentry-redis.qh1egb.0001.cnn1.cache.amazonaws.com.cn"  \
    -e "SENTRY_REDIS_DB=1"  \
    -e "SENTRY_POSTGRES_HOST=sentry-prod.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn"  \
    -e "SENTRY_EMAIL_HOST=172.31.12.235"  \
    -e "SENTRY_SERVER_EMAIL=noreplay@prod-ft.llsops.com"  \
    -e "SENTRY_USE_SSL=1"  \
    --name engzo-sentry_cron-8.22 \
    prod-reg.llsops.com/platform-rls/sentry:8.22 \
    run cron
root@sentry-frontend-prod:/opt/docker/deploy#
root@sentry-frontend-prod:/opt/docker/deploy# cat sentry_web/start_8.22.sh
#!/bin/bash

docker run -d \
     \
    --log-driver json-file \    -- 增加
    --log-opt max-size=2g \     -- 增加
    --restart=on-failure:3 \
    -e "DEPLOY_ENV=production" \
    -e "HOST_IP=ens3" \
    -e "HOST_NAME=sentry-frontend-prod" \
    -e "SENTRY_DB_NAME=sentryfrontend" \
    -e "SENTRY_DB_USER=root" \
    -e "SENTRY_DB_PASSWORD=&tgyg>2jP4Z97(HH3(3Jx]3{7&s(i*bPr29QAqx^" \
    -e "SENTRY_SECRET_KEY=y1=x)zfo+oclm7nn%z)+x%#l6pd(f03bet-ghvif*al&)o6umg"  \
    -e "SENTRY_REDIS_HOST=sentry-redis.qh1egb.0001.cnn1.cache.amazonaws.com.cn"  \
    -e "SENTRY_REDIS_DB=1"  \
    -e "SENTRY_POSTGRES_HOST=sentry-prod.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn"  \
    -e "SENTRY_EMAIL_HOST=172.31.12.235"  \
    -e "SENTRY_SERVER_EMAIL=noreplay@prod-ft.llsops.com"  \
    -e "SENTRY_USE_SSL=1"  \
    -p 9000:9000  \
    -v /opt/data/sentry/:/var/lib/sentry/files  \
    --name engzo-sentry_web-8.22 \
    prod-reg.llsops.com/platform-rls/sentry:8.22 \
    run web
root@sentry-frontend-prod:/opt/docker/deploy#
root@sentry-frontend-prod:/opt/docker/deploy# cat sentry_worker/start_8.22.sh
#!/bin/bash

docker run -d \
     \
    --log-driver json-file \    -- 增加
    --log-opt max-size=5g \     -- 增加
    --restart=on-failure:3 \
    -e "DEPLOY_ENV=production" \
    -e "HOST_IP=ens3" \
    -e "HOST_NAME=sentry-frontend-prod" \
    -e "SENTRY_DB_NAME=sentryfrontend" \
    -e "SENTRY_DB_USER=root" \
    -e "SENTRY_DB_PASSWORD=&tgyg>2jP4Z97(HH3(3Jx]3{7&s(i*bPr29QAqx^" \
    -e "SENTRY_SECRET_KEY=y1=x)zfo+oclm7nn%z)+x%#l6pd(f03bet-ghvif*al&)o6umg"  \
    -e "SENTRY_REDIS_HOST=sentry-redis.qh1egb.0001.cnn1.cache.amazonaws.com.cn"  \
    -e "SENTRY_REDIS_DB=1"  \
    -e "SENTRY_POSTGRES_HOST=sentry-prod.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn"  \
    -e "SENTRY_EMAIL_HOST=172.31.12.235"  \
    -e "SENTRY_SERVER_EMAIL=noreplay@prod-ft.llsops.com"  \
    -e "SENTRY_USE_SSL=1"  \
    -v /opt/data/sentry/:/var/lib/sentry/files  \
    --name engzo-sentry_worker-8.22 \
    prod-reg.llsops.com/platform-rls/sentry:8.22 \
    run worker
root@sentry-frontend-prod:/opt/docker/deploy#
```


查看调整后的配置

```
root@sentry-frontend-prod:/opt/docker/deploy# docker ps
CONTAINER ID        IMAGE                                          COMMAND                  CREATED             STATUS              PORTS                    NAMES
46bd5c3e105f        prod-reg.llsops.com/platform-rls/sentry:8.22   "/entrypoint.sh ru..."   3 minutes ago       Up 3 minutes        9000/tcp                 engzo-sentry_worker-8.22
3f9504c483ad        prod-reg.llsops.com/platform-rls/sentry:8.22   "/entrypoint.sh ru..."   3 minutes ago       Up 3 minutes        9000/tcp                 engzo-sentry_cron-8.22
367b4af2723b        prod-reg.llsops.com/platform-rls/sentry:8.22   "/entrypoint.sh ru..."   3 minutes ago       Up 3 minutes        0.0.0.0:9000->9000/tcp   engzo-sentry_web-8.22
ddf0f2838732        tianon/exim4                                   "entrypoint.sh tin..."   6 months ago        Up 6 months         0.0.0.0:25->25/tcp       email
root@sentry-frontend-prod:/opt/docker/deploy#
root@sentry-frontend-prod:/opt/docker/deploy# docker inspect --format='{{.HostConfig.LogConfig}}' `docker ps -q`
{json-file map[max-size:5g]}
{json-file map[max-size:2g]}
{json-file map[max-size:2g]}
{json-file map[]}
root@sentry-frontend-prod:/opt/docker/deploy#
```


### 基于 Nginx 进行非法访问过滤

调整 nginx 配置，进行非法 URL 过滤或限制；

### 系统层面安全防护

- 反爬机制
- 证书安全性
- 云服务安全问题
- and so on


## 结论

- 应该不是恶意攻击我们的系统，不过确实对系统会产生一定的影响（占用系统资源）；
- 既然爬虫能够利用 HTTPS 连接将爬取火车票 HTTP 请求发送到我们的 Sentry 系统，那么有利用相信，如果真的发生恶意攻击，将会造成更大的损失；
- 当前关于爬虫是如何利用 HTTPS 连接进入我们系统的问题没有完全想明白：不确定是因为 Sentry 中的 Client Keys (DSN) 被利用导致，还是因为其他什么原因；

