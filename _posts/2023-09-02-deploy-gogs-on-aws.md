---
layout: post
title: 在亚马逊AWS上部署GOGS
description: 亚马逊AWS部署GOGS代码仓库
categories: [deployment]
tags: [deployment, notes, git, gogs]
redirect_from:
  - /2023/09/02/

---

# 背景

早期在阿里云上部署了GOGS，好多年稳定运行。最近机器磁盘空间告急，因此着急需要扩容。阿里云的早期机器扩容无法弹性扩，那是相当费劲；大概是先把磁盘做成镜像，再申请一块更大的磁盘挂上去，同时把镜像写到这个磁盘上。

# 解法

为了一了百了，干脆用一台更便宜的AWS的机器替代了。AWS在磁盘扩容、CPU内存升级上几乎无痛，比阿里云香。
步骤如下：
备份并导出数据
1. 把阿里云的GOGS数据全部备份（/var/gogs），下面有git、gogs、ssh等3个目录；其中git是文件存储所在地；gogs则是相关配置；ssh是账号相关授权的public key，git自己的public + private key等等。
2. 把阿里云的GOGS Mysql数据全部备份。这里因为也是运行在docker里面的，粗暴点直接把mount的mysql存储文件全部导出就好了。

我们在Aws上重新部署Gogs + Mysql。
可以参考 [这篇旧文](https://www.ateijelo.com/blog/2016/07/09/share-port-22-between-docker-gogs-ssh-and-local-system)
这里有一个特别注意的是放了一个脚本，每次ssh的请求都会通过这个脚本转发到docker容器里面（命令中SSH端口是10022）。

```bash
#!/bin/sh

ssh -p 10022 -o StrictHostKeyChecking=no git@127.0.0.1 \
"SSH_ORIGINAL_COMMAND=\"$SSH_ORIGINAL_COMMAND\" $0 $@"
```
3. 为了版本兼容性问题，还是用了老版本镜像
```bash
docker pull gogs/gogs:0.12
docker pull mysql:5.7
```
4. 上传备份(这里比较简单，直接把备份内容通过scp命令复制上去就好了)
```bash
scp -r ./git-backup/gogs.tar.gz ubuntu@xxxx.compute.amazonaws.com:/home/ubuntu
scp -r ./git-backup/mysql.tar.gz ubuneut@xxxx.compute.amazonaws.com:/home/ubuntu
```
5. 把文件解压缩，我的mysql文件放在`/var/mysql/gogs`，gogs相关数据放在 `/var/gogs`了。
6. 创建git账号，参考自[gitea的攻略](https://docs.gitea.com/installation/install-from-binary)

```bash
#On Ubuntu/Debian:
adduser \
   --system \
   --shell /bin/bash \
   --gecos 'Git Version Control' \
   --group \
   --disabled-password \
   --home /home/git \
   git

#On Fedora/RHEL/CentOS:
groupadd --system git
adduser \
   --system \
   --shell /bin/bash \
   --comment 'Git Version Control' \
   --gid git \
   --home-dir /home/git \
   --create-home \
   git
```

7. 重新启动容器
MySQL
```bash
docker run \
  -d \
  --name "/gogs-mysql" \
  --volume "/var/mysql/gogs:/var/lib/mysql" \
  --restart "no" \
  --network "bridge" \
  --expose "3306/tcp" \
  --env "MYSQL_DATABASE=gogs" \
  --env "MYSQL_ROOT_PASSWORD=xxxxx" \
  mysql:5.7
```

以及gogs；GOGS这里有个坑最后再说
```bash
docker run \
  --name "/gogs2" \
  --volume "/var/gogs:/data" \
  --link "/gogs-mysql:/gogs2/mysql" \
  --restart "no" \
  --publish "127.0.0.1:10022:22/tcp" \
  --publish "0.0.0.0:3002:3000/tcp" \
  --hostname "fa5613dc5db5" \
  --expose "22/tcp" \
  --expose "3000/tcp" \
  --env "GOGS_CUSTOM=/data/gogs" \
  --env "PGID=112" \
  --env "PUID=115" \
  "gogs/gogs:0.12"   
```
8. 设置nginx反向代理。这里据说是cloudflare需要企业版才能支持web + git的方式，因此就先不折腾了，直接部署nginx。现在nginx和certbot结合比之前方便了。
可以参考[这篇文章](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04#step-5-%E2%80%93-setting-up-server-blocks-(recommended))
这个certbot的nginx插件会在`/etc/site-enabled/default`中加一些证书的地址和一些兜底的解析，如果不喜欢也可以自己单独写一个server文件。
cerbot现在也会自动处理LetsEncrypt的证书更新问题（文章说是单独启动了一个服务来处理更新问题）
9. 申请和安装证书，大体上可以用测试命令先把解析或者相关配置验证完
```bash
certbot --test-cert --nginx -d git.xxxx.com
```
之后再把上面的`--test-cert`去掉，通过续费的方式申请正式线上证书。
10. 如果顺利的话，这会儿应该可以通过`ssh -vT git@git.xxxx.com`来验证了。验证不通过会提示permission denied。

# 踩坑记录
1. 步骤2里面的脚本需要给运行权限
2. 需要在git用户的authorize_keys最上面增加一行，后面半截`ssh-rsa xxx`是给这个git账号生产的一个ssh key，通过`ssh-keygen`就可以生成，生成的pub文件内容替换即可
```
no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-rsa xxxxxxx git@aws-host
```
3. 重点！重点！
 AWS的Ubuntu系统，默认创建了一个Ubuntu账号（uid=1000，gid=1000）；在Gogs的[docker环境中](https://github.com/gogs/gogs/blob/069f1ed9a4651dd2598a513d94278a400a04e5a7/docker/start.sh#L51)，如果默认没有指定git的Uid和Gid，即UserId和GroupId，它就会默认认为git的uid、gid都是1000。
 这样导致的问题是，docker启动时会对所有的文件撸一遍文件读写权限，这样就把所有文件git的权限都刷成宿主机的ubuntu了。
 然而！我们在docker环境下，宿主机的git账号的ssh请求，都会被转发到docker容器里面，且为了保持一致，他们是共用一套.ssh目录。这样docker启动后，就导致.ssh目录下authorized_keys、id_rsa.pub、id_rsa等等这种文件宿主机都无权限读了。
 解决办法也简单，通过`id git`和`getent group git`就能知道我们git账号的uid和gid是啥，然后在gogs的docker启动时作为环境变量穿进去。前文引用的代码中有提及。
 中间我也试过，在容器中直接改id，但是重启容器后，运行start.sh，权限又被刷回去了。
```
/var/gogs/git# docker exec -it gogs2 sh
/app/gogs # id git
uid=1000(git) gid=1000(git) groups=1000(git),1000(git)
/app/gogs # groupmod -g 112 git
/app/gogs # id git
uid=1000(git) gid=112(git) groups=112(git),112(git)
/app/gogs # usermod -u 115 -g 112 git
/app/gogs # id git
uid=115(git) gid=112(git) groups=112(git),112(git)
```
真正解决问题其实就两个docker运行时传入的参数
```
--env "PGID=112" \
--env "PUID=115" \
```

# 结论
这个坑花了一晚上才出来，整体把数据、账号等都原封不动搬过来了，效果还不错。整体过程简要记录，万一有后来者到这里可以参考。如果有需要可以email联系 `qinwen.shi#gmail.com`