title: docker
date: 1970-1-1
categories: 
- MISC
---
写好dockerfile,xinetd
启动docker:
docker build -t name .
docker run -d -p port:port -h "name" --name="name" name

docker ps,列出所有正在运行的实例
docker images,列出所有镜像

docker rm -f name 删除实例
docker rmi id
docker exec -it id command
docker cp id:/ /

