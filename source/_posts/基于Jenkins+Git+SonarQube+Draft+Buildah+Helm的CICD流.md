---
title: 基于Jenkins+Git+SonarQube+Draft+Buildah+Helm的CI/CD流
date: {{ date }}
tags: [Achievements]
categories: [Achievements]
---

## 构建部署工具及镜像作用介绍

### 工具

[Jenkins](https://www.jenkins.io/zh/)：通过Pipeline实现CI/CD流

[Sonar](https://www.sonarsource.com/products/sonarqube/)：进行代码检查

[draft](https://github.com/Azure/draft)：根据项目自动生成Dockerfile与Helm Chart

[Buildah](https://buildah.io/)：打包、推送镜像

[Helm](https://helm.sh/)：部署Chart到Kubernetes集群

### 镜像

`beatrueman/builder:1.0`：整合了`draft`与`buildah`，负责镜像构建与镜像推送

`beatrueman/deployer:1.0`：整合了`Helm`，用于部署Chart到Kubernetes

`sonarsource/sonar-scanner-cli:latest`：用于执行代码检查

`jenkins/inbound-agent:3206.vb_15dcf73f6a_9-2`：它是 Jenkins Pipeline 中的一种代理机制，允许在 Jenkins 中动态创建代理节点以执行特定的构建任务。

## 整体流程

![image-20240607222125043](https://gitee.com/beatrueman/images/raw/master/20251208211002012.png)

1. 开发人员推送代码到Git仓库，自动触发Jenkins CI/CD流
2. SonarQube进行代码检查
3. 查找Dockerfile，如果没有则通过Draft自动生成Dockerfile和Helm Chart
4. 使用buildah进行镜像打包与镜像推送到Harbor仓库
5. 使用Helm将Chart部署在Kubernetes集群上，并把打包好的chart包推送至Harbor

## 参数

参数化构建

|变量名|表示值|可选项|
| ------------------| ----------------------------| ---------------|
|HARBOR_REGISTRY|Harbor仓库名||
|PROJECT_NAME|项目名称|必须小写|
|ENTRYPOINT|项目入口文件（仅用于Python）|app.py或main.py|
|PORT|项目暴露入口||
|IMAGE_NAME|镜像名称||
|TAG|镜像标签||
|SONAR_PROJECT_NAME|sonar代码检查项目名称||

## 准备

### 插件下载

在**插件管理**中搜索并下载以下插件

Kubernetes：[Kubernetes版本4238.v41b_3ef14a_5d8](https://plugins.jenkins.io/kubernetes)

SonarQube Scanner for Jenkins：[SonarQube Scanner for Jenkins版本](https://plugins.jenkins.io/sonar)

### 添加凭据

![image-20240607235051117](https://gitee.com/beatrueman/images/raw/master/20251208211002014.png)

1. SonarQube凭据保存的内容为在SonarQube中生成的全局令牌
2. Harbor-Secret凭据保存Harbor的用户名和密码
3. kubeconfig保存最后一步生成的kubeconfig

### Kubernetes集群连接

以使用Kubernerts部署的Jenkins为例（部署方法请自行查询）

![image-20240607233606595](https://gitee.com/beatrueman/images/raw/master/20251208211002015.png)

1.在**系统管理** >> **Clouds**中新增一个cloud

2.主要填入以下配置

- 名称
- Kubernetes地址：`https://<your_ip>:6443`
- Kubernetes命名空间（需要与jenkins部署在同一个命名空间）
- Jenkins地址：Jenkins在K8s部署，填入`http://ClusterIP:Port` `(http://10.96.3.38:8080）`。若不在K8s部署，需要将`/root/.kube/config`base64编码后保存为凭据，然后再填入jenkins的暴露地址。
- Jenkins通道：填入`10.96.1.180:50000`，注意一定不要加http

将cloud名称填入`cloud ""`

![image-20240608001827248](https://gitee.com/beatrueman/images/raw/master/20251208211002016.png)

![20240607-234551](https://gitee.com/beatrueman/images/raw/master/20251208211002017.png)

![image-20240607234620509](https://gitee.com/beatrueman/images/raw/master/20251208211002018.png)

可以点击**连接测试**检查是否可以连接集群

![image-20240607234655491](https://gitee.com/beatrueman/images/raw/master/20251208211002019.png)

### SonarQube准备

1.手工新建一个项目

2.新建一个**全局令牌**

![image-20240607232105268](https://gitee.com/beatrueman/images/raw/master/20251208211002020.png)

3.将该令牌生成的token添加进Jenkins的全局凭据中

4.在**系统配置**中，填入Sonar的服务地址与凭据

![20240607-235555](https://gitee.com/beatrueman/images/raw/master/20251208211002021.png)

### 使用受限的kubeconfig

使用该工具[kubeconfig-generator](https://gitlab.mikumikumi.xyz/base/kubeconfig-generator.git)，生成一个受限制的kubeconfig

1. 新建一个命名空间，用于最终项目的部署
2. 在` kubeconfig-generator.py`中，指定`NAMESPACE` 、`CLUSTER_SERVER`、`SA_NAME`

![image-20240607235927663](https://gitee.com/beatrueman/images/raw/master/20251208211002022.png)

3.将生成的`kubeconfig`下载，以`secret file`形式添加进Jnekins凭据

4.因为此时的config是受限的，需要生成一个`rolebinding`，用于jenkins命名空间下的default用户控制test命名空间下的一些操作

```rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-rolebinding
  namespace: test # 控制test命名空间
subjects:
- kind: ServiceAccount
  name: default
  namespace: jenkins # 这里假设Jenkins服务账户位于jenkins命名空间
roleRef:
  kind: ClusterRole
  name: edit # 或者你可以定义一个自定义的Role，只包含所需的最小权限
  apiGroup: rbac.authorization.k8s.io
```

## 完整Jenkinsfile

```
pipeline {
    agent {
        kubernetes {
            cloud "Redrock-Cloud"
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: jenkins-agent
spec:
  containers:
    - name: sonar-scanner
      image: sonarsource/sonar-scanner-cli:latest
      command:
        - cat
      tty: true
    - name: builder
      image: beatrueman/builder:1.0
      securityContext:
        privileged: true
      command:
        - cat
      tty: true
    - name: deployer
      image: beatrueman/deployer:1.0
      command:
        - cat
      tty: true
    - name: jnlp
      image: jenkins/inbound-agent:3206.vb_15dcf73f6a_9-2
      resources:
        limits:
          memory: "1Gi"
          cpu: "200m"
"""
        }
    }

    // environment {
    //     HARBOR_REGISTRY = ''
    //     PROJECT_NAME = 'myapp' // 项目名称，必须小写
    //     ENTRYPOINT = 'main.py' // 项目入口文件，app.py或main.py
    //     PORT = '6666' // 项目暴露的端口
    //     IMAGE_NAME = 'yicloud' // 镜像名称
    //     TAG = 'v1' // 镜像标签
    //     SONAR_PROJECT_NAME = 'Python' // sonar项目名称
    // }

    stages {
        stage('git clone') {
            steps {
                echo "1.正在克隆代码......"
                git url: ""
            }
        }

        stage('SonarQube code checking') {
            steps {
                container('sonar-scanner') {
                    echo '2.正在进行代码检查......'
                    echo '代码检查结果请在面板查看！'
                    withSonarQubeEnv('SonarQube') {
                        sh '''sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_NAME} \
                            -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                            -Dsonar.sources="/home/jenkins/agent/workspace/${JOB_NAME}" \
                            -Dsonar.projectVersion=1.0 \
                            -Dsonar.sourceEncoding=UTF-8
                        '''
                    }
                }
            }
        }

        stage('Image build') {
            steps {
                script {
                    echo '3.正在制作镜像......'
                    try {
                        def result = sh(script: "ls -al | grep Dockerfile", returnStatus: true)
                        if (result == 0) {
                            echo "找到 Dockerfile，开始构建镜像"
                            container('builder') {
                                sh '''
                                    draft create -a myapp --variable PORT=${PORT} \
                                        --variable APPNAME=${PROJECT_NAME} \
                                        --variable SERVICEPORT=8088 \
                                        --variable NAMESPACE=test \
                                        --variable IMAGENAME=${HARBOR_REGISTRY}/library/${IMAGE_NAME} \
                                        --variable IMAGETAG=${TAG} \
                                        --variable ENTRYPOINT=${ENTRYPOINT} \
                                        --variable VERSION=3 \
                                        --deploy-type helm \
                                        --deployment-only
                                    rm "/home/jenkins/agent/workspace/${JOB_NAME}/charts/templates/namespace.yaml"
                                    buildah bud -t ${HARBOR_REGISTRY}/library/${IMAGE_NAME}:${TAG} .
                                '''
                            }
                        } else {
                            echo "未找到 Dockerfile，将生成 Dockerfile"
                            container('builder') {
                                def jobNameLower = "${JOB_NAME}".toLowerCase()
                                sh '''
                                    draft create -a myapp --variable PORT=${PORT} \
                                        --variable APPNAME=${PROJECT_NAME} \
                                        --variable SERVICEPORT=8088 \
                                        --variable NAMESPACE=test \
                                        --variable IMAGENAME=${HARBOR_REGISTRY}/library/${IMAGE_NAME} \
                                        --variable IMAGETAG=${TAG} \
                                        --variable ENTRYPOINT=${ENTRYPOINT} \
                                        --variable VERSION=3 \
                                        --deploy-type helm
                                    rm "/home/jenkins/agent/workspace/${JOB_NAME}/charts/templates/namespace.yaml"
                                    buildah bud -t ${HARBOR_REGISTRY}/library/${IMAGE_NAME}:${TAG} .
                                '''
                            }
                        }
                    } catch (err) {
                        echo "查找 Dockerfile 发生错误: ${err}"
                    }
                }
            }
        }

        stage('Image push') {
            steps {
                echo "3. Pushing image"
                container('builder') {
                    script {
                        withCredentials([usernamePassword(credentialsId: 'Redrock-Harbor-Secret', passwordVariable: 'passwd', usernameVariable: 'username')]) {
                            sh "buildah login -u ${username} -p ${passwd} ${env.HARBOR_REGISTRY}"
                            sh "buildah images"
                            sh "buildah push ${HARBOR_REGISTRY}/library/${IMAGE_NAME}:${TAG}"
                        }
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "5.即将把服务部署在Kubernetes集群上......"
                container('deployer') {
                    script {
                        // 使用受限制的kubeconfig
                        def kubeConfigCreds = credentials('kubeconfig')
                        // 写入临时kubeconfig文件
                        sh 'echo "${kubeConfigCreds}" | base64 --decode > /tmp/kubeconfig.yaml'
                        sh 'ls'

			            // 部署并获取返回码，如果成功部署则打包chart并上传至Harbor仓库
                        def chartYamlContent = readFile "charts/Chart.yaml" // 读取Chart.yaml
                        def chartYaml = readYaml text: chartYamlContent
                        def chartVersion = chartYaml.version // 获取Chart的version
                        def packageName = "${PROJECT_NAME}-${chartVersion}.tgz" // 打包后的Chart名
                        
                        def helmDeployResult = sh(script: "helm install ${PROJECT_NAME} -n test --kubeconfig /tmp/kubeconfig.yaml charts", returnStatus: true)
			                if(helmDeployResult == 0) {
			                    echo '部署成功！'
			                    echo '正在打包Chart，并上传至Harbor仓库'
			                    
			                    sh "helm package charts"
			                    withCredentials([usernamePassword(credentialsId: 'Redrock-Harbor-Secret', passwordVariable: 'passwd', usernameVariable: 'username')]) {
			                        sh "helm registry login ${env.HARBOR_REGISTRY} -u ${username} -p ${passwd}"
			                        sh "helm push ${packageName} oci://${HARBOR_REGISTRY}/library"
			                    }
			                    
			                } else {
			                    error "Chart部署失败，Helm返回码: ${helmDeployResult}"
			                }
			                
                    }
                }
            }
        }
    }
}

```

![image-20240608002622612](https://gitee.com/beatrueman/images/raw/master/20251208211002023.png)

## 安全警示

根据[buildah image v1.34: Error: open /usr/lib/containers/storage/overlay-images/images.lock: permission denied · Issue #5332 · containers/buildah (github.com)](https://github.com/containers/buildah/issues/5332)

builder容器不得不开启**privileged**，否则无法进行正常的打包与推送镜像。

原因与解决可参考：https://opensource.com/article/19/3/tips-tricks-rootless-buildah

![image-20240608004704783](https://gitee.com/beatrueman/images/raw/master/20251208211002024.png)
