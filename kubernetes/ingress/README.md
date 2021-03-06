# Ingress 

## 1. Ingress 是什么？
- Ingress 是一个封装了的Nginx, 基于 Openresty。

- 通常情况下，service和pod的IP仅可在集群内部访问。集群外部的请求需要通过负载均衡转发到service在Node上暴露的NodePort上，然后再由kube-proxy将其转发给相关的Pod。

- 而Ingress就是为进入集群的请求提供路由规则的集合。

- Ingress可以给service提供集群外部访问的URL、负载均衡、SSL终止、HTTP路由等。为了配置这些Ingress规则，集群管理员需要部署一个Ingress controller，它监听Ingress和service的变化，并根据规则配置负载均衡并提供访问入口。


## 2. Ingress 安装
### 2.1 获取 Yaml 文件

  - 下载文件
    - curl -LO https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml

### 2.2 准备
  - 提前下载好所需Docker image并上传至自己的镜像仓库中

#### 2.2.1 defaultbackend
  - docker pull gcr.io/google_containers/defaultbackend:1.4
  - or
  - docker pull statemood/defaultbackend:1.4

#### 2.2.2 nginx-ingress-controller
  - docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.19.0
  - or
  - docker pull statemood/nginx-ingress-controller:0.19.0

### 2.3 修改文件
  - 将 mandatory.yaml 文件中镜像地址换为自己镜像仓库中地址
  
### 2.4 启动
  - 创建 Controller

        kubectl create -f mandatory.yaml

  - 创建 Service
    - ingress-controller-service.yaml

          apiVersion: v1
          kind: Service
          metadata:
            name: ingress-nginx
            namespace: ingress-nginx
          spec:
            type: NodePort
            ports:
            - name: http
              port: 80
              targetPort: 80
              nodePort: 30080
              protocol: TCP
            - name: https
              port: 443
              targetPort: 443
              nodePort: 30443
              protocol: TCP
            selector:
              app: ingress-nginx

### 2.5 查看状态

    kubectl get po -o wide -n ingress-nginx

## 3. 高可用
### 3.1 Keepalived
- VIP 192.168.50.60


### 3.2 Nginx L4 Proxy
#### 3.2.1 安装 Nginx

- 使用 yum 安装

      yum install -y nginx


#### 3.2.2 配置 Nginx
- 修改 Nginx 配置文件 /etc/nginx/nginx.conf
  - 请注意 Nginx 配置文件路径可能处于自己的安装目录下

        user nginx;
        worker_processes auto;
        error_log /var/log/nginx/error.log;
        
        pid /run/nginx.pid;

        include /usr/share/nginx/modules/*.conf;

        events {
            worker_connections 1024;
        }

        stream {
            upstream http_80 {
                server 192.168.50.60:30080;
            }
            upstream https_443 {
                server 192.168.50.60:30443;
            }
            server {
                listen 80;
                proxy_pass http_80;
            }
            server {
                listen 443;
                proxy_pass https_443;
            }
        }
        http {
            log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                              '$status $body_bytes_sent "$http_referer" '
                              '"$http_user_agent" "$http_x_forwarded_for"';

            access_log /var/log/nginx/access.log  main;
            
            sendfile            on;
            tcp_nopush          on;
            tcp_nodelay         on;
            keepalive_timeout   65;
            types_hash_max_size 2048;
            include             /etc/nginx/mime.types;
            default_type        application/octet-stream;
            include /etc/nginx/conf.d/*.conf;
        }

#### 3.2.3 测试 Nginx 配置文件是否正确
- 使用 nginx 命令

      nginx -t

- 如正常一般会显示类似如下信息:

      nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
      nginx: configuration file /etc/nginx/nginx.conf test is successful

- 如有异常则按提示行及指令进行检查

### 4. 创建项目域名配置
- 文件 demo-php.yaml

      apiVersion: extensions/v1beta1
      kind: Ingress
      metadata:
        name: demo-php
        namespace: project
      spec:
        rules:
        - host: h.linge.io
          http:
            paths:
            - backend:
                serviceName: demo-php
                servicePort: 8080

- 创建 

      kubectl create -f demo-php.yaml

- 访问
  - 直接访问域名 http://h.linge.io/ 即可
