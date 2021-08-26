# Centos 7 安装 PostgreSQL PGAdmin4 #

---

本文只讲PostgreSQL在CentOS 7.x 下的安装，其他系统请查看：https://www.postgresql.org/download

PostgreSQL 所用版本为：PostgreSQL 10

1.安装存储库


    yum install https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm
    
 
2.安装客户端``


    yum install postgresql10
    

3.安装服务端


    yum install postgresql10-server
    
    
4.验证是否安装成功


    rpm -aq| grep postgres
    
    
输出如下：

![](./images/668104-20171101143601920-445382560.png)


4.初始化数据库


    /usr/pgsql-10/bin/postgresql-10-setup initdb
    

5.启用开机自启动


    systemctl enable postgresql-10
    systemctl start postgresql-10
    

6.配置防火墙


    firewall-cmd --permanent --add-port=5432/tcp  
    firewall-cmd --permanent --add-port=80/tcp  
    firewall-cmd --reload  
    

7.修改用户密码


    su - postgres  //切换用户，执行后提示符会变为 '-bash-4.2$'
    psql -U postgres //登录数据库，执行后提示符变为 'postgres=#'
    ALTER USER postgres WITH PASSWORD 'postgres'  //设置postgres用户密码为postgres
    \q  //退出数据库
    

8.开启远程访问


    vim /var/lib/pgsql/10/data/postgresql.conf
    修改#listen_addresses = 'localhost'  为  listen_addresses='*'
    当然，此处‘*’也可以改为任何你想开放的服务器IP

9.信任远程连接

    vim /var/lib/pgsql/10/data/pg_hba.conf
    

修改如下内容，信任指定服务器连接


    # IPv4 local connections:
    hostallall  127.0.0.1/32  trust
    hostallall  192.168.157.1/32（需要连接的服务器IP，如果是所有则0.0.0.0/0）  trust
    

10.重启服务

    systemctl restart postgresql-10


11.使用DBMS软件连接

这里使用的是Navicat

![](./images/668104-20171101144149998-1959445258.png)


连接成功：

![](./images/668104-20171101144221670-1263091951.png)


12.安装 PGAdmin4:

    yum -y install epel-release
    
    yum install pgadmin4
    
    /usr/pgadmin4/bin/pgadmin4-web-setup.sh

    cd /usr/lib/python2.7/site-packages/pgadmin4-web/
    
    vim config_local.py
    #设置你希望的IP，0.0.0.0表示不受限制，如果你只希望局域网内访问，请使用适当的局域网ip
    DEFAULT_SERVER = '0.0.0.0'
    #设置你希望的端口
    DEFAULT_SERVER_PORT = 8080
     
    python /usr/lib/python2.7/site-packages/pgadmin4-web/pgAdmin4.py
    
转载于:https://www.cnblogs.com/littlewrong/p/9048248.html