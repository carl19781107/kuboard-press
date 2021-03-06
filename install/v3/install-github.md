---
lessAds: false
description: 一行命令开启Kubernetes多集群管理之路_Kuboard_V3安装
meta:
  - name: keywords
    content: Kubernetes Dashboard安装,Kuboard安装,K8S Dashboard安装
---

# 安装 Kuboard v3 - github

<AdSenseTitle/>


Kuboard 支持多种认证方式：

* 内建用户库
* GitLab Community Edition / GitLab Enterprise Edition / gitlab.com
* GitHub Enterprise / github.com
* LDAP

本文描述了如何配置 Kuboard v3 使其与 github.com / github ee 实现单点登录。

## 前提条件

* 用于安装 Kuboard v3.0 的机器已经安装了 docker，并且版本不低于 docker 19.03
* 您已经有自己的 Kubernetes 集群，并且版本不低于 Kubernetes v1.13
* 您已经安装好了 github ee 或者您打算直接使用 github.com 中的账号

## 部署计划

在正式安装 kuboard v3 之前，需做好一个简单的部署计划的设计，在本例中，各组件之间的连接方式，如下图所示：

* 用户通过 https://github.com 访问 github.com；

* 用户通过 http://kuboard.mycompany.com 访问 Kuboard v3；

* Kuboard 通过 https://github.com 访问 GitHub API；

* 安装在 Kubernetes 中的 Kuboard Agent 通过 `kuboard.mycompany.com` 访问 Kuboard 的 Web 服务端口 80 / 443 和 Kuboard Agent Server 端口 10081。


![image-20201115230700829](./install-github.assets/image-20201115230700829.png)

本例子中，假设：

* 您已经准备好了一个 Linux 服务器用于安装 Kuboard-V3
* 您将要配置 Kuboard 使用 github.com 实现单点登录
  * 也可以支持自己本地安装的 GitHub Enterprise Edition



## 准备 GitHub

* 在 github.com （或者 github ee）中导航到 ***Settings / Developer settings*** 菜单，如下图所示：

  ![image-20201114153558542](./install-github.assets/image-20201114153558542.png)

* 在 ***Settings / Developer settings*** 菜单中导航到 ***OAuth Apps*** 菜单，如下图所示：

  ![image-20201114154213721](./install-github.assets/image-20201114154213721.png)

* 在上图中点击 ***New OAuth App***，并填写如下表单：

  | 字段名称                   | 字段取值                                  | 字段描述                                                     |
  | -------------------------- | ----------------------------------------- | ------------------------------------------------------------ |
  | Application name           | kuboard-v3                                | 标识用，填写任意名称即可                                     |
  | Homepage URL               | http://kuboard.mycompany.com              | 根据 [部署计划](#部署计划)，此处应该填写 http://kuboard.mycompany.com |
  | Application description    | Kuboard v3.0.0                            | 描述，可以为空                                               |
  | Authorization callback URL | http://kuboard.mycompany.com/sso/callback | 根据 [部署计划](#部署计划)，此处应该填写 http://kuboard.mycompany.com/sso/callback，`/sso/callback` 为 Kuboard 中处理 OAuth 回调的 URL 路径 |

  ![image-20201114154856032](./install-github.assets/image-20201114154856032.png)

* 在上图中点击 ***Register application*** 按钮之后，将进入新创建的 OAuth application 的详情页面，在该页面可以获得 ***Client ID*** 和 ***Client Secret*** 两个字段，如下图所示：

  ![image-20201114155605975](./install-github.assets/image-20201114155605975.png)



## 启动 Kuboard 

使用如下命令启动 Kuboard v3：
``` sh
sudo docker run -d \
  --restart=unless-stopped \
  --name=kuboard \
  -p 80:80/tcp \
  -p 10081:10081/udp \
  -p 10081:10081/tcp \
  -v /root/kuboard-data:/data \
  -e KUBOARD_LOGIN_TYPE="github" \
  -e KUBOARD_ENDPOINT="http://kuboard.mycompany.com" \
  -e KUBOARD_ROOT_USER="shaohq" \
  -e GITHUB_CLIENT_ID="17577d45e4de7dad88e0" \
  -e GITHUB_CLIENT_SECRET="ff738553a8c7e9ad39569c8d02c1d85ec19115a7" \
  eipwork/kuboard:v3-alpha
```

::: tip 参数说明
* 建议将此命令保存为一个 shell 脚本，例如 `start-kuboard.sh`，后续升级 Kuboard 或恢复 Kuboard 时，需要通过此命令了解到最初安装 Kuboard 时所使用的参数；
* 第 4 行，Kuboard v3.0 需要暴露 `80` 端口，如安装指令的第三行所示，默认映射到了宿主机的 `80` 端口，您可以根据自己的情况选择宿主机的其他端口；
* 第 5、6 行，Kuboard v3.0 需要暴露 `10081` 端口 TCP / UDP，默认映射到了宿主机的 `10081` 端口，您可以根据自己的情况选择宿主机的其他端口；
* 第 7 行，Kuboard v3.0 的持久化数据存储在 `/data` 目录，默认映射到了宿主机的 `/root/kuboard-data` 路径，请根据您自己的情况进行调整；
* 第 8 行，将 Kuboard v3.0 与 GitHub 进行单点登录集成时，必须指定环境变量 `KUBOARD_LOGIN_TYPE` 为 `github` （适用于 github.com / github-ee）；
* 第 9 行，必须指定 `KUBOARD_ENDPOINT` 环境变量为访问 Kuboard 界面的 URL；（如 [部署计划](#部署计划) 中所描述，本例子中，使用 `http://kuboard.mycompany.com` 作为通过执行此命令启动的 Kuboard 的访问 URL）；此参数不能以 `/` 结尾；
* 第 10 行，必须指定 `KUBOARD_ROOT_USER`，使用该 GitHub 用户登录到 Kuboard 以后，该用户具备 Kuboard 的所有权限；
* 第 11 行，必须指定 `GITHUB_CLIENT_ID`，该参数来自于 [准备 GitHub](#准备-github) 步骤中创建的 GitHub OAuth Application 的 `Client ID` 字段
* 第 12 行，必须指定 `GITHUB_CLIENT_SECRET`，该参数来自于 [准备 GitHub](#准备-github) 步骤中创建的 GitHub OAuth Application 的 `Client Secret` 字段
:::

::: tip GitHub EE
上面的命令行可以用于和 github.com 集成，如果您使用 GitHub Enterprise，需要在命令行中增加如下两个参数：
* `GITHUB_HOSTNAME`，GitHub Enterprise 的 hostname，如
  ```sh
  -e GITHUB_HOSTNAME="github.mycompany.com"
  ```
* `GITHUB_ROOT_CA`，如果您的 GitHub Enterprise 使用了自签名证书，则需要指定 GITHUB_ROOT_CA，否则，可以忽略此参数，如：
  ```sh
  -v /path/to/your/rootca.crt:/github-root-ca.crt
  -e GITHUB_ROOT_CA="/github-root-ca.crt"
  ```
:::


## 访问 Kuboard 界面

* 在浏览器中输入 `http://kuboard.mycompany.com`，您将被重定向到 GitLab 登录界面；
* 在 GitLab 登录界面使用 docker run 命令中 `KUBOARD_ROOT_USER` 参数指定的用户完成登录后，GitLab 将提示您是否授权访问 Kuboard，如下图所示：

  ![image-20201113215827177](./install-gitlab.assets/image-20201113215827177.png)

* 点击上图中的 ***Authorize*** 按钮后，您将成功登录 Kuboard 界面，第一次登录时，界面显示如下所示：

  根据 [部署计划](#部署计划) 的设想，如下表单的填写内容为：

  | 参数名称                                             | 参数值                                                       | 参数说明                                                     |
  | ---------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | <div style="width: 170px;">Kuboard UI 访问地址</div> | <div style="width: 130px;">http://kuboard.mycompany.com</div> | 根据 [部署计划](#部署计划) ，安装在 Kubernetes 中的 Kuboard Agent 通过 http://kuboard.mycompany.com 这个 URL 访问 Kuboard Web 界面，此处为 Kubernetes 集群和 Kuboard 浏览器端用户设定了不同的访问域名（在 docker run 命令的 `KUBOARD_ENDPOINT` 参数指定）。 |
  | Agent Server Host                                    | kuboard.mycompany.com                                        | 根据 [部署计划](#部署计划) ，安装在 Kubernetes 中的 Kuboard Agent 通过 kuboard.mycompany.com 解析到 Kuboard 所在宿主机的 IP 地址 |
  | Agent Server UDP 端口                                | 10081                                                        | 此端口必须与 docker run 命令中映射的 10081/udp 端口一致      |
  | Agent Server TCP 端口                                | 10081                                                        | 此端口必须与 docker run 命令中映射的 10081/tcp 端口一致      |
  
  
  ![image-20201115230236714](install-gitlab.assets/image-20201115230236714.png)
  
* 点击上图中的 ***保存*** 按钮，您将进入 Kuboard 集群列表页，此时，您可以向 Kuboard 添加 Kubernetes 集群，如下图所示：

  ![image-20201113221543277](./install-gitlab.assets/image-20201113221543277.png)

## 授权用户访问 Kuboard

默认情况下，只有 `KUBOARD_ROOT_USER` 参数指定的用户可以执行 Kuboard 中的所有操作，其他用户通过单点登录进入 Kuboard 系统后，除了退出系统，几乎什么事情也做不了。为了让单点登录的用户获得合适的权限，您需要在 Kuboard 中为对应的用户/用户组授权。请参考 [为单点登录的用户/用户组授权](./auth-user-sso.html)
