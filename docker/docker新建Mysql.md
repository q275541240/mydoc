# Docker创建Mysql
##下载镜像
docker pull mysql
##启动镜像
docker run -d -it --name mysql -e MYSQL_ROOT_PASSWORD=123456 -p 3306:3306 -v mysql-vol:/var/lib/mysql mysql

-d:后台启动
-e:配置MYSQL默认密码
-p:配置端口映射
-v:配置数据与宿主机的挂载，在/var/lib/docker/volums/下能看到具体的挂载文件信息
##修改认证模式
进入MySQL容器

docker exec -it mysql /bin/bash

使用MySQL命令行工具连接MySQL

mysql -h localhost -u root -p

连接成功后，接下来我们就可以使用SQL语句来修改“root”账户的认证模式了：

ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';

最后连接mysql
jdbc:mysql://192.168.1.110:3306?useUnicode=true&characterEncoding=UTF-8&useSSL=false&autoReconnect=true&failOverReadOnly=false
