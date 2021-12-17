PostgreSQL插件：Postgis编译安装

云平_Stephen 2018-11-20 14:20:31  2304  收藏 2
分类专栏： PostgreSQL 文章标签： postgresql postgis
版权

PostgreSQL
专栏收录该内容
29 篇文章3 订阅
订阅专栏


**文章目录**


概述
安装准备
安装Proj4
安装GEOS
安装GDAL
安装postgis
make时的报错
安装SFCGAL
测试
总结


**概述**

在迁移的过程中，发现用户数据库中还安装了postgis拓展，所以在测试时也需要安装一下此拓展
PostGIS是对象关系型数据库系统PostgreSQL的一个扩展，PostGIS提供如下空间信息服务功能:空间对象、空间索引、空间操作函数和空间操作符。

**安装准备**

postgis的安装需要一些依赖软件，这些软件我们试着先用yum源安装，但是好像不行，postgis在编译时会报错，所以最好这些软件都需要编译安装

**安装Proj4**

```
$ wget http://download.osgeo.org/proj/proj-4.9.3.tar.gz
$ tar -xf proj-4.9.3.tar.gz
$ cd proj-4.9.3
$ ./configure --prefix=/usr/proj
$ make
$ make install
# echo "/usr/proj/lib" > /etc/ld.so.conf.d/proj-4.9.3.conf
# ldconfig
```

**安装GEOS**

```
$ wget http://download.osgeo.org/geos/geos-3.6.1.tar.bz2
$ tar -jxf geos-3.6.1.tar.bz2
$ cd geos-3.6.1
$ ./configure --prefix=/usr/geos
$ make
$ make install
# echo "/usr/geos/lib" > /etc/ld.so.conf.d/geos-3.6.1.conf
# ldconfig
```

**安装GDAL**

```
$ wget http://download.osgeo.org/gdal/2.1.2/gdal-2.1.2.tar.gz
$ tar -xf gdal-2.1.2.tar.gz
$ cd gdal-2.1.2
$ ./configure --prefix=/usr/gdal
$ make
$ make install
# echo "/usr/gdal/lib" > /etc/ld.so.conf.d/gdal-2.1.2.conf
# ldconfig
```

**安装postgis**

```
$ wget http://download.osgeo.org/postgis/source/postgis-2.2.5.tar.gz
$ tar -xf postgis-2.2.5.tar.gz
$ cd postgis-2.2.5
$ ./configure --with-pgconfig=/usr/pgsql/bin/pg_config --with-geosconfig=/usr/geos/bin/geos-config --with-gdalconfig=/usr/gdal/bin/gdal-config  --with-projdir=/usr/proj --with-sfcgal=/usr/bin/sfcgal-config  --prefix=/usr/pgsql/share/contrib/postgis
checking build system type... x86_64-pc-linux-gnu
checking host system type... x86_64-pc-linux-gnu
checking how to print strings... printf
checking for gcc... gcc
checking whether the C compiler works... yes
checking for C compiler default output file name... a.out
checking for suffix of executables... 
checking whether we are cross compiling... no
checking for suffix of object files... o
checking whether we are using the GNU C compiler... yes
checking whether gcc accepts -g... yes
checking for gcc option to accept ISO C89... none needed
...........
config.status: executing po-directories commands

  PostGIS is now configured for x86_64-pc-linux-gnu

 -------------- Compiler Info ------------- 
  C compiler:           gcc -g -O2
  SQL preprocessor:     /usr/bin/cpp -traditional-cpp -w -P

 -------------- Dependencies -------------- 
  GEOS config:          /usr/geos/bin/geos-config
  GEOS version:         3.6.1
  GDAL config:          /usr/gdal/bin/gdal-config
  GDAL version:         2.1.2
  SFCGAL config:        /usr/bin/sfcgal-config
  SFCGAL version:       1.2.2
  PostgreSQL config:    /usr/pgsql/bin/pg_config
  PostgreSQL version:   PostgreSQL 10.3
  PROJ4 version:        49
  Libxml2 config:       /usr/bin/xml2-config
  Libxml2 version:      2.9.1
  JSON-C support:       yes
  protobuf-c support:   no
  PCRE support:         yes
  Perl:                 /usr/bin/perl

 --------------- Extensions --------------- 
  PostGIS Raster:       enabled
  PostGIS Topology:     enabled
  SFCGAL support:       enabled
  Address Standardizer support:       enabled

 -------- Documentation Generation -------- 
  xsltproc:             /usr/bin/xsltproc
  xsl style sheets:     
  dblatex:              
  convert:              
  mathml2.dtd:          http://www.w3.org/Math/DTD/mathml2/mathml2.dtd

configure: WARNING:  --------- GEOS VERSION WARNING ------------ 
configure: WARNING:   You are building against GEOS 3.6.1 
configure: WARNING:   To take advantage of all the features of 
configure: WARNING:   this PostGIS version requires GEOS 3.7.0 or higher which is not out yet.
configure: WARNING:   To take advantage of most of the features of this PostGIS
configure: WARNING:   we recommend GEOS 3.6 or higher
configure: WARNING:   You can download the latest versions from 
configure: WARNING:   http://trac.osgeo.org/geos 
configure: WARNING: 

```

**make时的报错**

我们在编译完成之后，执行make的时候报错

    lwgeom_sfcgal.h:28:34: fatal error: SFCGAL/capi/sfcgal_c.h: No such file or directory

这个错误的原因应该是SFCGAL这个软件没有安装，这个也需要编译安装

**安装SFCGAL**

下载地址：

    http://oslandia.github.io/SFCGAL/installation.html

此软件的编译需要cmake，所以需要先把cmake安装好

Compilation
The compilation process is based on CMake. On linux run:
    
    cmake . && make && sudo make install

在这个过程中cmake可能还会报错。gdal的类似错误
解决方法是：

    https://www.cgal.org/download.html

将 CGAL-4.7.tar.gz 下载并安装

cmake才会成功

测试

连接到pg数据库上，创建拓展

    proxydb=# create extension postgis;
    CREATE EXTENSION
    proxydb=# create extension postgis_topology ;
    CREATE EXTENSION
    proxydb=# 

总结
安装时的依赖过多，且层层依赖，需要一个个解决。

————————————————

版权声明：本文为CSDN博主「云平_Stephen」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/qq_43303221/article/details/84301234