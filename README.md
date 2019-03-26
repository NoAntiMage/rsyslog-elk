# elk+syslog
https://github.com/NoAntiMage/rsyslog-elk

### 启动elk+syslog

docker-compose.yml
```
version: '2'
services:
  app:
    image: voxxit/rsyslog
    ports:
      - "514:514"
      - "514:514/udp"
    volumes:
      - ./rsyslog.d:/etc/rsyslog.d
    restart: always
```

### 配置syslog
cat /rsyslog.d/01-json-template.conf
```
template(name="json_lines"
  type="list"
  option.json="on") {
    constant(value="{")
      constant(value="\"@timestamp\":\"")       property(name="timereported" dateFormat="rfc3339")
      constant(value="\", \"@version\":\"1")
      constant(value="\",\"tag\":\"")           property(name="syslogtag")
      constant(value="\",\"message\":\"")       property(name="msg")
      constant(value="\",\"severity\":\"")      property(name="syslogseverity-text")
      constant(value="\",\"facility\":\"")      property(name="syslogfacility-text")
      constant(value="\",\"hostname\":\"")      property(name="hostname")
      constant(value="\", \"procid\":\"")       property(name="procid")
      constant(value="\", \"programname\":\"")  property(name="programname")
    constant(value="\"}\n")
}
```
cat 60-logstash.conf
```
*.* @@${LOGSTASH_SERVER_IP}:${LOGSTASH_SERVER_PORT};json_lines
```

### 启动elk
docker-compose.yml

```
version: '2'

services:

  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro

    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk

  logstash:
    build:
      context: logstash/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - /var/:/var/
    ports:
      - "5000:5000"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    build:
      context: kibana/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - ./kibana/config/:/usr/share/kibana/config:ro
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:

  elk:
    driver: bridge

logstash.conf
input {
        tcp {
                port => 5000
                # type => "rsyslog"
                codec => "json"
        }
}

output {
        elasticsearch {
                hosts => "elasticsearch:9200"
        }
}

```








reference:
https://www.jianshu.com/p/bf2eb121ac62
https://zhuanlan.zhihu.com/p/24912074

需求：docker 日志 自动 保存为  YYYY-MM-DD-HH.log的形式
调研步骤：

### 1. 了解docker-log ，确认docker-log的log-driver类型
https://docs.docker.com/config/containers/logging/configure/

json-file	The logs are formatted as JSON. The default logging driver for Docker. [查询日志保存路径
/var/lib/docker/containers/]
local	Writes logs messages to local filesystem in binary files using Protobuf.
syslog	Writes logging messages to the syslog facility. The syslog daemon must be running on the host machine.
journald	Writes log messages to journald. The journald daemon must be running on the host machine.
gelf	Writes log messages to a Graylog Extended Log Format (GELF) endpoint such as Graylog or Logstash.

none：容器不输出任何日志；
json-file：容器输出的日志以 JSON 格式写入文件中（默认）；
syslog：容器输出的日志写入宿主机的 Syslog 中；
journald：容器输出的日志写入宿主机的 Journald 中；
gelf：容器输出到日志以 GELF（Graylog Extended Log Format）格式写入 Graylog中；
fluentd：容器输出的日志写入宿主机的 Fluented 中；
awslogs：容器输出的日志写入 Amazon CloudWatch Logs 中；
splunk：容器输出的日志写入 splunk 中；
etwlogs：容器输出的日志写入 ETW （Event Tracing for Windows）；
mats：容器输出的日志写入 NATS 服务中；
我们可以在 docker run 命令中通过  --log-driver 参数来设置具体的 Docker 日志驱动，也可以通过 --log-opt 参数来指定对应日志驱动的相关选项。就拿 json-file 来说，其实可以这样启动 Docker 容器：

更改log-driver的方法
linux： 
cat /etc/docker/daemon.json
```
{
  "log-driver": "syslog",
  "log-opts": {
    "syslog-address": "tcp://$SYSCONFIG_SERVER:514"
  }
}
```
查询log-driver的方式
docker info --format '{{.LoggingDriver}}'
docker info | grep 'Logging Driver'




### 2. 查询哪种日志 log-driver 符合条件
   2.1 官方都不符合 x
   2.2 三方log-driver  x




### 3. 继续做下去耗时较多，考虑elk
 logging-driver使用：syslog，将docker容器日志输出到syslog，syslog接入elk

### 3.1 docker-log-driver改为 syslog
```
 cat > /etc/docker/daemon.json << EOF
{
   "log-driver": "syslog",
   "log-opts": {
     "syslog-address": "tcp://$SYSCONFIG_SERVER:514"
   }
}
EOF
```

systemctl restart docker 

### 3.2 Rsyslog服务开启

docker-compose.yml
```
version: '2'
services:
  app:
    image: voxxit/rsyslog
    ports:
      - "514:514"
      - "514:514/udp"
    volumes:
      - ./rsyslog.d:/etc/rsyslog.d
    restart: always
```




### 3.3 logstash 收集syslog

docker-compose.yml


### 3.4 启动nginx 产生一些日志测试一下


知识点：


syslog本地部署
rsyslogd -v
```
rsyslogd 8.24.0, compiled with:
    PLATFORM:               x86_64-redhat-linux-gnu
    PLATFORM (lsb_release -d):      
    FEATURE_REGEXP:             Yes
    GSSAPI Kerberos 5 support:      Yes
    FEATURE_DEBUG (debug build, slow code): No
    32bit Atomic operations supported:  Yes
    64bit Atomic operations supported:  Yes
    memory allocator:           system default
    Runtime Instrumentation (slow code):    No
    uuid support:               Yes
    Number of Bits in RainerScript integers: 64
```

### docker-logging-driver使用syslog
```
 cat > /etc/docker/daemon.json << EOF
{
   "log-driver": "syslog",
   "log-opts": {
     "syslog-address": "tcp://{SYSLOG_SERVER_IP}:514"
   }
}
EOF
```
```
systemctl restart docker
```

vi /etc/rsyslog.conf
```
$ModLoad imtcp
$InputTcpServerRun 514
```


```
syslog配置详解
#rsyslog v3 config file

# if you experience problems, check

# http://www.rsyslog.com/troubleshoot for assistance

#### MODULES ####    加载 模块

$ModLoad imuxsock.so  –> 模块名    # provides support for local system logging (e.g. via logger command) 本地系统日志

$ModLoad imklog.so                    # provides kernel logging support (previously done by rklogd)

#$ModLoad immark.so              # provides –MARK– message capability

# Provides UDP syslog reception

# 允许514端口接收使用UDP协议转发过来的日志

#$ModLoad imudp.so

#$UDPServerRun 514

# Provides TCP syslog reception

# 允许514端口接收使用TCP协议转发过来的日志

#$ModLoad imtcp.so

#$InputTCPServerRun 514

#### GLOBAL DIRECTIVES ####

定义日志格式默认模板  

# Use default timestamp format

$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat  

# File syncing capability is disabled by default. This feature is usually not required,

# not useful and an extreme performance hit

#$ActionFileEnableSync on

#### RULES ####

# Log all kernel messages to the console.

# Logging much else clutters up the screen.

#kern.*                                                 /dev/console    关于内核的所有日志都放到/dev/console(控制台)

# Log anything (except mail) of level info or higher.

# Don’t log private authentication messages!

# 记录所有日志类型的info级别以及大于info级别的信息到/var/log/messages，但是mail邮件信息，authpriv验证方面的信息和cron时间任务相关的信息除外

*.info;mail.none;authpriv.none;cron.none                /var/log/messages  

# The authpriv file has restricted access.

# authpriv验证相关的所有信息存放在/var/log/secure

authpriv.*                                              /var/log/secure  

# Log all the mail messages in one place.

# 邮件的所有信息存放在/var/log/maillog; 这里有一个-符号, 表示是使用异步的方式记录, 因为日志一般会比较大

mail.*                                                  -/var/log/maillog  

# Log cron stuff

# 计划任务有关的信息存放在/var/log/cron

cron.*                                                  /var/log/cron  

# Everybody gets emergency messages

# 记录所有的大于等于emerg级别信息, 以wall方式发送给每个登录到系统的人

*.emerg                                                 *                  *代表所有在线用户  

# Save news errors of level crit and higher in a special file.

# 记录uucp,news.crit等存放在/var/log/spooler

uucp,news.crit                                          /var/log/spooler  

# Save boot messages also to boot.log     启动的相关信息

local7.*                                                /var/log/boot.log

#:rawmsg, contains, “sdns_log” @@192.168.56.7:10514

#:rawmsg, contains, “sdns_log” ~

# ### begin forwarding rule ###  转发规则

# The statement between the begin … end define a SINGLE forwarding

# rule. They belong together, do NOT split them. If you create multiple

# forwarding rules, duplicate the whole block!

# Remote Logging (we use TCP for reliable delivery)

#

# An on-disk queue is created for this action. If the remote host is

# down, messages are spooled to disk and sent when it is up again.

#$WorkDirectory /var/spppl/rsyslog # where to place spool files

#$ActionQueueFileName fwdRule1 # unique name prefix for spool files

#$ActionQueueMaxDiskSpace 1g   # 1gb space limit (use as much as possible)

#$ActionQueueSaveOnShutdown on # save messages to disk on shutdown

#$ActionQueueType LinkedList   # run asynchronously

#$ActionResumeRetryCount -1    # infinite retries if host is down

# remote host is: name/ip:port, e.g. 192.168.0.1:514, port optional

#*.* @@remote-host:514                    # @@表示通过tcp协议发送    @表示通过udp进行转发

#local3.info  @@localhost:514

#local7.*                                    #            @@192.168.56.7:514

# ### end of the forwarding rule ###

```