#Docker 学习

**镜像**
---

```
sudo docker images #查看所有
sudo docker rmi 镜像id #删除
sudo docker tag hub.c.163.com/library/busybox busybox #改名
```

**容器**
---

```
sudo docker run -p 8080:80 --name my_nginx -i -t my_nginx /bin/bash #运行nginx 映射容器80端口到本机8080端口
sudo docker run -i -t -v /home/weihaozhe/Downloads:/usr/Downloads --name centos daocloud.io/centos:7 /bin/bash #-v映射目录
sudo docker ps -a #查看容器列表
sudo docker stop 容器id  #停止某个容器
sudo docker stop $(sudo docker ps -a -q) #停止所有容器
sudo docker rm 容器id #删除某个容器
sudo docker commit eea67209cecf my_nginx #保存对容器的修改
```

**根据Dokerfile创建镜像** 
---
`sudo docker build -t mysql-5.7 .`
>  注意: Docker Client会默认发送Dockerfile同级目录下的所有文件到Dockerdaemon中。

###相关资料
[Docker —— 从入门到实践](https://www.gitbook.com/book/yeasy/docker_practice)