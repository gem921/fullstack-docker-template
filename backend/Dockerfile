# 后端基础镜像：可换成官方的 eclipse-temurin:8-jre / 17-jre，或你自己仓库的镜像
FROM eclipse-temurin:8-jre
# 设置工作目录
WORKDIR /opt
# 拷贝 jar 包到工作目录（把 your-app.jar 换成你 jar 的真实文件名）
COPY your-app.jar app.jar
# 创建日志目录（与 docker-compose 的挂载对应）
RUN mkdir -p /opt/logs
# 暴露端口
EXPOSE 8080
# 启动：用 exec 让 java 成为 PID 1，docker stop 的信号才能传入，触发优雅停机
ENTRYPOINT ["sh", "-c", "exec java -jar app.jar"]
