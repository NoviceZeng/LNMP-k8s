FROM nginx:1.17.0
COPY nginx.conf /etc/nginx/nginx.conf
RUN rm -rf /etc/nginx/conf.d/default.conf \
    && mkdir -p /data/conf/viba_back \
    && mkdir -p /data/client_h5/dist \
    && mkdir -p /data/dev_back 
COPY client_h5/dist  /data/client_h5/dist/ 
COPY viba_back /data/conf/viba_back/
COPY dev_back /data/dev_back/
