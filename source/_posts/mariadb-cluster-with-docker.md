---
title: 用docker创建一个mariadb集群
date: 2019-01-09 09:15:41
tags:
- mariadb
- docker
---
## 创建数据目录
```shell
mkdir -p /home/docker/mariadb/cluster0/conf
mkdir -p /home/docker/mariadb/cluster0/data
mkdir -p /home/docker/mariadb/cluster1/conf
mkdir -p /home/docker/mariadb/cluster1/data
mkdir -p /home/docker/mariadb/cluster2/conf
mkdir -p /home/docker/mariadb/cluster2/data
```

### 创建数据库配置文件

wsrep_cluster_address：在初始化的时候不需要，初始化完成以后再取消注释，应该是用于节点连接集群使用的，应该是填写集群内任意一个或多个地址即可

```shell
vim /home/docker/mariadb/cluster0/conf/server.cnf
```

```ini
[server]
[mysqld]
server_id=130
pid-file=/var/run/mysqld/mysqld.pid
socket=/var/run/mysqld/mysqld.sock
basedir=/usr
datadir=/var/lib/mysql
tmpdir=/tmp
user=mysql
skip-external-locking
skip-name-resolve
character-set-server=utf8
port=3306


#
# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
#bind-address        = 127.0.0.1
#
# * Fine Tuning
#
max_connections=1000
connect_timeout=5
wait_timeout=600
max_allowed_packet=16M
thread_cache_size=128
sort_buffer_size=4M
bulk_insert_buffer_size    =16M
tmp_table_size=32M
max_heap_table_size    =32M
[galera]  
wsrep_causal_reads=ON  #节点应用完事务才返回查询请求  
wsrep_provider_options="gcache.size=128M"#同步复制缓冲池  
wsrep_certify_nonPK=ON   #为没有显式申明主键的表生成一个用于certificationtest的主键，默认为ON  
#log-bin=/app/galera/mysql-bin  #如果不接从库，注释掉  
#log_slave_updates=1         #如果不接从库，注释掉  
query_cache_size=0           #关闭查询缓存  
wsrep_on=ON   #开启全同步复制模式  
wsrep_provider=/usr/lib/galera/libgalera_smm.so #galera library  
wsrep_cluster_name=MariaDB-Galera-Cluster  
#wsrep_cluster_address="gcomm://192.168.1.9:4567,192.168.1.9:4568,192.168.1.9:4569"  #galera cluster URL  
wsrep_node_name=mariadb-0 
#wsrep_node_address=172.18.0.4
wsrep_sst_auth=syncuser:syncuser
#wsrep_sst_method=xtrabackup-v2
wsrep_sst_method=rsync
binlog_format=row  
default_storage_engine=InnoDB  
innodb_autoinc_lock_mode=2   #主键自增模式修改为交叉模式  
wsrep_slave_threads=8  #开启并行复制线程，根据CPU核数设置  
innodb_flush_log_at_trx_commit=0   #事务提交每隔1秒刷盘  
innodb_buffer_pool_size=2G
[embedded]  
[mariadb]  
[mariadb-10.3]
```
## 运行docker
```shell
docker run --rm -d \
    --name mariadb-cluster0 \
    -v /home/docker/mariadb/cluster0/conf:/etc/mysql/conf.d \
    -v /home/docker/mariadb/cluster0/data:/var/lib/mysql \
    -e MYSQL_ALLOW_EMPTY_PASSWORD=yes \
    -e TIMEZONE=Asia/Shanghai \
    mariadb
```
## 创建同步用户
```sql
    -- init.sql
    GRANT ALL PRIVILEGES ON *.* TO 'syncuser'@'%' IDENTIFIED BY 'syncuser' WITH GRANT OPTION;
    flush privileges;
    shutdown;
```

可以把上面的sql保存到conf/init.sql 直接执行
```shell
    docker exec <containerId> bash -c "/etc/mysql/conf.d/init.sql"
```

## 正式启动集群

后面的节点相应的增加 同步端口与数据库端口  
--wsrep-new-cluster仅第一个节点启动时添加

```shell
docker run -d \
    --name mariadb-cluster0 \
    --expose 4567 \
    -p 4567:4567 -p 3306:3306 \
    -v /home/docker/mariadb/cluster0/conf:/etc/mysql/conf.d \
    -v /home/docker/mariadb/cluster0/data:/var/lib/mysql \
    mariadb [--wsrep-new-cluster]
```

## 2019/03/14 更新
      
当初年少无知，上面的操作是可以，但是，太麻烦了
      
时隔这么久，再来填一下坑

### 开始

mariadb初始化脚本 init.sql(就是上面的那个)，只需要GRANT ALL ... 就可以了
      
还有一个 init.sh
      
```shell
cat <<EOF > /etc/mysql/conf.d/galera.cnf

# Node specifics 
[mysqld] 
# next 3 params disabled for the moment, since they are not mandatory and get changed with each new instance.
# they also triggered problems when trying to persist data with a backup service, since also the config has to be 
# persisted, but HOSTNAME changes at container startup.
# wsrep-node-name = $HOSTNAME 
# wsrep-sst-receive-address = $HOSTNAME
# wsrep-node-incoming-address = $HOSTNAME
# Cluster settings
wsrep-on=ON
wsrep-cluster-name = "$CLUSTER_NAME" 
wsrep-cluster-address = $CLUSTER_ADDRESS
wsrep-provider = /usr/lib/galera/libgalera_smm.so 
wsrep-provider-options = "gcache.size=256M;gcache.page_size=128M;debug=no" 
wsrep-sst-auth = "$GALERA_USER:$GALERA_PASS" 
wsrep_sst_method = rsync
binlog-format = row 
default-storage-engine = InnoDB 
innodb-doublewrite = 1 
innodb-autoinc-lock-mode = 2 
innodb-flush-log-at-trx-commit = 2 

EOF

```

有了这两个文件以后 把他们放在initdb文件夹里面
      
然后创建一个dockerfile
      
```yml
FROM mariadb

ADD initdb /docker-entrypoint-initdb.d/.

RUN touch /etc/mysql/conf.d/galera.cnf \
    && chown mysql.mysql /etc/mysql/conf.d/galera.cnf \
    && chown mysql.mysql /docker-entrypoint-initdb.d/*.sql

ENV GALERA_USER=syncuser \
    GALERA_PASS=syncuser \
    CLUSTER_NAME=docker_cluster \
    MYSQL_ALLOW_EMPTY_PASSWORD=yes
```

最后，再来一个docker-compose.yml 编排集群
      
```yml
      version: '3'
services:
    node1:
        image: 'mariadb-cluster'
        environment:
            - CLUSTER_ADDRESS="gcomm://"
        ports:
            - "3307:3306"
    node2:
        image: 'mariadb-cluster'
        environment:
            - CLUSTER_ADDRESS="gcomm://node1"
        links:
            - node1
        ports:
            - "3308:3306"
    node3:
        image: 'mariadb-cluster'
        environment:
            - CLUSTER_ADDRESS="gcomm://node1"
        links:
            - node1
        ports:
            - "3309:3306"
```

这样，想开多少个节点只需要在docker-compose里面多加一个就可以了

最后 docker-compose up 搞定，是不是简单多了呢