CentOS7上安装Postgresql10热备份读写分离

---

1、更新CentOS安装源

	yum install https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm -y

2、安装postgresql-10数据库

	yum install postgresql10-contrib postgresql10-server -y

3、初始化一个新的数据库，已经配置好的数据库不用初始化

	/usr/pgsql-10/bin/initdb /var/lib/pgsql/10/pg_root -E UTF8 --locale=C -U postgres -W

4、启动服务

	systemctl start postgresql-10

5、设置密码为123456

    su - postgres
    psql
    ALTER USER postgres WITH PASSWORD '123456';

---

对于已经安装并配置好的数据库，前5步可以忽略。

下面是热备配置要点，主要以用户postgres来操作：

- 主库172.21.16.2
- 备库172.16.0.59

6、主库上修改配置文件postgresql.conf

设置一下6个参数：

	wal_level = replica                     # minimal, replica, or logical
	archive_mode = on               # enables archiving; off, on, or always
	archive_command = '/bin/date'           # command to use to archive a logfile segment
	max_wal_senders = 10            # max number of walsender processes
	wal_keep_segments = 512         # in logfile segments, 16MB each; 0 disables
	hot_standby = on                        # "off" disallows queries during recovery

7、主库上配置的pg_hba.conf，添加以下内容

	host    replication     repuser         172.21.16.2/32            md5
	host    replication     repuser         172.16.0.59/32            md5

8、主库上新建数据库用户repuser

	$ psql
	psql (10.14)
	Type "help" for help.
	
	postgres=# CREATE USER repuser
	postgres=# REPLICATION
	postgres=# LOGIN
	postgres=# CONNECTION LIMIT 5
	postgres=# ENCRYPTED PASSWORD '123456';

或者执行SQL:

	CREATE USER repuser REPLICATION LOGIN CONNECTION LIMIT 5 ENCRYPTED PASSWORD '123456';


9、停止备库服务，备份并删除备库目录$PGDATA，在备库上用工具做一个基准备份。

	systemctl stop postgresql-10
	mv /var/lib/pgsql/10/data /var/lib/pgsql/10/data_bak
	rm -rf /var/lib/pgsql/10/data
	/usr/pgsql-10/bin/pg_basebackup -D /var/lib/pgsql/10/data -Fp -Xs -v -P -h 172.21.16.2 -p 5432 -U repuser

10、备库上新建配置文件recovery.conf

	cp /usr/pgsql-10/share/recovery.conf.sample /var/lib/pgsql/10/data/recovery.conf

配置以下参数

	recovery_target_timeline = 'latest'
	standby_mode = on
	primary_conninfo = 'host=172.21.16.2 port=5432 user=repuser password=123456'

确认目录data权限

	chown postgres:postgres /var/lib/pgsql/10/data -R

11、重新启动备库服务

	systemctl restart postgresql-10

12、热备成功，备库提升为主库

	/usr/pgsql-10/bin/pg_ctl promote -D /var/lib/pgsql/10/data