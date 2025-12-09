---
title: Gitlab CICD学习记录
tags:
  - 服务器运维
  - CICD
categories:
  - 服务器运维
date:
  '[object Object]': null
abbrlink: 63036
---
<meta name="referrer" content="no-referrer"/>
# Gitlab CICD学习记录

# 用Flask在K8s集群编写后端项目

## 编写app.py

编写一个最简单的后端项目，命名为app.py，返回hello world!

```
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return "hello world!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

## 用pipreqs导出依赖文件

```
pip3 install pipreqs
```

```
pipreqs . --encoding=utf8 --force
```

![image-20230607202155815](https://gitee.com/beatrueman/images/raw/master/20251208210638359.png)

## 构建Docker镜像

编写dockerfile

```
FROM python:3.9

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python","app.py"]
```

```
docker build -t beatrueman/ci:2.0 .
```

上传镜像

```
docker push beatrueman/ci:2.0
```

## K8s集群部署后端服务

使用命令创建Pod。创建一个名为`flask-hello-world`的Pod，并将容器的端口映射到主机的端口8080。

```
kubectl run flask-hello-world --image=beatrueman/ci:2.0 --port=8080 
```

![image-20230609140113784](https://gitee.com/beatrueman/images/raw/master/20251208210645849.png)

使用命令暴露服务。以LoadBalancer类型创建服务，映射到Pod的8080端口

```
 kubectl expose pod flask-hello-world --type=LoadBalancer --port=80 --target-port=8080
```

查看服务

```
kubectl get service flask-hello-world
```

![image-20230609140541562](https://gitee.com/beatrueman/images/raw/master/20251208210645850.png)

开放32091端口，访问外部ip

![image-20230609140631331](https://gitee.com/beatrueman/images/raw/master/20251208210645851.png)

![image-20230609140646482](https://gitee.com/beatrueman/images/raw/master/20251208210645852.png)

成功

# 安装runner

## 获取registration-token

![image-20230611174544623](https://gitee.com/beatrueman/images/raw/master/20251208210645853.png)

## 使用docker安装runner

### 安装

```
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

### 注册

```
docker exec -it gitlab-runner bash
```

```
gitlab-runner register \
--non-interactive \
--url "https://gitlab.com/" \
--registration-token "GR1348941F_CsNBNspPP2DccgUVcx" \ 
--executor "docker" \
--docker-image ubuntu:22.04 \
--description "docker-runner" \
--maintenance-note "Free-form maintainer notes about this runner" \
--tag-list "docker" \
--run-untagged="true" \
--locked="false" \
--access-level="not_protected"

```

成功

![image-20230610222258752](https://gitee.com/beatrueman/images/raw/master/20251208210645854.png)

#一直把registration-token当成runner里的token了，导致后面不停报错#

## 使用helm安装runner（失败）

### 下载package并解压

```
helm repo add gitlab https://charts.gitlab.io
helm pull gitlab/gitlab-runner
tar -zxvf gitlab-runner-0.53.2.tgz
```

文件目录如下

![image-20230609224911559](https://gitee.com/beatrueman/images/raw/master/20251208210645855.png)

### 配置token填入和docker鉴权

```
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-runner-secret
type: Opaque
data:
# 配置runner的token
  runner-registration-token: "R1IxMzQ4OTQxRl9Dc05CTnNwUFAyRGNjZ1VWY3gK" # 将token转为base64后写这里
  runner-token: ""
---
apiVersion: v1
data:
# 配置docker的config.json
  .dockerconfigjson: "ewogICAgICAgICJhdXRocyI6IHsKICAgICAgICAgICAgICAgICJodHRwczovL2luZGV4LmRvY2tlci5pby92MS8iOiB7CiAgICAgICAgICAgICAgICAgICAgICAgICJhdXRoIjogIlltVmhkSEoxWlcxaGJqcHNlWGd3TXpBNU1UWT0iCiAgICAgICAgICAgICAgICB9CiAgICAgICAgfQp9Cgo="
kind: Secret
metadata:
  name: regsecret
type: kubernetes.io/dockerconfigjson
```

### 在values.yaml中应用配置文件

```
gitlabUrl: https://gitlab.com/Beatrueman/test.git

runners:
# 在这里指定我们刚刚创建的secret
  secret: gitlab-runner-secret
  imagePullSecrets: regsecret
  tags: "k8s"
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "ubuntu:20.04"
```

# 镜像上传

### 自定义kaniko镜像

提前在**CI/CD &gt; Variables**下设置好变量

![image-20230611224208441](https://gitee.com/beatrueman/images/raw/master/20251208210645856.png)

在`/root/kaniko`下编写dockerfile和build-upload

![image-20230610152445422](https://gitee.com/beatrueman/images/raw/master/20251208210645857.png)

dockerfile

```
FROM lonewalkerchr/kaniko:debug

COPY ./build-upload /kaniko/build-upload

SHELL ["/busybox/sh"]

RUN ["/busybox/chmod", "a+x", "/kaniko/build-upload"]
```

build-upload

```
printf "%s" "${DOCKER_AUTH}" > /kaniko/.docker/config.json

/kaniko/executor \
     --force \
     --cache \
     --context "${CI_PROJECT_DIR}" \
     --dockerfile dockerfile \
     --destination ${destination}
```

构建镜像并上传至dockerhub

```
docker build -t beatrueman/kaniko:1.0 .

docker push beatrueman/kaniko:1.0
```

# 上传代码

## 创建并配置.gitlab-ci.yaml

添加KUBE_CONFIG变量，内容为`~/.kube/config`的内容

![image-20230610153746130](https://gitee.com/beatrueman/images/raw/master/20251208210645858.png)

创建并配置.gitlab-ci.yaml

```
stages:
  - build
  - deploy

imagebuilder:
  image: beatrueman/ci:kaniko
  stage: build
  variables:
    destination: beatrueman/ci:2.0
  script:
    - /kaniko/build-upload

deploy:
  image: bitnami/kubectl
  stage: deploy
  script:
    - echo ${KUBE_CONFIG} > kubeconfig
    - kubectl run flask-hello-world --image=beatrueman/ci:2.0 --port=8080 
    - kubectl expose pod flask-hello-world --type=LoadBalancer --port=80 --target-port=8080
```

## 上传到仓库

```
git init

git add .

git commit -m "first"

git remote add origin https://gitlab.com/Beatrueman/test-ci.git


git push -u origin master
```

![image-20230610155728095](https://gitee.com/beatrueman/images/raw/master/20251208210645859.png)

# 问题与解决

1.构建Docker镜像时CMD的时候少了一个后引号，折腾了半天。#感谢一下杨鑫同学了，多亏了他的慧眼。

2.安装runner时用的是registration-token，不是创建runner时候的token。

3.${CI_PROJECT_DIR}变量是自带的，是runner里面的一个不固定的位置，可以通过它来找到绝对路径，不能自行指定，不然会找不到文件导致报错。

![image-20230612110258681](https://gitee.com/beatrueman/images/raw/master/20251208210645860.png)

```
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple
```

pip安装的时候使用-i指定镜像源，下载速度会提升很多很多。

5..gitlab-ci.yaml里deploy加entrypoint覆盖。

![image-20230611223818531](https://gitee.com/beatrueman/images/raw/master/20251208210645861.png)

6..gitlab-ci.yaml里deploy阶段执行kubectl命令建议每条命令都加--kuboeconfig=./kubeconfig，保证每条命令都有权限。
