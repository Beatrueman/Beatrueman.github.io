---
title: wasm-plugin-github action
date: {{ date }}
tags: [CICD]
categories: [Achievements]
---

## 功能描述

功能：添加了利用 GitHub Actions 来自动完成相应的镜像构建和发布工作的Workflow。支持通过push tag和手动触发两种方式。同时也遵循使用oras打包工具。  
特点：完全按照[Wasm 插件镜像规范](https://higress.io/zh-cn/docs/user/wasm-image-spec/)进行镜像打包。

更新：

1. 添加了查找文件的逻辑，会在插件目录中查找`spec.yaml`​、`README.md`​和`README-{lang}.md`​三个文件，有则在打包推送镜像时设置相应的`media type`​。
2. 修改了review后存在的问题。
3. 修改了`builder`​容器的启动命令为 `docker run -itd --name builder xxx /bin/bash`​，之前使用`sleep 99999`​是希望容器保持后台持续运行，这样只适合测试容器，不适合在生产环境中使用。
4. 重新提交`PR`​是因为错误地提交了一些没用的`commit`​，不太会删除，担心破坏仓库并且为了保持`PR`​整洁所以重新提交。

1.准备工作  
添加Repository Secrets和 Repository variables。  
​![img](https://gitee.com/beatrueman/images/raw/master/20251208211058659.png)  
​![img](https://camo.githubusercontent.com/c70982b37d47a9898639e98f34d2db7c51a199e9dacf1b9f8118192c683f1a7f/68747470733a2f2f67697465652e636f6d2f626561747275656d616e2f696d616765732f7261772f6d61737465722f696d672f3230323430363237313230303530352e706e67)  
2.通过`push tag`​触发，先查找相关文件，然后将特定的插件打包成镜像并推送至指定仓库。  
​![image](https://gitee.com/beatrueman/images/raw/master/20251208211103715.png)

![image-20240629120147125](https://gitee.com/beatrueman/images/raw/master/20251208211155646.png)

![image-20240629125144171](https://gitee.com/beatrueman/images/raw/master/20251208211157961.png)  
​![image-20240629125028239](https://gitee.com/beatrueman/images/raw/master/20251208211206989.png)  
3.人工触发，指定插件名和版本号  
​![51e1fc7ab267c66492ecbcfb43cd23d0](https://gitee.com/beatrueman/images/raw/master/20251208211206990.png)![image](https://gitee.com/beatrueman/images/raw/master/20251208211206991.png)

本次贡献借助了通义灵码以及通义千问。  
​![image](https://gitee.com/beatrueman/images/raw/master/20251208211206992.png)  
​![image](https://gitee.com/beatrueman/images/raw/master/20251208211206993.png)  
​![image](https://gitee.com/beatrueman/images/raw/master/20251208211206994.png)  
​![image](https://gitee.com/beatrueman/images/raw/master/20251208211206995.png)

‍

## 验证

1. 纠正了打包容器的`TinyGO`​版本为`0.28.1`​，确保镜像可用。
2. 修改了编译命令为`tinygo build -o ./plugin.wasm -scheduler=none -target=wasi -gc=custom -tags='custommalloc nottinygc_finalizer' ./main.go`​，确保`plugin.wasm`​可用。
3. 修改`push_command`​，最后设置`plugin.tar.gz`​的`media type`​，确保其位于镜像最后一层。

**验证**

环境介绍

- Kubernetes部署Higress，Gateway位于32459端口。
- `higress.yiiong.top`​解析至服务器。
- Kubernetes部署了一个Nginx测试服务，增加一个`hello.html`​，访问会返回`Hello World!`​
- 增加一个`ingress`​

![image-20240708124939848](https://gitee.com/beatrueman/images/raw/master/20251208211206996.png)

![image-20240708124900182](https://gitee.com/beatrueman/images/raw/master/20251208211206997.png)

经`Github Action`​打包出镜像。

![image-20240708124301572](https://gitee.com/beatrueman/images/raw/master/20251208211206998.png)

途径一：通过配置清单部署打包好的`request-block`​插件，拒绝访问`/hello.html`​，可以看到成功被拒绝访问。

![image-20240708125215025](https://gitee.com/beatrueman/images/raw/master/20251208211207000.png)

途径二：通过控制台新增插件，同样可以拒绝访问。

![20240708-125510](https://gitee.com/beatrueman/images/raw/master/20251208211207001.png)

![image-20240708125515152](https://gitee.com/beatrueman/images/raw/master/20251208211207002.png)

![image-20240708125542985](https://gitee.com/beatrueman/images/raw/master/20251208211207003.png)

![image-20240708125640304](https://gitee.com/beatrueman/images/raw/master/20251208211207004.png)

## Github Action

```undefined
name: Build and Push Wasm Plugin Image

on:
  push:
    tags:
    - "wasm-go-*-v*.*.*" # 匹配 wasm-go-{pluginName}-vX.Y.Z 格式的标签
  workflow_dispatch:
    inputs:
      plugin_name:
        description: 'Name of the plugin'
        required: true
        type: string
      version:
        description: 'Version of the plugin (optional, without leading v)'
        required: false
        type: string

jobs:
  build-and-push-image:
    runs-on: ubuntu-latest
    environment:
      name: image-registry-msg
    env:
      IMAGE_REGISTRY_SERVICE: ${{ vars.IMAGE_REGISTRY_SERVICE || 'higress-registry.cn-hangzhou.cr.aliyuncs.com' }}
      IMAGE_REPOSITORY: ${{ vars.IMAGE_REPOSITORY || 'plugins' }}
      GO_VERSION: 1.19
      TINYGO_VERSION: 0.28.1
      ORAS_VERSION: 1.0.0
    steps:
      - name: Set plugin_name and version from inputs or ref_name
        id: set_vars
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            plugin_name="${{ github.event.inputs.plugin_name }}"
            version="${{ github.event.inputs.version }}"
          else
            ref_name=${{ github.ref_name }}
            plugin_name=${ref_name#*-*-} # 删除插件名前面的字段(wasm-go-)
            plugin_name=${plugin_name%-*} # 删除插件名后面的字段(-vX.Y.Z)
            version=$(echo "$ref_name" | awk -F'v' '{print $2}')
          fi
          echo "PLUGIN_NAME=$plugin_name" >> $GITHUB_ENV
          echo "VERSION=$version" >> $GITHUB_ENV
      - name: Checkout code
        uses: actions/checkout@v3

      - name: File Check
        run: | 
          workspace=${{ github.workspace }}/plugins/wasm-go/extensions/${PLUGIN_NAME}
          push_command="./plugin.tar.gz:application/vnd.oci.image.layer.v1.tar+gzip"
          # 查找spec.yaml
          if [ -f "${workspace}/spec.yaml" ]; then
            echo "spec.yaml exists"
            push_command="./spec.yaml:application/vnd.module.wasm.spec.v1+yaml $push_command "
          fi
          # 查找README.md
          if [ -f "${workspace}/README.md" ];then
              echo "README.md exists"
              push_command="./README.md:application/vnd.module.wasm.doc.v1+markdown $push_command "
          fi
        
          # 查找README_{lang}.md
          for file in ${workspace}/README_*.md; do
            if [ -f "$file" ]; then
              file_name=$(basename $file)
              echo "$file_name exists"
              lang=$(basename $file | sed 's/README_//; s/.md//')
              push_command="./$file_name:application/vnd.module.wasm.doc.v1.$lang+markdown $push_command "
            fi
          done
          echo "PUSH_COMMAND=\"$push_command\"" >> $GITHUB_ENV
      
      - name: Run a wasm-go-builder
        env: 
          PLUGIN_NAME: ${{ env.PLUGIN_NAME }}
          BUILDER_IMAGE: higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/wasm-go-builder:go${{ env.GO_VERSION }}-tinygo${{ env.TINYGO_VERSION }}-oras${{ env.ORAS_VERSION }}
        run: |
          docker run -itd --name builder -v ${{ github.workspace }}:/workspace -e PLUGIN_NAME=${{ env.PLUGIN_NAME }} --rm ${{ env.BUILDER_IMAGE }} /bin/bash
      - name: Build Image and Push
        run: |
        
          push_command=${{ env.PUSH_COMMAND }}
          push_command=${push_command#\"}
          push_command=${push_command%\"} # 删除PUSH_COMMAND中的双引号，确保oras push正常解析
          command="
          cd /workspace/plugins/wasm-go/extensions/${PLUGIN_NAME}
          go mod tidy
          tinygo build -o ./plugin.wasm -scheduler=none -target=wasi -gc=custom -tags='custommalloc nottinygc_finalizer' ./main.go
          tar czvf plugin.tar.gz plugin.wasm
          echo ${{ secrets.REGISTRY_PASSWORD }} | oras login -u ${{ secrets.REGISTRY_USERNAME }} --password-stdin ${{ env.IMAGE_REGISTRY_SERVICE }}
          oras push ${IMAGE_REGISTRY_SERVICE}/${IMAGE_REPOSITORY}:${VERSION} ${push_command}
          "
          docker exec builder bash -c "$command"
```
