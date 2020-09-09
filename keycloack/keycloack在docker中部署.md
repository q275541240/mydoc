#KeyCloack在docker容器中部署
1.先安装mysql
```
docker pull ncc0706/alpine-mysql

docker run -d --name mysql -p 3306:3306 -e MYSQL_DATABASE=cms -e MYSQL_USER=cms -e MYSQL_PASSWORD=cms2018 -e MYSQL_ROOT_PASSWORD=root ncc0706/alpine-mysql
```
2.安装keycloack
```
docker pull jboss/keycloak

# 注意需要提前创建数据库，默认名称为（keycloak）
# -e MYSQL_DATABASE=keycloak
# 指定现有数据库，默认管理员密码
docker run -d -p 10001:8080 --name keycloak -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin -e DB_VENDOR=mysql -e DB_USER=root -e DB_PASSWORD=123456 -e DB_ADDR=192.168.1.110 -e DB_PORT=3306 -e DB_DATABASE=keycloak -e JDBC_PARAMS='useUnicode=true&characterEncoding=UTF-8&useSSL=false&autoReconnect=true&failOverReadOnly=false' jboss/keycloak
```




