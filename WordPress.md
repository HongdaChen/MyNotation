传统的使用wordpress搭建网站，意味着你需要搭建以下四个环境：

- php
- apache/linux
- mysql
- wordpress

这里面主要是php的搭建真心麻烦，各种依赖，版本不兼容，然后还有php跟mysql的插件，”我”是吃了它很大的苦，寻求让”我”快乐的方法，知道”我”发现了它。使用docker容器技术5分钟快速搭建wordpress，相信”我”，真的是五分钟。

需要Linux服务器一台。

## 1. 新建并启动 MySQL 容器

```
sudo docker container run  \
-d \
-p 3306:3306 \
-v "$PWD/wordpress/data":/var/lib/mysql  \
-e MYSQL_ROOT_PASSWORD=123456 \
-e MYSQL_DATABASE=wordpress \
--name wordpressdb \
mysql:5.7
```

- –env MYSQL_ROOT_PASSWORD=123456：向容器进程传入一个环境变量MYSQL_ROOT_PASSWORD，该变量会被用作 MySQL 的根密码。
- –env MYSQL_DATABASE=wordpress：向容器进程传入一个环境变量MYSQL_DATABASE，容器里面的 MySQL 会根据该变量创建一个同名数据库（本例是WordPress）。

## 2. 新建并启动 WordPress 容器

基于官方的 WordPress image，新建并启动 WordPress 容器。

```
sudo docker container run \
-d  \
-p 80:80 \
--name wordpress  \
-e WORDPRESS_DB_PASSWORD=123456 \
--link wordpressdb:mysql \
--volume "$PWD/wordpress/pages":/var/www/html \
wordpress
```

该命令执行完后，会在当前目录下生成一个wordpress目录。

```
-p 80:80：将容器的 80 端口映射到当前物理机的80端口。
--volume "$PWD/wordpress/pages":/var/www/html：将容器的`/var/www/html`目录映射到当前目录的wordpress/pages子目录，操作$pwd/wordpress/pages目录，相当于操作容器里面的/var/www/html目录了。
```

如果这个时候网页不可以访问到需要检查端口是否开放，以及安全组是否开放相应端口。

## 3. 修改WordPress配置文件

```
$ vim wordpress/wp-config.php
# 修改以下三个地方：
/** The name of the database for WordPress */
define( 'DB_NAME', getenv_docker('WORDPRESS_DB_NAME', 'wordpress') );
/** MySQL database username */
define( 'DB_USER', getenv_docker('WORDPRESS_DB_USER', 'root') );
/** MySQL database password */
define( 'DB_PASSWORD', getenv_docker('WORDPRESS_DB_PASSWORD', '123456') );
```

然后浏览器输入公网ip:80，即可进去wordpress安装界面，跟着向导，几分钟就完成了。

安装完成后，可以通过插件实现头像的自定义。

## 4. 博客数据迁移

总之分为两步：数据库脚本备份恢复和wordpress目录备份恢复。

### 【1】将mysql容器内的wordpress数据库导出为sql语句，这个时候可以有两种方法：

- 如果在开启mysql容器的时候并没有添加端口映射，这个时候必须进入到容器内部，采用命令的方式将数据库导出，命令如下：

```
# 执行如下命令导出数据库，并输入密码
mysqldump -u root -p eva -P 3306 > wordpress.sql
# 退出容器后，将导出的wordpress.sql文件从容器内部复制出来
sudo docker cp containerid/wordpress.sql:./
```

- 如果在开启mysql容器的时候添加端口映射，那么可以从外界使用数据库连接工具导出数据库。

在开启新的mysql容器时指定该目录，并从数据库脚本恢复数据库。

```
sudo docker container run  \
-d \
-p 3306:3306 \
-v "$PWD/wordpress/data":/var/lib/mysql  \
-e MYSQL_ROOT_PASSWORD=123456 \
-e MYSQL_DATABASE=wordpress \
--name wordpressdb \
mysql:5.7
```

### 【2】将wordpress/pages备份，并在开启新的wordpress容器时指定该目录。

```
sudo docker container run \
-d  \
-p 80:80 \
--name wordpress  \
-e WORDPRESS_DB_PASSWORD=123456 \
--link wordpressdb:mysql \
--volume "$PWD/wordpress/pages":/var/www/html \
wordpress
```
