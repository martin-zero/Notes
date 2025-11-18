`Docker Compose` 是 Docker 官方提供的一个**多容器编排工具**，用来**定义并运行多个 Docker 容器的应用程序**。它可以通过一个 `docker-compose.yml` 文件来配置应用中所有服务（容器），然后用一个命令就可以一键启动、停止、构建所有服务。

# docker-compose.yml配置文件示例

```yaml
version: "3.8"  # 指定 Docker Compose 的版本

services:  # 所有要运行的服务容器在这里统一定义

  mysql-db:  # ⛽ 自定义服务名，表示 MySQL 数据库容器
    image: mysql:8  # 使用官方 MySQL 8 的镜像
    container_name: my-mysql  # 指定容器名称为 my-mysql（可选）
    environment:  # 设置环境变量来初始化数据库
      MYSQL_ROOT_PASSWORD: root  # 设置 root 用户密码为 root
      MYSQL_DATABASE: demo  # 初始化时创建名为 demo 的数据库
    ports:
      - "3306:3306"  # 映射主机端口 3306 到容器的 3306 端口
    volumes:
      - ./mysql-data:/var/lib/mysql  # 数据持久化，把容器的数据挂载到本地目录

  backend:  # 🚀 Spring Boot 后端服务容器
    build: ./backend  # 根据 backend 目录中的 Dockerfile 构建镜像
    container_name: my-backend  # 指定容器名称为 my-backend
    ports:
      - "8080:8080"  # 映射本地 8080 端口到容器内 8080（后端服务监听端口）
    depends_on:
      - mysql-db  # 表示此服务依赖 mysql-db，启动顺序优先
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql-db:3306/demo  # Spring Boot 连接数据库的 URL，使用服务名作为主机名
      SPRING_DATASOURCE_USERNAME: root  # 数据库用户名
      SPRING_DATASOURCE_PASSWORD: root  # 数据库密码

  frontend:  # 🎨 Vue 前端服务容器
    build: ./frontend  # 根据 frontend 目录中的 Dockerfile 构建镜像
    container_name: my-frontend  # 指定容器名称
    ports:
      - "80:80"  # 映射主机的 80 端口到容器内 80（默认网页访问端口）
    depends_on:
      - backend  # 表示前端依赖后端服务

```

# 常用命令

启动与停止

|命令|作用|
|---|---|
|docker compose up|构建并启动所有服务（前台运行）|
|docker compose up -d|构建并启动所有服务（后台运行）|
|docker compose down|停止并移除所有服务|
|ocker compose restart|重启所有服务|
|docker compose stop|停止服务，但不移除容器|
|docker compose start|启动已停止的服务|

查看服务状态与日志信息

|命令|作用|
|---|---|
|docker compose ps|查看当前compose项目中运行的服务|
|docker compose logs|查看所有服务日志|
|docker compose logs -f|实时追踪日志|
|docker compose top|查看服务进程状态|