# Ubuntu22.04系统docker的安装

参考如下文章:

- [docker官网](https://www.docker.com/)

- [docker guides](https://docs.docker.com/desktop/)

- [ubuntu上docker安装](https://docs.docker.com/engine/install/ubuntu/)

- [使用 GPG Key 来构建签名、加密及认证体系](https://zhuanlan.zhihu.com/p/481900853?utm_id=0)


## docker的安装

1) 建立docker的apt仓库

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

2) 安装最新版本docker包

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

3) 校验docker engine是否安装成功

```
sudo docker run hello-world
```

该命令会从docker仓库下载一个测试镜像，然后运行在一个容器中。当容器成功运行会打印相关的确认信息。