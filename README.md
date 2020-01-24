环境搭建

```bash
docker pull zookeeper

docker run -d \
-p 2181:2181 \
--name=zookeeper  \
--privileged zookeeper
```

# 配置文件使用zk 不实用file,server的配置信息存在zookeeper中
```bash

# vi script/config-center/zk/zk-params.txt
zkAddr=localhost:2181
zkHome=/usr/local/zookeeper

cd script/config-center/zk

zk-config.sh


修改config.txt 的 db配置文件

store.mode=db
store.db.datasource=druid
store.db.db-type=mysql
store.db.driver-class-name=com.mysql.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true
store.db.user=root
store.db.password=root
store.db.min-conn=1
store.db.max-conn=3
store.db.global.table=global_table
store.db.branch.table=branch_table
store.db.query-limit=100
store.db.lock-table=lock_table



transport.type=TCP
transport.server=NIO
transport.heartbeat=true
transport.enable-client-batch-send-request=false
transport.thread-factory.boss-thread-prefix=NettyBoss
transport.thread-factory.worker-thread-prefix=NettyServerNIOWorker
transport.thread-factory.server-executor-thread-prefix=NettyServerBizHandler
transport.thread-factory.share-boss-worker=false
transport.thread-factory.client-selector-thread-prefix=NettyClientSelector
transport.thread-factory.client-selector-thread-size=1
transport.thread-factory.client-worker-thread-prefix=NettyClientWorkerThread
transport.thread-factory.boss-thread-size=1
transport.thread-factory.worker-thread-size=8
transport.shutdown.wait=3
service.vgroup_mapping.my_test_tx_group=default
service.default.grouplist=127.0.0.1:8091
service.enableDegrade=false
service.disableGlobalTransaction=false
client.rm.async.commit.buffer.limit=10000
client.rm.lock.retry.internal=10
client.rm.lock.retry.times=30
client.rm.report.retry.count=5
client.rm.lock.retry.policy.branch-rollback-on-conflict=true
client.rm.table.meta.check.enable=false
client.rm.report.success.enable=true
client.tm.commit.retry.count=5
client.tm.rollback.retry.count=5
store.mode=db
store.file.dir=file_store/data
store.file.max-branch-session-size=16384
store.file.max-global-session-size=512
store.file.file-write-buffer-cache-size=16384
store.file.flush-disk-mode=async
store.file.session.reload.read_size=100
store.db.datasource=druid
store.db.db-type=mysql
store.db.driver-class-name=com.mysql.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true
store.db.user=root
store.db.password=root
store.db.min-conn=1
store.db.max-conn=3
store.db.global.table=global_table
store.db.branch.table=branch_table
store.db.query-limit=100
store.db.lock-table=lock_table
server.recovery.committing-retry-period=1000
server.recovery.asyn-committing-retry-period=1000
server.recovery.rollbacking-retry-period=1000
server.recovery.timeout-retry-period=1000
server.max.commit.retry.timeout=-1
server.max.rollback.retry.timeout=-1
client.undo.data.validation=true
client.undo.log.serialization=jackson
server.undo.log.save.days=7
server.undo.log.delete.period=86400000
client.undo.log.table=undo_log
client.log.exceptionRate=100
transport.serialization=seata
transport.compressor=none
metrics.enabled=true
metrics.registry-type=compact
metrics.exporter-list=prometheus
metrics.exporter-prometheus-port=9898
client.support.spring.datasource.autoproxy=false

```

初始化zookeeper配置文件

![](img/zk.png)

![](img/zkconfig.png) 
这里就只使用zookeeper 配置文件，上图中的file.conf 就可以删除了


TC 在db中的存储数据库 记录事务状态，日志回滚日志，
```sql

-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
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
    KEY `idx_gmt_modified_status` (`gmt_modified`, `status`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store BranchSession data
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
    `gmt_create`        DATETIME,
    `gmt_modified`      DATETIME,
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(96),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_branch_id` (`branch_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

```



springboot-dubbo-seata 为seata-sample中的dubbo-mysql-mybatis-druid 例子，比较通用，在此单独clone出来了

```sql

/*
Navicat MySQL Data Transfer

Source Server         : account
Source Server Version : 50614
Source Host           : localhost:3306
Source Database       : db_gts_fescar

Target Server Type    : MYSQL
Target Server Version : 50614
File Encoding         : 65001

Date: 2019-01-26 10:23:10
*/

SET FOREIGN_KEY_CHECKS=0;

-- ----------------------------
-- Table structure for t_account
-- ----------------------------
DROP TABLE IF EXISTS `t_account`;
CREATE TABLE `t_account` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(255) DEFAULT NULL,
  `amount` double(14,2) DEFAULT '0.00',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of t_account
-- ----------------------------
INSERT INTO `t_account` VALUES ('1', '1', '4000.00');

-- ----------------------------
-- Table structure for t_order
-- ----------------------------
DROP TABLE IF EXISTS `t_order`;
CREATE TABLE `t_order` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `order_no` varchar(255) DEFAULT NULL,
  `user_id` varchar(255) DEFAULT NULL,
  `commodity_code` varchar(255) DEFAULT NULL,
  `count` int(11) DEFAULT '0',
  `amount` double(14,2) DEFAULT '0.00',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=64 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of t_order
-- ----------------------------

-- ----------------------------
-- Table structure for t_storage
-- ----------------------------
DROP TABLE IF EXISTS `t_storage`;
CREATE TABLE `t_storage` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `commodity_code` varchar(255) DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  `count` int(11) DEFAULT '0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `commodity_code` (`commodity_code`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of t_storage
-- ----------------------------
INSERT INTO `t_storage` VALUES ('1', 'C201901140001', '水杯', '1000');

-- ----------------------------
-- Table structure for undo_log
-- 注意此处0.3.0+ 增加唯一索引 ux_undo_log
-- ----------------------------
DROP TABLE IF EXISTS `undo_log`;
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of undo_log
-- ----------------------------
SET FOREIGN_KEY_CHECKS=1;

```

![](img/bussiness.png)

一个下单服务，有order，stroge，account

其中bussiness为TM，为发起全局事务的请求者。

现在开始分析源代码

io.seata.config.ConfigurationFactory 为配置文件 默认读取registry.conf 
registry.conf 
```text
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "zk"
  zk {
    cluster = "default"
    serverAddr = "127.0.0.1:2181"
    session.timeout = 6000
    connect.timeout = 2000
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "zk"
  zk {
    serverAddr = "127.0.0.1:2181"
    session.timeout = 6000
    connect.timeout = 2000
  }
}

```

```java

public static final Configuration CURRENT_FILE_INSTANCE;

    static {
        String seataConfigName = System.getProperty(SYSTEM_PROPERTY_SEATA_CONFIG_NAME);
        if (null == seataConfigName) {
            seataConfigName = System.getenv(ENV_SEATA_CONFIG_NAME);
        }
        if (null == seataConfigName) {
            seataConfigName = REGISTRY_CONF_PREFIX;
        }
        String envValue = System.getProperty(ENV_PROPERTY_KEY);
        if (null == envValue) {
            envValue = System.getenv(ENV_SYSTEM_KEY);
        }
        Configuration configuration = (null == envValue) ? new FileConfiguration(seataConfigName + REGISTRY_CONF_SUFFIX,
            false) : new FileConfiguration(seataConfigName + "-" + envValue + REGISTRY_CONF_SUFFIX, false);
        Configuration extConfiguration = null;
        try {
            extConfiguration = EnhancedServiceLoader.load(ExtConfigurationProvider.class).provide(configuration);
            if (LOGGER.isInfoEnabled()) {
                LOGGER.info("load extConfiguration:{}",
                    extConfiguration == null ? null : extConfiguration.getClass().getSimpleName());
            }
        } catch (Exception e) {
            LOGGER.warn("failed to load extConfiguration:{}", e.getMessage(), e);
        }
        CURRENT_FILE_INSTANCE = null == extConfiguration ? configuration : extConfiguration;
    }

```


用spi的方式读取配置,类似dubbospi，但功能简单多了
io.seata.config.zk.ZookeeperConfiguration  初始化一个zkClient ，按照前面初始化的数据目录，读取zk 目录下的配置信息
```java

    public ZookeeperConfiguration() {
        if (zkClient == null) {
            synchronized (ZookeeperConfiguration.class) {
                if (null == zkClient) {
                    zkClient = new ZkClient(FILE_CONFIG.getConfig(FILE_CONFIG_KEY_PREFIX + SERVER_ADDR_KEY),
                        FILE_CONFIG.getInt(FILE_CONFIG_KEY_PREFIX + SESSION_TIMEOUT_KEY, DEFAULT_SESSION_TIMEOUT),
                        FILE_CONFIG.getInt(FILE_CONFIG_KEY_PREFIX + CONNECT_TIMEOUT_KEY, DEFAULT_CONNECT_TIMEOUT));

                    zkClient.setZkSerializer(new ZkSerializer() {
                        @Override
                        public byte[] serialize(Object o) throws ZkMarshallingError {
                            // fixme
                            return ((String)(o)).getBytes();
                        }

                        @Override
                        public Object deserialize(byte[] bytes) throws ZkMarshallingError {
                            return new String(bytes);
                        }
                    });
                }
            }
            if (!zkClient.exists(ROOT_PATH)) {
                zkClient.createPersistent(ROOT_PATH, true);
            }
        }
    }


@Override
    public String getConfig(String dataId, String defaultValue, long timeoutMills) {
        String value;
        if ((value = getConfigFromSysPro(dataId)) != null) {
            return value;
        }
        FutureTask<String> future = new FutureTask<String>(new Callable<String>() {
            @Override
            public String call() {
                String path = ROOT_PATH + ZK_PATH_SPLIT_CHAR + dataId;
                String value = zkClient.readData(path);
                return StringUtils.isNullOrEmpty(value) ? defaultValue : value;
            }
        });
        CONFIG_EXECUTOR.execute(future);
        try {
            return future.get(timeoutMills, TimeUnit.MILLISECONDS);
        } catch (Exception e) {
            LOGGER.error("getConfig {} is error or timeout,return defaultValue {}", dataId, defaultValue);
            return defaultValue;
        }
    }
```


springboot 需要整合seata 

需要两个特别的bean ，其中DataSourceProxy 非必须，为依赖数据库所需要的bean 
```java
    

        @Bean
        public DataSourceProxy dataSourceProxy(DruidDataSource druidDataSource){
            return new DataSourceProxy(druidDataSource);
        } 
        @Bean
        public GlobalTransactionScanner globalTransactionScanner(){
            return new GlobalTransactionScanner("dubbo-gts-seata-example", "my_test_tx_group");
        }
```