# 在私有资源上以单机模式安装WeCube

在这里，我们将为您说明如何使用 [Docker :fa-external-link:](https://docs.docker.com/){: target=\_blank} 在您自己的机器资源上安装以单机模式运行的WeCube。

## 先决条件

### 硬件资源

在单机模式下，我们建议您为WeCube的正常运行预留至少 **4核CPU的计算资源**、**16GB的内存资源** 和 **50GB的硬盘存储资源**。

### 系统和软件资源

#### 操作系统

WeCube的安装和运行仅仅依赖于Docker，对操作系统没有其它强制要求。所以，您可以查阅 [此站点 :fa-external-link:](https://docs.docker.com/engine/install/#supported-platforms){: target=\_blank} 来获取Docker所支持的操作系统平台详情。

!!! info "WeCube安装脚本对Linux系统的依赖"
    由于WeCube的安装脚本目前是基于Linux系统的Shell Script制作的，因此非常抱歉，本文中的后续安装步骤只能在Linux系统下执行，我们会在合适的时机补充Windows系统下的安装方式。

#### Docker

您需要安装最新稳定版本的 [Docker Engine :fa-external-link:](https://docs.docker.com/engine/install/){: target=\_blank} 和 [Docker Compose :fa-external-link:](https://docs.docker.com/compose/install/){: target=\_blank}，请参阅此处提供的链接所指向的相关站点获取它们各自的安装信息和指引。

!!! warning "请注意"
    您需要将Docker Engine配置为在运行时监听主机上**非本地回环地址127.0.0.1**的**TCP 2375**端口，因为WeCube将使用此端口调用Docker Engine API来进行插件运行时的管理。您可以参阅 [此站点 :fa-external-link:](https://docs.docker.com/engine/install/linux-postinstall/#configure-where-the-docker-daemon-listens-for-connections){: target=\_blank} 获取如何进行此配置的相关信息和指引。

??? note "如果您使用CentOS，也可以考虑使用这里提供的命令行指令来进行Docker的安装与配置，请展开来查看。"
    但是，我们还是**强烈建议**您从 [Docker官方网站 :fa-external-link:](https://docs.docker.com/engine/install/){: target=\_blank} 获取安装和配置的信息和指引。

    ``` bash
    # 移除已安装的旧版本Docker
    yum remove docker \
               docker-client \
               docker-client-latest \
               docker-common \
               docker-latest \
               docker-latest-logrotate \
               docker-logrotate \
               docker-engine

    # 安装Docker
    yum install -y yum-utils device-mapper-persistent-data lvm2
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    yum makecache fast
    yum install -y docker-ce docker-ce-cli containerd.io
    
    # 安装Docker Compose
    curl -L --fail https://github.com/docker/compose/releases/download/1.25.4/run.sh -o /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose

    # 配置Docker Engine以监听远程API请求
    # 我们在这里启用了腾讯云的Docker Hub镜像为中国大陆境内的访问进行加速，请根据您自己的实际情况进行调整
    mkdir -p /etc/systemd/system/docker.service.d
    cat <<EOF >/etc/systemd/system/docker.service.d/docker-wecube-override.conf
    [Service]
    ExecStart=
    ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375 --registry-mirror=https://mirror.ccs.tencentyun.com
    EOF

    # 启动Docker服务
    systemctl enable docker.service
    systemctl start docker.service

    # 启用IP转发并配置桥接来解决Docker容器对外部网络的通信问题
    cat <<EOF >/etc/sysctl.d/zzz.net-forward-and-bridge-for-docker.conf
    net.ipv4.ip_forward = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    sysctl -p /etc/sysctl.d/zzz.net-forward-and-bridge-for-docker.conf

    ####
    ```

??? note "如果您的主机在访问公共网络时必须启用网络代理，请记得为Docker Engine服务设置正确的启动环境变量。"
    如果您采用上面提供的命令行指令安装Docker，那么您需要在文件 `/etc/systemd/system/docker.service.d/docker-wecube-override.conf` 中添加类似如下的配置内容，具体的配置请与您的网络管理员联系。

    ``` bash
    # 请联系您的网络管理员来确认这些环境变量的配置
    Environment="http_proxy=http://<PROXY_IP>:<PROXY_PORT>"
    Environment="no_proxy=localhost, 127.0.0.1, ::1"
    ```

安装并配置完成后，您可以使用以下命令行指令来确认Docker的运行情况：

``` bash
docker version
docker-compose version
curl http://127.0.0.1:2375/version

```

如果以上指令的版本信息输出一切正常，那么恭喜，我们现在就可以开始安装WeCube了。

## 执行WeCube安装脚本

请执行如下命令行指令：
``` bash
curl -fsSL https://raw.githubusercontent.com/WeBankPartners/wecube-docs/master/get-wecube.sh -o get-wecube.sh && sh get-wecube.sh

```

脚本执行时首先会检查Docker的安装和运行情况，检查通过后将会提示您输入以下安装配置项：

| 配置项名称 | 默认值 | 用途说明 |
| - | - | - |
| install_target_host | *127.0.0.1* | WeCube安装的目标主机名称或IP地址<br/>（**请勿使用此默认值**，详见下方说明。） |
| dest_dir | */data/wecube* | WeCube的安装目录 |
| wecube_version | *v2.1.1* | WeCube安装的目标版本 |
| mysql_password | *WeCube1qazXSW@* | MySQL数据库root账号的密码 |

!!! warning "请注意"

    由于当前版本的WeCube设计，**请勿使用**默认的本地回环地址127.0.0.1作为安装目标主机的IP地址。大部分情况下，您应当使用为主机分配的内网IP地址作为部署配置项`install_target_host`的输入值。

请根据情况提供合适的输入值，如要使用默认值直接回车即可。提供了所有输入值之后，安装脚本将最后再次请您确认以上3个配置项的值，确认后将开始执行WeCube的安装过程。安装脚本执行完毕后，将输出如下内容：

```
WeCube installation completed. Please visit WeCube at http://<您输入的主机名称或IP地址>:19090
```

请依据提示，使用默认的用户名 `umadmin` 和密码 `umadmin` 来访问安装好的WeCube。

## 进一步了解

关于WeCube安装目录结构的详细信息，请参见文档“[WeCube安装目录结构](directory-structure.md)”。
