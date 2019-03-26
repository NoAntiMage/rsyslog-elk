# elk+syslog
https://github.com/NoAntiMage/rsyslog-elk


需要使用到的文件路径
```
tree 

.
├── README.md
├── elk
│   ├── docker-compose.yml
│   └── logstash
│       └── pipeline
│           └── logstash.conf
├── nginx
│   └── docker-compose.yml
└── rsyslog
    ├── docker-compose.yml
    └── rsyslog.d
        └── 60-logstash.conf
```

### 配置syslog
rsyslog/rsyslog.d/60-logstash.conf
```
# :programname, contains, "docker"
*.* @@${LOGSTASH_SERVER}:5000;json_lines
```

### 启动syslog
/rsyslog
docker-compose -up -d 

### 配置确认elk
elk/logstash/pipeline
```
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

### 启动elk
/elk
docker-compose up -d

### 配置之后、启动nginx测试一下
/nginx
cat docker-compose.yml

```yml
services:
  nginx:
    image: nginx:alpine
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://${SYSLOG_SERVER}:514"
        tag: "{{.Name}}.{{.ID}}"
    ports:
      - "8080:80"
    restart: always

```
docker-compose up -d



**知识点：**
1. docker-logging-driver 日志驱动使用：syslog。
2. container-log导入syslog
3. syslog接入elk
4. curl $NGINX-IP:8080 ，产生一些nginx日志
5. 登录：$ip:5601 查看