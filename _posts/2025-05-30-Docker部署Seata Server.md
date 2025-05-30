---
layout: post
title: Docker部署Seata Server
date: 2025-05-30
tags: [concurrency]
---

1. 拉取seata server镜像
    ```shell
    docker pull seataio/seata-server:latest
    ```
2. mysql新建seata库，并初始化导入表结构(global_table，branch_table, lock_table)
    ```sql
    CREATE TABLE IF NOT EXISTS `global_table`
    (
        `xid`                       VARCHAR(128) NOT NULL,
        `transaction_id`            BIGINT,
        `status`                    TINYINT      NOT NULL,
        `application_id`            VARCHAR(32),
        `transaction_service_group` VARCHAR(32),
        `transaction_name`          VARCHAR(128),
        `timeout`                   INT,
        `begin_time`                BIGINT,
        `application_data`          VARCHAR(2000),
        `gmt_create`                DATETIME,
        `gmt_modified`              DATETIME,
        PRIMARY KEY (`xid`),
        KEY `idx_status_gmt_modified` (`status` , `gmt_modified`),
        KEY `idx_transaction_id` (`transaction_id`)
    ) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;
    
    CREATE TABLE IF NOT EXISTS `branch_table`
    (
        `branch_id`         BIGINT       NOT NULL,
        `xid`               VARCHAR(128) NOT NULL,
        `transaction_id`    BIGINT,
        `resource_group_id` VARCHAR(32),
        `resource_id`       VARCHAR(256),
        `branch_type`       VARCHAR(8),
        `status`            TINYINT,
        `client_id`         VARCHAR(64),
        `application_data`  VARCHAR(2000),
        `gmt_create`        DATETIME(6),
        `gmt_modified`      DATETIME(6),
        PRIMARY KEY (`branch_id`),
        KEY `idx_xid` (`xid`)
    ) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;
    
    CREATE TABLE IF NOT EXISTS `lock_table`
    (
        `row_key`        VARCHAR(128) NOT NULL,
        `xid`            VARCHAR(128),
        `transaction_id` BIGINT,
        `branch_id`      BIGINT       NOT NULL,
        `resource_id`    VARCHAR(256),
        `table_name`     VARCHAR(32),
        `pk`             VARCHAR(36),
        `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',
        `gmt_create`     DATETIME,
        `gmt_modified`   DATETIME,
        PRIMARY KEY (`row_key`),
        KEY `idx_status` (`status`),
        KEY `idx_branch_id` (`branch_id`),
        KEY `idx_xid` (`xid`)
    ) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;
    
    CREATE TABLE IF NOT EXISTS `distributed_lock`
    (
        `lock_key`       CHAR(20) NOT NULL,
        `lock_value`     VARCHAR(500) NOT NULL,
        `expire`         BIGINT,
        primary key (`lock_key`)
    ) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;
    
    INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('AsyncCommitting', ' ', 0);
    INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryCommitting', ' ', 0);
    INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryRollbacking', ' ', 0);
    INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('TxTimeoutCheck', ' ', 0);
    ```
3. 配置seata server配置（application.yml）
   ```yaml
   # seata console端口
   server:
     port: 7091
   
   spring:
     application:
       name: seata-server
   
   logging:
     config: classpath:logback-spring.xml
     file:
       path: ${log.home:${user.home}/logs/seata}
   
   # seata console 用户名密码
   console:
     user:
       username: seata
       password: seata
   seata:
     config:
       # 使用nacos作为seata配置中心
       type: nacos
       nacos:
         server-addr: host.docker.internal:8848
         namespace: 5c181ad6-86da-40b1-8665-c251d87d5917
         group: SEATA_GROUP
         username: nacos
         password: nacos
         context-path: /nacos
         # seata server部分配置放入到配置中心，如db、事务组等
         data-id: seataServer.properties
     registry: 
       # 使用nacos作为seata注册中心
       type: nacos
       nacos:
         application: seata-server
         server-addr: host.docker.internal:8848
         group: SEATA_GROUP
         namespace: 5c181ad6-86da-40b1-8665-c251d87d5917
         username: nacos
         password: nacos
         context-path: /nacos
     # seata server端口，注册到nacos上的端口
     server:
       service-port: 8091
     # seata console 用户认证信息
     security:
       secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
       tokenValidityInMilliseconds: 1800000
       ignore:
         urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.jpeg,/**/*.ico,/api/v1/auth/login,/health,/error
   ```
4. nacos配置事务组及数据库（seataServer.properties）
   ```properties
   # 配置事务组名称为seata_tx_group，该事务组映射的集群为default
   service.vgroupMapping.seata_tx_group=default
   
   #事务会话信息存储方式
   store.mode=db
   #事务锁信息存储方式
   store.lock.mode=db
   #事务会话信息存储方式
   store.session.mode=db
   #存储方式为db
   store.db.dbType=mysql
   store.db.datasource=druid
   store.db.driverClassName=com.mysql.cj.jdbc.Driver
   store.db.url=jdbc:mysql://host.docker.internal:3306/seata?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
   store.db.user=root
   store.db.password=123456
   store.db.minConn=5
   store.db.maxConn=30
   store.db.globalTable=global_table
   store.db.branchTable=branch_table
   store.db.distributedLockTable=distributed_lock
   ```
5. 将镜像运行容器
   ```shell
   containt_name=seata-dev
   image_name=seataio/seata-server:latest
   docker stop $containt_name
   docker rm -f $containt_name
   
   # 获取宿主机IP（linux 可通过 --network host指定本地）
   HOST_IP=$(ifconfig en0 | grep "inet " | awk '{print $2}')
   # 运行容器，映射console端口（7091），映射server端口（8091），挂载配置文件，挂载日志目录
   docker run -d --name=$containt_name -p 7091:7091 -p 8091:8091 -e SEATA_IP=$HOST_IP --privileged=true --restart=always -v $PWD/resources:/seata-server/resources -v $PWD/logs:/root/logs/seata $image_name
   ```




