---
title: "Docker初探——用Docker部署Homepage"
date: 2024-04-08T00:48:00+08:00
draft: false
image: "docker.webp"
tags: ["Docker", "服务器"]
categories: ["技术"]
---

前段时间看到一个很有意思的项目，[Homepage](https://github.com/gethomepage/homepage)。就是一个很好看的静态导航页，想到我那用HTML手搓的（甚至没写CSS）简陋到用“简陋”一词都不足以形容的导航页，我顿时心动了。

我原本以为既然Document里声称这个项目是纯静态的网页，那我是不是在本地生成好静态页面之后扔到我某个服务器的www目录，再随便写点配置就行了呢？但我鼓捣了半天才发现并非如此。首先这个项目并不希望你这样做，比如他的npm run build默认不会生成dist文件夹；此外这个网页也并不是完全静态的，它涉及一些系统参数的获取（CPU占用、内存占用等）以及天气API，如果非要生成一个静态页面，那这些API也不能正常工作。

而官方最推荐的部署方式就是使用Docker，这下从没用过Docker的我也打算尝试一下了，顺便把我的Minecraft服务器也搬到Docker上，这样就可以完全变成一个服务，方便地自启动和重启了（每次重启机器都去开TMUX手动启动实在是太不优雅了）。

简单了解了一下，Docker其实就是一个介于虚拟机和直接多开程序之间的一种方案，它的资源占用要比开虚拟机更少，同时也能给不同的服务提供独立的环境，避免互相干扰。

看了一下Docker的Manual以及一些教程，我决定使用docker compose，这个按照我的理解就是把一个Docker的配置写进一个配置文件里，这样每次启动这个Docker就只需要读这个配置文件就行了，这样更容易部署成一个服务。

首先创建一个新文件夹，在里面创建一个名叫compose.yaml的文件，这就是这个Docker的配置文件了。按照Homepage的教程，这个文件应该有如下内容：

```yaml
version: "3.3"
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - 3000:3000
    volumes:
      - /path/to/config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```

这里的image代表所使用的Docker镜像；ports代表把外部的端口（左侧）映射到Docker内部的某个端口；volumes则是代表将外部的目录（左侧）挂载到Docker内部的某个位置。restart一项是我额外添加的，为了让这个容器在开机时能自启动。

写完这个compose.yaml，我们就可以把自己写好的Homepage配置文件放到上面指定的目录下，待会运行Docker的时候Docker里面的Homepage服务就可以根据这个配置文件渲染对应的页面。

接下来在终端输入：

```bash
docker compose up -d
```

这个命令就会根据你在compose.yaml里指定的配置为你生成一个Docker并运行，这时我们curl localhost:3000，可以看到页面源代码，说明配置成功了。再在终端输入：

```bash
sudo systemctl enable docker.service
```

这样docker服务就会被启用，如果你在compose.yaml正确配置了restart参数，那这个容器就能开机自启动了。

接下来就是在你的nginx或者其他服务器软件里设置一个端口转发，让你的页面能正确显示在公网，我的配置如下，供参考：

```
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name nvg.clf3.org;
    ssl_certificate /certificate/path;
    ssl_certificate_key /privkey/path;
    client_max_body_size 50G;
    location / {
      proxy_pass http://localhost:3000/;
      proxy_set_header Host $http_host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
    }
}
server {
    listen 80;
    listen [::]:80;
    server_name nvg.clf3.org;
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}
```

最后配置好的效果[在这里](https://nvg.clf3.org)：

![](docker.webp)

感觉用Docker部署这种配置好就不用怎么动的服务还是很适合的，现在有点想把我的所有服务都docker化了。主要是想配置的服务越来越多，但每次都开一个新的虚拟机，即使是64G内存也有点捉襟见肘，如果都放到一个Linux机器上，又担心环境冲突。等到我又想部署新项目的时候，可能就考虑把之前的Wordpress、Gitea之类的给Docker化了。