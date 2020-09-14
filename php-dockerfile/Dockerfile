FROM  php:7.2-fpm
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/bin/
RUN install-php-extensions redis amqp mysqli pdo_mysql \
    && mkdir -p /data/dev_back
#考虑到以上相关PHP模块安装耗时，可以将以上打包成新镜像
COPY dev_back /data/dev_back/
