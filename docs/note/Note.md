# RocketMQ源码笔记

### 启动类设置

```
# NameServer
org.apache.rocketmq.namesrv.NamesrvStartup

# Evironment variables
ROCKETMQ_HOME=$PROJECT_DIR$/config
```

![1-1](img\1-1.png)

```
# Broker
org.apache.rocketmq.broker.BrokerStartup

# 启动参数
-c $PROJECT_DIR$/config/conf/broker.conf

# Evironment variables
ROCKETMQ_HOME=$PROJECT_DIR$/config
```

![1-2](img\1-2.png)

