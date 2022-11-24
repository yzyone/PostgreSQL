# PostgREST 完整示例教程 #

**文章目录**

```
	前言
	准备环境
	初始化数据库
	配置 PostgREST
	前端交互演示
	结束
	jwt 认证系统原理
```

## 前言 ##

两年前写过一篇介绍 PostgREST 的文章，还建了个 QQ 交流群。不过一直还在学前端的东西，一直没用根据这套技术做什么东西。

这些天找出空来又研究了一下，还是觉得非常好玩非常便捷。所以写了一个完整的包括注册、登录鉴权，查询表和修改表的小示例，希望能给愿意尝试的同学一点帮助。

主要参考了官方文档和这篇文章

## 准备环境 ##

准备一个 CentOS 7.0 以上版本的 Linux 主机或者虚拟机，使用 ssh 连接。推荐一下 阿里云 1 折，活动优惠不是一般大, 100 就能用一年。

安装 docker，参照这篇文章。

安好后设置一下 docker 国内镜像地址，解决下载 docker images 太慢的问题。这里使用腾讯云加速。shell 下执行下面的命令：

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://mirror.ccs.tencentyun.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

本文很多地方的操作需要 root 权限，如不切换 root 账户的话，命令前面加 sudo 获取权限，后续不再提示。

## 初始化数据库 ##

首先在 docker 中下载和运行官方镜像，即安装好 postgeresql 数据库，密码设置为 mysecretpassword, 并启动在 5433 接口。同时为了插件文件的共享，在本机新建 dshare 目录并映射到 docker 内：

```
mkdir /dshare
docker run --name post -p 5433:5432 -v /dshare:/dshare \
    -e POSTGRES_PASSWORD=mysecretpassword -d postgres
```

然后下载 jwt 插件源码及我提供的初始脚本到共享目录，如还没有安装 Git 用 yum install git 安装一下。

cd /dshare
git clone https://github.com/Rackar/pgjwt
1
2
然后进入 docker 内命令行，更新和安装 make，之后安装 pgjwt 插件到数据库。

```
docker exec -it post bash
cd /dshare/pgjwt
apt-get update
apt-get install make
make install
psql -U postgres
create extension if not exists pgcrypto;
CREATE EXTENSION if not exists pgjwt;
```

然后可以导入我编辑好的 music_lover.sql 脚本文件进行建表、创建用户及赋权限、创建函数和触发器等。内有详细注释：

```
# docker内执行
psql -d postgres -U postgres -f /dshare/pgjwt/music_lover.sql

# 如主机内则执行
docker exec -it post psql -U postgres -f /dshare/pgjwt/music_lover.sql
```

导入成功后，数据库已经全部准备完成。可以开启 postgrest。

## 配置 PostgREST ##

下面定位到合适的目录下，下载 postgrest 。我是放到了 /root/postgrest 下。

```
mkdir /root/postgrest
cd /root/postgrest

# 官方地址下载
wget https://github.com/PostgREST/postgrest/releases/download/v6.0.2/postgrest-v6.0.2-centos7.tar.xz

# 或者我传的七牛云镜像，国内下载
wget http://q6o50zwkj.bkt.clouddn.com/postgrest-v6.0.2-centos7.tar.xz

# 原地解压
tar xfJ postgrest-v6.0.2-centos7.tar.xz
```

**还需要先安装一个依赖**，否则会报错 error:libpq.so.5 找不到：

    yum -y install postgresql-libs

好了，现在可以直接执行 ./postgrest 命令，出现用法示例，说明正常。Ctrl+C 关闭。

然后开始编写配置文件，执行下面的脚本：

```
cat <<EOF > pgrest.conf
db-uri = "postgres://app_role:change_this@localhost:5433/postgres"
db-schema = "public"
db-anon-role = "anon"
jwt-secret = "iDYR4j2Qp3QT05kpd9oIcF2WPWEWVrI3"
EOF
```

即在当前目录下创建出了 pgrest.conf 文件，其中各项都在 sql 脚本文件中有体现。如登录用户 app_role 及其密码 change_this，匿名用户 anon，以及在 login 函数中 sign 方法中用到的密钥 iDYR4j2Qp3QT05kpd9oIcF2WPWEWVrI3 ，脚本和配置文件修改时要对应。密钥记得换成自己的。

配置完毕，可以使用 ./postgrest pgrest.conf 执行。服务器会默认开启在 3000 端口。

## 前端交互演示 ##

新建 index.html 文件，cdn 方式使用 vue.js 和 axios 两个库，复制下面代码。记得修改 axios.defaults.baseURL 为自己的 ip 地址。注册、登录、使用登录后获取的 token 请求数据，提交数据打分。全套演示程序完成。

```
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <script src="https://cdn.jsdelivr.net/npm/vue"></script>
    <script src="https://unpkg.com/axios/dist/axios.min.js"></script>

    <title>demo</title>
  </head>
  <body style="-webkit-app-region: drag">
    <div id="app">
      {{message}}
      <div v-if="!token">
        邮箱： <input v-model="email" /> 密码：
        <input v-model="pass" type="password" />
        <button @click="signup">注册</button>
        <button @click="login">登录</button>
      </div>
      <div v-else>
        <select v-model="selectedArtist">
          <option v-for="option in optionsArtist" v-bind:value="option.value">
            {{ option.text }}
          </option>
        </select>
        <select v-model="selectedRatingType">
          <option
            v-for="option in optionsRatingType"
            v-bind:value="option.value"
          >
            {{ option.text }}
          </option>
        </select>
        <input type="number" v-model="rating" />
        <button @click="submit">打分</button>
      </div>
    </div>
    <script>
      axios.defaults.baseURL = "http://49.232.137.34:3000";
      var app = new Vue({
        el: "#app",
        data() {
          return {
            email: "",
            pass: "",
            token: "",
            selectedArtist: "",
            selectedRatingType: "",
            optionsArtist: [],
            optionsRatingType: [],
            rating: 0
          };
        },
        computed: {
          message() {
            return this.token ? "已登录" : "未登录";
          }
        },
        methods: {
          signup() {
            axios
              .post("/rpc/signup", { email: this.email, pass: this.pass })
              .then(info => {
                if (info.status == 200) {
                  this.login();
                } else {
                  alert("注册失败。");
                }
              });
          },
          login() {
            axios
              .post("/rpc/login", { email: this.email, pass: this.pass })
              .then(info => {
                console.log(info);
                if (info.status == 200) {
                  this.token = info.data[0].token;
                  this.getData();
                } else {
                  alert("登录失败。");
                }
              });
          },
          getData() {
            axios.defaults.headers.common["Authorization"] =
              "Bearer " + this.token;
            axios.get("/artists").then(info => {
              console.log(info);
              this.optionsArtist = info.data.map(artist => {
                return { text: artist.name, value: artist.name };
              });
              this.selectedArtist = this.optionsArtist[0].value;
            });
            axios.get("/rating_types").then(info => {
              console.log(info);
              this.optionsRatingType = info.data.map(type => {
                return { text: type.name, value: type.name };
              });
              this.selectedRatingType = this.optionsRatingType[0].value;
            });
          },
          submit() {
            let obj = {
              artist_name: this.selectedArtist,
              email: this.email,
              rating_type_name: this.selectedRatingType,
              rating: this.rating
            };
            axios.post("/ratings", obj).then(info => {
              console.log(info);
              if (info.status == 201) {
                alert("打分成功");
              }
            });
          }
        }
      });
    </script>
  </body>
</html>
```

## 结束 ##

使用 PostgREST 作为 API 服务器，真的不用写后端代码了。对个人做项目真的省去好多的心力。但是需要设计好表和关系，还需更加了解 POSTGRESQL 一些。

PostgREST 的联表查询和分页查询等也是非常方便，这里没有做进一步的展示。希望有过深入研究的同学能出一些质量更高的教程。

**正文结束。以下内容为供参考内容，可跳过。**

记录的其他命令：

```
# 安装完 pgjwt 插件后，可以创建自定义镜像
sudo docker commit -m "add post ext pgjwt" -a "Docker rackar" 14f90fd7b69b rackar/posrgres:v1

# 运行 sql 文件
docker exec -it post psql -U postgres -f /dshare/pgjwt/music_lover.sql

# 撤销权限。users 表需要匿名的 insert 权限，不需要 select 权限就可以返回 token。
revoke select on users from anon;

# 查看 docker 运行
docker ps -a

# 持续运行服务器
nohup ./postgrest pgrest.conf &

# linux 杀进程：
ps -ef | grep postgrest
kill -s 9 12931

```

**jwt 认证系统原理**

jwt 的 playload 中有一个关键词为 role。收到 token 并解析出 role 的时候，pgrest 会自动切换数据库账户名为 role 的值。当然这是在管理员已经 grant 后，允许切换才行。如GRANT auth_user TO authenticator;

如果没有 token 或者没有 role，会使用 PG 数据设置中的匿名账户。所以要确保匿名账户要访问的内容已设置好。

role 也很灵活，可以视为单个用户或者用户组。

如果把 role 视为单个角色，可以对单个角色执行行级安全策略。

同时 PG9.5 以上拥有行级访问控制，所以可以通过自定义策略来确定行级数据访问权限。如官网的一个聊天室示例，查询权限只能查 from or to 为本用户的行，插入和修改只能在 from 为本用户的行。

如果 role 为用户组，你还可以在 playoad 中加入 email 或者 username 来做用户区分。

这样通过`current_setting('request.jwt.claim.email', true)`来获取 email。

————————————————

版权声明：本文为CSDN博主「henjuewang」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/henjuewang/article/details/105021615
