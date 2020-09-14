传统PHP项目迁移到k8s上
PHP NGINX k8s dockerfile

1. 迁移前的准备
多个PHP项目，各个项目框架相同，lnmp方式。迁移到nginx可以实现快速扩容，不通项目使用同一套k8s而不需要新建太多虚拟机，节省了成本。

前期只讲无状态应用迁移到容器中，有状态应用如mysql、redis、rabbitmq仍使用之前的，因为目前项目全都是放在云平台，只需要将k8s部署到之前相同的网段，既可以实现平滑无缝迁移。

传统项目中，nginx和PHP都是安装在同一台server中，通过fastcgi监听127.0.0.1:9000来获取PHP代码的解析结果，考虑到部署的便捷性，将PHP-FPM和nginx放在同一个pod里面的不同容器内，这样仍然可以通过localhost间进行通信。

2. 相关dockerfile及yaml文件
所有文件已放在GitHub，可点击查看 NoviceZeng/nginx-php-k8s

(1). nginx的Dockerfile
nginx.conf一般不会修改，直接放入镜像里面，此处关于nginx调用redsi、mysql、rabbitmq的PHP相关配置main-local.php、params-local.php也放入镜像

FROM nginx:1.17.0
COPY nginx.conf /etc/nginx/nginx.conf
RUN rm -rf /etc/nginx/conf.d/default.conf \
    && mkdir -p /data/conf/viba_back \
    && mkdir -p /data/client_h5/dist \
    && mkdir -p /data/dev_back 
COPY client_h5/dist  /data/client_h5/dist/ 
COPY viba_back /data/conf/viba_back/
COPY dev_back /data/dev_back/
(2). php的Dockerfile
根据实际项目安装PHP扩展，此处需要安装redis、rabbitmq和mysql的扩展，通过php-extension-installer使用PHP官方镜像，很easy实现了模块扩展

FROM  php:7.2-fpm
#使用install-php-extensions安装PHP的扩展模块redis、rabbitmq、mysql
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/bin/
RUN install-php-extensions redis amqp mysqli pdo_mysql \
    && mkdir -p /data/dev_back
#考虑到以上相关PHP模块安装耗时，可以将以上打包成新镜像
COPY dev_back /data/dev_back/
(3). nginx的ConfigMap
nginx中conf.d下面的配置文件通过configmap通过存储卷挂载到pod中，如下：

kind: ConfigMap 
apiVersion: v1
metadata:
  name: nginx-php-config 
data: 
  m.abc.com.conf: |
    server {
        listen      80;
        server_name m.abc.com;
        charset     utf-8;
        root    /data/client_h5/dist;
        access_log  /var/log/nginx/m.abc.com-access_json.log json;
        error_log   /var/log/nginx/m.abc.com-error_json.log;
        index   index.html index.htm;
                error_page   500 502 503 504  /50x.html;
        location = /50x.html {
                root   html;
            }
        location / {
                try_files $uri $uri/ /index.html$is_args$args;
            }
        location ~ \.php$ {
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_pass 127.0.0.1:9000;
                try_files $uri =404;
            }
        }
  api.abc.com.conf: |
    server {
        listen      81;
        server_name _;
        charset     utf-8;
        root /data/dev_back/appapi/web;
        access_log  /var/log/nginx/m.abc.com-access_json.log json;
        error_log   /var/log/nginx/m.abc.com-error_json.log;
        index   index.php;
            error_page   500 502 503 504  /50x.html;
        location = /50x.html {
                root   html;
            }
        location / {
                if ( $request_method = OPTIONS ) {
                        add_header Access-Control-Allow-Origin *;
                        add_header Access-Control-Allow-Methods GET,POST,PUT,PATCH,DELETE,OPTIONS,HEAD;
                        add_header Access-Control-Allow-Headers Origin,X-Requested-With,Content-Type,Accept,Authorization;
                        return 200;
                }
                try_files $uri $uri/ /index.php$is_args$args;
            }
        location ~ \.php$ {
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param  HTTPS    on;
                fastcgi_pass 127.0.0.1:9000;
                try_files $uri =404;
            }
        location ~* /\. {
                deny all;
            }
        }
  test.conf: |
      server {
        listen 82 default_server;
        root /data/;
        index index.php;
        server_name _;
        location / {
          try_files $uri $uri/ =404;
        }
        location ~ \.php$ {
          include fastcgi_params;
          fastcgi_param REQUEST_METHOD $request_method;
          fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
          fastcgi_pass 127.0.0.1:9000;
        }
      }
(4). nginx/php deployment
部署deployment需要先部署一下配置：
kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml
kubectl apply -f nginx-cm.yml

部署deployment: kubectl apply -f php-nginx-deployment.yaml
注意事项：

更新发布前端js代码时，只需要打包构建nginx镜像，推送到镜像仓库后后，修改deployment中新的image地址，例如：nginx-1.17.0:3
更新发布后端PHP代码时，由于nginx和PHP都需要获取PHP代码，因此此处需要将新的后端代码分别打包构建成新的php、nginx镜像，因为后端代码不需要编译，即便多一次打包，理论上不会花费太多时间。如果nginx和PHP放入同一个容器中，就如同部署到同一台虚拟机中，只需要发布一次，不过这样违反k8s一个容器一个进程的理念，不能解耦软件依赖
deployment中添加reloader注解，实现configmap变更后的热更新，即修改nginx配置文件，例如增加域名后，实现deployment滚动升级（至少75%的pod副本处于运行状态）
kind: Deployment
apiVersion: apps/v1
metadata:
  name: php-fpm-nginx 
  annotations:
    configmap.reloader.stakater.com/reload: "nginx-php-config"
spec:
  selector:
    matchLabels:
      app: php-fpm-nginx
  replicas: 1 
  template: 
    metadata: 
      labels:
        app: php-fpm-nginx
    spec:
      containers:
        - name:  php-fpm
          image: 10.70.128.51/library/php-fpm:3
          ports:
            - containerPort: 9000 
        - name: nginx 
          image: 10.70.128.51/library/nginx-1.17.0:3
          ports:
            - containerPort: 80
          volumeMounts:
           # nginx调用redis、rabbitmq、mysql等配置文件，由于之前架构设计，这里暂且将conf和代码打包到镜像，而不是挂载
           # - name: php-conf
           #   mountPath: /data/
            - name: nginx-php-config
              mountPath: /etc/nginx/conf.d/
      volumes:
        - name: nginx-php-config
          configMap:
            name: nginx-php-config
        #- name: php-conf
        #  persistentVolumeClaim:
        #    claimName: php-conf
(5). k8s service.yaml
kubectl apply -f service.yaml

kind: Service
apiVersion: v1 
metadata: 
  name: php-fpm-nginx
spec:
  selector:
    app: php-fpm-nginx
  type: NodePort
  ports:
    - name: m
      port: 80 
      targetPort: 80 
      nodePort: 30010
    - name: api
      port: 81 
      targetPort: 81 
      nodePort: 30011
(6). k8s service.yaml
前端通过api接口调用后端，此处api接口也需要通过ingress暴露出来
kubectl apply -f service.yaml

kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: php-fpm-nginx
spec:
  rules:
    - host: m.abc.com
      http:
        paths:
          - backend:
              serviceName: php-fpm-nginx
              servicePort: 80
    - host: api.abc.com
      http:
        paths:
          - backend:
              serviceName: php-fpm-nginx
              servicePort: 81
