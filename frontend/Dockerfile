# 基础镜像（固定版本，避免 latest 带来的不可复现问题）
FROM nginx:1.27-alpine

# 删除默认配置
RUN rm /etc/nginx/conf.d/default.conf

# 拷贝自定义 Nginx 配置
COPY default.conf /etc/nginx/conf.d/

# 拷贝 SSL 证书到容器内（文件名需与 ssl/ 目录一致）
COPY ssl/certkey.pem /opt/nginx/ssl/certkey.pem
COPY ssl/fullchain.pem /opt/nginx/ssl/fullchain.pem

# 拷贝前端打包产物到 Nginx web 根目录
COPY dist/ /usr/share/nginx/html/
