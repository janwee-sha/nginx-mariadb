# 使用NGINX作为MySQL的TCP负载均衡器

> Refers to https://abhizer.com/using-nginx-as-a-tcp-load-balancer-for-your-mysql-or-any-other-database/

## Overview
随着应用程序的膨胀，我们往往需要扩展程序的基础架构。这篇文章是使用NGINX作为MySQL的TCP负载均衡器的实践。

使用TCP负载均衡器的好处：
- 对数据库进行负载均衡，避免数据库过载。
- 应用程序可以使用同样的IP或者主机名访问集群中的所有数据库
- 如果一个节点崩溃，可以绕过该节点并保持应用持续运行
- 横向扩展比纵向扩展更简单

## Let's Do It
我将使用Docker Compose在Linux系统中部署多个MySQL节点和一个NGINX节点。Docker Compose启动文件`docker-compose.yml`如下：

```
version: '3.8'

networks: 
  nginx_mariadb:

services:
  load_balancer:
    image: nginx:stable-alpine
    container_name: nginx-load-balancer
    ports: 
      - "3306:3306"
      - "33060:33060"
    volumes: 
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - mariadb-master
      - mariadb-slave
    networks:
      - nginx_mariadb

  mariadb-master:
    image: 'bitnami/mariadb:latest'
    hostname: "master.janwee.ubuntu"
    container_name: mariadb_master
    volumes:
      - ./mariadb:/var/lib/mysql
    restart: unless-stopped
    tty: true
    environment:
      MARIADB_REPLICATION_MODE: master
      MARIADB_DATABASE: nerddb
      MARIADB_USER: janwee
      MARIADB_PASSWORD: janwee
      MARIADB_ROOT_PASSWORD: janwee
      MARIADB_REPLICATION_USER: rplusr
      MARIADB_REPLICATION_PASSWORD: janwee
    networks:
      - nginx_mariadb

  mariadb-slave:
    image: 'bitnami/mariadb:latest'
    hostname: "slave.janwee.ubuntu"
    container_name: mariadb_slave
    restart: unless-stopped
    tty: true
    environment:
      MARIADB_REPLICATION_MODE: slave
      MARIADB_REPLICATION_USER: rplusr
      MARIADB_REPLICATION_PASSWORD: janwee
      MARIADB_MASTER_HOST: "master.janwee.ubuntu"
      MARIADB_MASTER_PORT_NUMBER: 3306
      MARIADB_MASTER_ROOT_PASSWORD: janwee
    networks:
      - nginx_mariadb
    depends_on:
      - mariadb-master
```

这里使用 bitnami 的镜像是因为更容易部署 MySQL主从复制架构。主节点的主机名是 `master.janwee.ubuntu`,从节点的主机名是 `slave.janwee.ubuntu`。将三个节点都放在同一个网络下，在NGINX负载均衡器中暴露 `3306` 和 `33060` 端口。
NGINX节点中的 `volumes` 选项将配置文件 `nginx/nginx.conf` 挂载到容器内部：

```
worker_processes 1;

# Configuration of connection processing
events {
    worker_connections 1024;
}

# Configuration specific to TCP/UDP and affecting all virtual servers
stream {
    log_format log_stream '$remote_addr - [$time_local] $protocol $status $bytes_sent $bytes_received $session_time "$upstream_addr"';
    access_log /var/log/nginx/mysql.log log_stream;

    upstream mariadb_read {
        server master.janwee.ubuntu:3306; 
        server slave.janwee.ubuntu:3306;
    }
    
    # Configuration of a TCP virtual server
    server {
        listen 3306;
	# Specify the several addresse
        proxy_pass mariadb_read; 
        proxy_connect_timeout 1s;
        error_log /var/log/nginx/mysql_error.log;
    }

    upstream mariadb_write {
        server master.janwee.ubuntu:3306;
    }

    server {
        listen 33060;
        proxy_pass mariadb_write;
        proxy_connect_timeout 1s;
        error_log /var/log/nginx/mysql_error.log;
    }
}
```

这里设置了两个输入流：`mariadb_read` 和 `mariadb_write`。read同时指向主节点和从节点，write指向主节点。

然后将nginx主机名 `janwee.ubuntu` 加入本地DNS配置文件 `/etc/hosts`中，在Linux终端中使用如下命令：

```
echo "127.0.0.1 janwee.ubuntu" >> /etc/hosts
```

在`docker-compose.yml`所在目录下运行如下命令启动容器组：

```
docker compose up -d
```

测试连接到3306端口并使用 `SELECT @@hostname;`查询当前主机名：

```
$ mysql -h janwee.ubuntu -P 3306 -u root -pjanwee -e "select @@hostname";
+---------------------+
| @@hostname          |
+---------------------+
| slave.janwee.ubuntu |
+---------------------+

$ mysql -h janwee.ubuntu -P 3306 -u root -pjanwee -e "select @@hostname";
+----------------------+
| @@hostname           |
+----------------------+
| master.janwee.ubuntu |
+----------------------+
```

测试连接到 `33060` 端口并使用 `SELECT @@hostname;` 查询当前主机名：

```
$ mysql -h janwee.ubuntu -P 33060 -u root -pjanwee -e "select @@hostname";
+---------------------+
| @@hostname          |
+---------------------+
| master.janwee.ubuntu |
+---------------------+

$ mysql -h janwee.ubuntu -P 33060 -u root -pjanwee -e "select @@hostname";
+----------------------+
| @@hostname           |
+----------------------+
| master.janwee.ubuntu |
+----------------------+
```

测试结果表明，当连接 3306 端口时NGINX会在主节点和从节点之间变更，而连接33060时只会连接到主节点。
