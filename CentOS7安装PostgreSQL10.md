
# CentOS7.6安装postgresql10

---

记录一下centos7.6下安装postgresql10的过程

### 安装

1 更新源

```
yum install https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm -y
```

2 安装postgresql

```
yum install postgresql10-contrib postgresql10-server -y
```

3 初始化数据库

```
/usr/pgsql-10/bin/postgresql-10-setup initdb
```

4 启动服务

```
systemctl start postgresql-10
```

5 设置密码
postgresql 会默认添加一个用户postgres，我们需要给他设置一下密码

```
su - postgres
psql
ALTER USER postgres WITH PASSWORD 'yourpassword';
```

到这里已经可以使用了，但是为了更方便的使用，我们还可以进行其他设置。

### 远程登录

postgresql默认不支持远程登录，我们需要修改配置文件pg_hba.conf

```
vim /var/lib/pgsql/10/data/pg_hba.conf
```

修改后

```
host    all             all             0.0.0.0/0            md5
```

然后修改另一个配置文件postgresql.conf

```
vim /var/lib/pgsql/10/data/postgresql.conf
```

修改后

```
listen_addresses = '*'
port = 5432
max_connections = 100
```

重启postgresql

```
systemctl restart postgresql-10
```

之后就可以用pgadmin远程登录了。

### 开机自启动

设置开机启动

```
systemctl enable postgresql-10.service
```

### 备份与恢复

postgresql的备份命令为

```
pg_dump -h ipaddress -p port -U user dbname>dumpfile
```

例如，将本地数据库postgres备份到/var/www/backup.sql文件中

```
pg_dump -h 127.0.0.1 -p 5432 -U postgres postgres>/var/www/backup.sql
```

postgresql的恢复命令为

```
psql -h ipaddress -p port -U user dbname<dumpfile
```

例如，从/var/www/backup.sql恢复数据库到本地postgres中

```
psql -h 127.0.0.1 -p 5432 -U postgres postgres</var/www/backup.sql
```

值得一提的是，恢复命令不会创建数据库，所以需要自行创建一个新的数据库（在上例中，新数据库名仍是postgres）

### 定时备份

由备份命令我们可以创建一个脚本来自动化备份数据。

我在/var/www/下新建一个文件dailybackup.sh，脚本内容大致如下

```
#!/bin/bash

cur_day=$(date '+%Y%m%d')

echo "start backup postgresql..."

/usr/pgsql-10/bin/pg_dump -h 127.0.0.1 -p 5432 -U postgres postgres>/var/www/backup_$cur_day.sql

echo "finish backup."
```

记得顺便给该文件授予可执行权限

```
chmod +x /var/www/dailybackup.sh
```

再新建一个定时任务，执行命令

```
crontab -e
```

输入如下内容，意思是每天凌晨1点执行/var/www/dailybackup.sh

```
0 1 * * * /var/www/dailybackup.sh
```

保存退出，由此自动化备份数据库完成。

