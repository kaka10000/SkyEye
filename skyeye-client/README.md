# 项目介绍
日志生产系统，包括应用系统的日志入队列、埋点、应用系统注册等
- 自定义一些log框架的appender，包含logback和log4j
- 向注册中心注册应用
- 应用埋点进行监控报警
- rpc trace数据产生器

# 使用方式
## logback
### 依赖
gradle或者pom中加入skyeye-client的依赖

``` xml
compile ("skyeye:skyeye-client:0.0.1") {
  exclude group: 'log4j', module: 'log4j'
}
```
### 配置
在logback.xml中加入一个kafkaAppender，并在properties中配置好相关的值，如下（rpc这个项目前支持none和dubbo，所以如果项目中有dubbo服务的配置成dubbo，没有dubbo服务的配置成none，以后会支持其他的rpc框架，如：thrift、spring cloud等）：

``` xml
<property name="APP_NAME" value="your-app-name" />
<!-- kafka appender -->
<appender name="kafkaAppender" class="com.jthink.skyeye.client.kafka.logback.KafkaAppender">
  <encoder class="com.jthink.skyeye.client.kafka.logback.encoder.KafkaLayoutEncoder">
    <layout class="ch.qos.logback.classic.PatternLayout">
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS};${CONTEXT_NAME};${HOSTNAME};%thread;%-5level;%logger{96};%line;%msg%n</pattern>
    </layout>
  </encoder>
  <topic>${kafka.topic}</topic>
  <rpc>none</rpc>
  <zkServers>${zookeeper.servers}</zkServers>
  <mail>${mail}</mail>
  <keyBuilder class="com.jthink.skyeye.client.kafka.partitioner.AppHostKeyBuilder" />

  <config>bootstrap.servers=${kafka.bootstrap.servers}</config>
  <config>acks=0</config>
  <config>linger.ms=100</config>
  <config>max.block.ms=5000</config>
  <config>client.id=${CONTEXT_NAME}-${HOSTNAME}-logback</config>
</appender>
```
## log4j
### 依赖
gradle或者pom中加入skyeye-client的依赖

``` xml
compile ("skyeye:skyeye-client:0.0.1") {
  exclude group: 'ch.qos.logback', module: 'logback-classic'
}
```
### 配置
在log4j.xml中加入一个kafkaAppender，并在properties中配置好相关的值，如下（rpc这个项目前支持none和dubbo，所以如果项目中有dubbo服务的配置成dubbo，没有dubbo服务的配置成none，以后会支持其他的rpc框架，如：thrift、spring cloud等）：

``` xml
<appender name="kafkaAppender" class="com.jthink.skyeye.client.kafka.log4j.KafkaAppender">
  <param name="topic" value="${kafka.topic}"/>
  <param name="zkServers" value="${zookeeper.servers}"/>
  <param name="app" value="${app.name}"/>
  <param name="mail" value="${mail}"/>
  <param name="rpc" value="dubbo" />
  <param name="bootstrapServers" value="${kafka.bootstrap.servers}"/>
  <param name="acks" value="0"/>
  <param name="maxBlockMs" value="5000"/>
  <param name="lingerMs" value="100"/>

  <layout class="org.apache.log4j.PatternLayout">
    <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS};APP_NAME;HOSTNAME;%t;%p;%c;%L;%m%n"/>
  </layout>
</appender>
```
## 注意点
## logback
- 目前公司很多项目采用的是spring-boot，版本为1.3.6.RELEASE，该版本自带logback版本为1.1.7，该版本结合kafka有bug，需要降低一个版本
- logback bug: http://jira.qos.ch/browse/LOGBACK-1158, 1.1.8版本会fix
- 示例：

``` shell
compile ("org.springframework.boot:spring-boot-starter") {
  exclude group: 'ch.qos.logback', module: 'logback-classic'
  exclude group: 'ch.qos.logback', module: 'logback-core'
}
compile "ch.qos.logback:logback-classic:1.1.6"
compile "ch.qos.logback:logback-core:1.1.6"
```
### log4j
由于log4j本身的appender比较复杂难写，所以在稳定性和性能上没有logback支持得好，应用能使用logback请尽量使用logback
### 中间件
如果项目中有使用到zkClient、，统一使用自己打包的版本，以防日志收集出错或者异常（PS：zk必须为3.4.6版本，尽量使用gradle进行打包部署）
### rpc trace
使用自己打包的dubbox（https://github.com/JThink/dubbox/tree/skyeye-trace），在soa中间件dubbox中封装了rpc的跟踪

``` shell
compile "com.101tec:zkclient:0.9.1-up"
compile ("com.alibaba:dubbo:2.8.4-skyeye-trace") {
  exclude group: 'org.springframework', module: 'spring'
}
```
## 应用注册中心设计
### zookeeper注册中心节点tree
![](zknode.png)
- kafkaAppender初始化向zk进行app节点注册，并写入相关的信息
- kafkaAppender发生异常暂停工作会向app节点写入相关信息，以便监控系统能够实时感知并发送报警
- app正常或者异常退出后，zk中的app临时节点会消失，shutdownhook会正常运行，监控系统能够实时感知并发送报警
- zk中有永久节点用来记录app的最近一次部署信息

## 埋点对接
### 日志类型
 日志类型 | 说明
:------  |:-----
 normal  | 正常入库日志
 invoke_interface  | api调用日志
 middleware_opt  | 中间件操作日志(目前仅支持hbase和mongo)
 job_execute  | job执行日志
 rpc_trace  | rpc trace跟踪日志
 custom_log  | 自定义埋点日志
 thirdparty_call  | 第三方系统调用日志
### 正常日志

``` shell
LOGGER.info("我是测试日志打印")
```
### api日志

``` shell
// 参数依次为EventType(事件类型)、api、账号、请求耗时、成功还是失败、具体自定义的日志内容
LOGGER.info(ApiLog.buildApiLog(EventType.invoke_interface, "/app/status", "800001", 100, EventLog.MONITOR_STATUS_SUCCESS, "我是mock api成功日志").toString());
LOGGER.info(ApiLog.buildApiLog(EventType.invoke_interface, "/app/status", "800001", 10, EventLog.MONITOR_STATUS_FAILED, "我是mock api失败日志").toString());
```
### 中间件日志

``` shell
// 参数依次为EventType(事件类型)、MiddleWare(中间件名称)、操作耗时、成功还是失败、具体自定义的日志内容
LOGGER.info(EventLog.buildEventLog(EventType.middleware_opt, MiddleWare.HBASE.symbol(), 100, EventLog.MONITOR_STATUS_SUCCESS, "我是mock middle ware成功日志").toString());
LOGGER.info(EventLog.buildEventLog(EventType.middleware_opt, MiddleWare.MONGO.symbol(), 10, EventLog.MONITOR_STATUS_FAILED, "我是mock middle ware失败日志").toString());
```
### job执行日志

```
// job执行仅仅处理失败的日志（成功的不做处理，所以只需要构造失败的日志）, 参数依次为EventType(事件类型)、job 的id号、操作耗时、失败、具体自定义的日志内容
LOGGER.info(EventLog.buildEventLog(EventType.job_execute, "application_1477705439920_0544", 10, EventLog.MONITOR_STATUS_FAILED, "我是mock job exec失败日志").toString());
```

### 第三方请求日志

```
// 参数依次为EventType(事件类型)、第三方名称、操作耗时、成功还是失败、具体自定义的日志内容
LOGGER.info(EventLog.buildEventLog(EventType.thirdparty_call, "xx1", 100, EventLog.MONITOR_STATUS_FAILED, "我是mock third 失败日志").toString());
LOGGER.info(EventLog.buildEventLog(EventType.thirdparty_call, "xx1", 100, EventLog.MONITOR_STATUS_SUCCESS, "我是mock third 成功日志").toString());
LOGGER.info(EventLog.buildEventLog(EventType.thirdparty_call, "xx2", 100, EventLog.MONITOR_STATUS_SUCCESS, "我是mock third 成功日志").toString());
LOGGER.info(EventLog.buildEventLog(EventType.thirdparty_call, "xx2", 100, EventLog.MONITOR_STATUS_FAILED, "我是mock third 失败日志").toString());
```

