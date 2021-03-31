# 使用 docker 镜像搭建 siber

运行 siber 并不需要很高的配置。4c8g 足矣。

## **创建 siber 目录**

```text
mkdir -p /home/siber
cd /home/siber

mkdir config logs
```



## 配置文件

以下为配置文件的示例，可自行修改 IP、port、user、password 等信息。

nginx、mongo、Minio（oss）可与其他应用共用。

### siber-server

在  `/home/siber/config/siber.conf` 文件中，复制以下内容：

```text
debug = true

app_name = "siber"

envoy_addr_test = "172.17.202.51:9000"

[flag]
private_deploy = true

[log]
dir = "/home/works/program/logs"
runtime = true
debug=true

[protofile]
root_path = "/home/works/program/logs"

[port]
prometheus = 51000

[jaeger]
disable = false
agent_port = 5719
payload = false
service_name = "siber"

[mongo]
uri="mongodb://admin:admin@172.17.202.51:27027/admin?"
name="admin"
password="admin"
dbname="admin"
host="172.17.202.51"
port=27027
maxPoolSize =200
minPoolSize=30
maxIdleTimeS=86400

[mongo_ops]
uri="mongodb://admin:admin@172.17.202.51:27027/admin?"
name="admin"
password="admin"
dbname="admin"
host="172.17.202.51"
port=27027
maxPoolSize =200
minPoolSize=30
maxIdleTimeS=86400

[cibot]
host="172.17.202.51"

[wechatGroup]
host="172.17.202.51"
```

### siber-web

在文件 `/home/siber/config/siber-web.conf` 中复制以下内容：

```text
{
  "env": {
    /*注释*/ 项目启动端口号
    "port": "9080",
    /*注释*/ 服务端主机，可能有 HTTP 和 gRPC 两种形式的接口
    "apiHost": {
      "http": "http://172.17.202.51:88"
    },
    /*注释*/ 服务运行环境，需要读配置字典，开发=dev, 测试=test, 灰度=pre, 线上=prod, 私有部署=pvt
    "mode": "pvt",
    "oss": {
      "region": "",
      "bucket": "laiye-im-saas",
      "endpoint": "http://172.17.202.51:9000",
      "accessKey": "siber",
      "accessSecret": "siber"
    }
  }
}
```

### siber-nginx

在文件 `/home/siber/config/siber-nginx.conf` 中复制以下内容：

```text
user  nginx;
worker_processes  8;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  204800;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    proxy_http_version 1.1;
    proxy_set_header Connection "";

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" '
                      '"$upstream_addr" "$upstream_status" "$upstream_response_time" '
                      '"$request_time" "$request_length"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    tcp_nopush     on;

    keepalive_timeout  0;
    server_tokens off;

    gzip  on;
    gzip_buffers 16 8k;
    gzip_comp_level 5;
    gzip_http_version 1.1;
    gzip_min_length 256;
    gzip_proxied any;
    gzip_vary on;
    gzip_types text/plain text/css text/javascript application/x-javascript application/javascript application/json image/jpeg image/jpg imag
e/gif image/png;
    client_header_buffer_size 128k;
    large_client_header_buffers 4 128k;


    upstream siber {
        server 172.17.202.51:11002;
    }

    upstream siber_test {
        server 172.17.202.51:11001;
    }


    server {
        client_max_body_size 40m;
        listen 80;
        server_name proxy;
        #rewrite ^(.*) https://$server_name$1 permanent;
        access_log  /var/log/nginx/access.log main;
        error_log   /var/log/nginx/error.log;
        proxy_pass_header Server;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504 non_idempotent;
        proxy_next_upstream_tries 3;
        proxy_next_upstream_timeout 30;
        client_header_buffer_size 200k;
        large_client_header_buffers 4 32k;

        location / {
            proxy_pass http://siber;
            add_header Access-Control-Allow-Origin *;
            proxy_next_upstream http_502;
            proxy_next_upstream_tries 3;
        }
        location /siberhttp {
            proxy_pass http://siber_test;
            add_header Access-Control-Allow-Origin *;
            proxy_next_upstream http_502;
            proxy_next_upstream_tries 3;
        }
    }
}
```

## docker-compose

将以下内容复制到  `/home/siber/docker-compose.yml` 文件中

```text
version: "3.5"

services:

  siber-server:
    image: tinaliu/siber-server:latest
    container_name: siber-server
    ports:
      - 11001:9080
      #- 12001:19080
    #environment:
    #  - CLUSTER_NAME=KUBE
    #  - ENV=prod
    entrypoint:
      - /home/works/program/api-test-siber
    command:
      - --port=19080
      - --gwport=9080
      - --conf=/home/works/program/conf/online.conf
      - -log=send
    volumes:
      - .#/protos:/home/works/program/im-saas-msgs-protos
      - ./config/siber.conf:/home/works/program/conf/online.conf
      - ./logs/siber-server:/home/works/program/logs
      - ./protos:/home/works/program/im-saas-msgs-protos
    depends_on:
      - siber-mongo

  siber-web:
    image: tinaliu/siber-web:latest
    container_name: siber-web
    ports:
      - 11002:9080
      #- 12002:19080
    volumes:
      - ./config/siber-web.conf:/apps/conf/online.conf
      - ./logs/siber-web:/home/works/program/logs
    depends_on:
      - siber-server

  siber-mongo:
    image: mongo:4.0
    container_name: siber-mongo
    ports:
      - 27027:27017
    volumes:
      - ./siber_mongo/:/siber_mongo/
    command:
      - mongod
      - --port
      - '27017'
      - --shardsvr
      - --noprealloc
      - --smallfiles
      - --oplogSize
      - '16'
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=admin
      
  siber-nginx:
    container_name: siber-nginx
    image: nginx:1.11
    volumes:
      - ./config/siber-nginx.conf:/etc/nginx/nginx.conf:ro
      - ./logs:/var/log/nginx:rw
      - /etc/localtime:/etc/localtime
    ports:
      - "88:80"

  minio:
    restart: always
    image: minio/minio:RELEASE.2020-01-16T22-40-29Z
    container_name: minio
    volumes:
      - ./data/data1:/data1
      - ./data/data2:/data2
      - ./data/data3:/data3
      - ./data/data4:/data4

    network_mode: host
    environment:
      MINIO_ACCESS_KEY: siber
      MINIO_SECRET_KEY: siber
      MINIO_PROMETHEUS_AUTH_TYPE: "public"
    command: server  --address :9000 http://172.17.202.51/data1 http://172.17.202.51/data2 http://172.17.202.51/data3 http://172.17.202.51/data4
    #ports:
    #  - "9000:9000"

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
```

保存后，使用命令 `docker-compose up -d` 启动即可。

如有疑问或建议，可以评论此篇文章，也可以在 “来也技术” 公众号中留言。

