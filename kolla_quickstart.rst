================================
利用 Kolla 部署 OpenStack 云平台
================================

Kolla 是 OpenStack 大帐篷模式下的官方子项目之一，其主要目标是通过利用 Docker 容器以及 Ansible 自动化部署工具，来为 OpenStack 云平台提
供一个简单而灵活的部署和管理的方式。它允许完全的自主化，使得用户可以根据自身的特殊需求自行修改任一 OpenStack 服务的配置项。


宿主机配置建议
--------------

- 2 个或以上网卡
- 至少 8GB 内存
- 至少 40GB 磁盘空间

(注：本文以 CentOS 7.2, OpenStack Liberty 为例进行说明)

这里需要解释的是，在 OpenStack 中，宿主机上不同网卡的作用一般可归为如下几种：

- eth0  默认的第一块网卡，可作为管理网络使用，即登录管理本宿主机，或不同宿主机间通信。
- eth1  可作为数据网络(VLAN/VXLAN)使用，不同虚拟机之间进行通信时（东西向流量），经过此物理网卡。
- eth2  可作为外部网络使用，即当虚拟机需要与外网进行通信时（南北向流量），经过此物理网卡。
- eth3  可作为存储网络使用，即当虚拟机与后端存储设备进行交互时，经过此物理网卡。

当然，理论上如果只使用一块网卡的话也是可以的，只是将网络流量这样细分开来后，可分担物理网卡的压力，也便于后期的问题定位。
根据 Kolla 的配置项解释，此处我们暂时使用两块网卡：

- eth0  192.168.0.10 作为管理网络 + 数据网络 + 存储网络，用于 OpenStack 集群内部通信
- eth1  172.16.0.10 作为外部网络，用于虚拟机与外部通信


相关依赖资源的安装
------------------

请确保 ``pip`` 包管理器已经安装，并升级到了最新版本。
使用 ``pip`` 进行装的好处在于，安装包的版本可以被指定。

::

    yum -y install epel-release
    yum -y install python-pip
    pip install -U pip


在进行 OpenStack Liberty 发行本进行部署的时候，Kolla 对于以下组件有版本的特定要求：

==============  ============  ============
组件            最小版本号    最大版本号
==============  ============  ============
Ansible         1.9.4         < 2.0.0 
Docker          1.10.0        无
Docker Python   1.6.0         无
Python Jinja2   2.6.0         无
==============  ============  ============

由于 Docker 是构建镜像以及运行容器的必要工具，因此建议从 Docker 官方安装已经打包好的稳定版本。但值得注意的是，Kolla 的发布并不是和 
Docker 同步的，因此当前测试过的 Docker 版本为 1.10.0 以上的版本。通过运行如下命令，系统会通过从 Docker 页面上拉取安装和配置 Docker 
的脚本，并于本地执行安装。

::

    # 安装 Docker
    curl -sSL https://get.docker.io | bash
	
    # 查看 Docker 版本
    docker --version

	
当通过 systemd 运行 Docker 时，在 ``/usr/lib/systemd/system/docker.service`` 文件中的 MountFlags 选项参数需要修改，由 ``slave`` 改为 
``shared``，以确保 ``kolla-ansible`` 可以正常将 OpenStack 各服务部署成功。在对文件进行修改后，为使 Docker 配置生效，需重启 Docker 进
程：

::

    systemctl daemon-reload
    systemctl restart docker
	
	
在 OpenStack CLI/Python 代码运行的系统上，推荐安装 OpenStack Python 客户端。这样，在所在系统的控制台就可以使用 OpenStack 命令行工具
了。不过，在安装 OpenStack Python 客户端之前，一些相关的依赖资源也需要被安装，以便能够构建客户端的代码：

::

    # 相关依赖资源
    yum install -y python-devel libffi-devel openssl-devel gcc git
    
    # 安装 OpenStack Python 客户端
    pip install python-openstackclient
    
    
为了使用 Kolla 相关工具来部署 OpenStack，可使用如下命令来安装，并将相关配置文件复制到 ``/etc/`` 下：

::

    # 克隆 Kolla 代码仓库
    git clone https://git.openstack.org/openstack/kolla
    
    # 切换至 stable/liberty 分支
    git checkout remotes/origin/stable/liberty
    
    # 安装 Kolla 工具及依赖
    pip install kolla/
    
    # 复制配置文件
    cp -r kolla/etc/kolla /etc/


在很多操作系统上，Libvirt 是默认启动的，这个组件是用于管理虚拟机的，但是使用 Kolla 部署 OpenStack，Libvirt 会在容器中运行来直接管理
虚拟机，因此宿主机中的 Libvirt 需要被关闭，以避免冲突：

::

    systemctl stop libvirtd.service
    systemctl disable libvirtd.service
    
    
Ansible 是 Kolla 部署 OpenStack 的主要工具，根据版本要求，这里推荐安装 Ansible 1.9.4：

::

    pip install ansible==1.9.4
    
    
构建容器镜像
------------

对于容器镜像的构建，首先需要生成一个 Kolla 构建容器镜像的配置文件，这个可以通过 ``tox`` 来生成。请确保执行命令时，所在路径为 ``kolla/`` 
下的主目录内：

::

    pip install tox
    tox -e genconfig
    

所生成的配置文件为 ``kolla/etc/kolla/-build.conf``，可以将其复制到 ``/etc/kolla/`` 下。其中一些配置项需要特别指出，可参考如下参数：

::

    # 基于 CentOS 操作系统
    base = centos
	
    # 使用源码安装	
    install_type = source
	
    # 镜像标签 1.1.1 （Liberty）	
    tag = 1.1.1
    
    
若对特定服务的源有定制化需求，也可自行修改相应服务配置，以通过定制化源进行镜像构建。

构建镜像可直接使用如下命令：

::

    kolla-build
    
    
由于需要构建全部所需镜像，因此构建镜像需要花费一些时间，并且很有可能出现一些镜像构建失败的情况。对于构建失败的镜像，可以单独对其进
行重新构建，无需重建所有镜像：

::

    # 以 neutron 为例，重新构建 neutron 相关镜像
    kolla-build neutron
    
    # 更多的参数也可参考帮助提示
    kolla-build -h
    
    
部署仓库注册服务器
----------------------

在镜像全部构建好后，部署一个本地的仓库注册服务器可以极大地方便后续镜像的重复使用，以避免反复从 Docker Hub 上拉取镜像而花费不必要的时间。
对于使用的仓库注册服务器镜像，建议使用 2.3 或更新的版本，通过如下命令运行：

::

    docker run -d -p 4000:5000 --restart=always --name kollaglue registry:2
	
    # 获取更多运行 docker 容器帮助
    docker help run
	
	
在开启了本地仓库注册服务器之后，还需在 ``/usr/lib/systemd/system/docker.service`` 文件中添加本地仓库注册服务器通信端口地址，使得 Docker 
可以和本地仓库注册服务器进行通信。本地仓库注册服务器端口地址的格式为 ``IP_OF_THE_MACHINE:PORT``，例如：172.16.0.100:4000，则参照如下修改：

::

    ExecStart=/usr/bin/docker daemon -H fd:// --insecure-registry 172.16.0.100:4000
	
	
通过重新启动 Docker 进程，加载所修改配置项：

::

    systemctl daemon-reload
    systemctl restart docker
	
	
通过如下命令，可查看本地已经构建的 Docker 镜像，并通过标签（TAG）列确认所构建镜像为 1.1.1 版：

::

    docker images
	
	
通过以下组合命令，可将所构建的全部镜像一次性打上标签，并推到仓库注册服务器中：

::

    # 若仓库注册服务器端口地址为 172.16.0.100:4000，对镜像打标签
    docker images | awk '{print $1}' | xargs -t -i docker tag {}:1.1.1 172.16.0.100:4000/{}:1.1.1
	
    # 将镜像推入仓库注册服务器
    docker images | grep 172.16.0.100:4000 | awk '{print $1}' | xargs -t -i docker push {}:1.1.1
	
	
使用 Kolla 部署 OpenStack
-------------------------

Kolla 提供了两种部署方式，单节点部署和多节点部署。单节点部署，即把所有的 OpenStack 服务都部署到一个物理节点；而多节点部署则是根据需要，将
不同的 OpenStack 服务部署到不同的节点中。本文着重介绍单节点部署的情景。

在进行部署前，``/etc/kolla/globals.yml`` 和 ``/etc/kolla/passwords.yml`` 两个文件需要根据需要进行配置调整。``/etc/kolla/globals.yml`` 文件
中集合了部署所需的一些参数变量，而 ``/etc/kolla/passwords.yml`` 文件中则是一些密码项。初始状况下，``/etc/kolla/passwords.yml`` 中所有密码
项都是空的，因此可通过以下命令自动生成随机密码，并将其填充到该文件中：

::

    kolla-genpwd
	
	
一些在 ``/etc/kolla/globals.yml`` 中需要修改的参数如下：

::

    # 使用源码安装部署
    kolla_install_type: "source"
	
    # OpenStack 发行版
    openstack_release: "1.1.1"
	
    # VIP，应与宿主机首块网卡地址在同一网段且未被占用 IP 地址
    kolla_internal_vip_address: "192.168.0.200"
	
    # Docker 仓库注册服务器
    docker_registry: "172.16.0.100:4000"
	
    # Docker 镜像仓库命名空间
    docker_namespace: "kollaglue"
	
    # 首块网卡
    network_interace: "eth0"
	
    # 有互联网连接的网卡
    neutron_external_interface: "eth1"
	
	
上述参数是需要注意的，其他参数可根据需求自行调整。

在配置文件按需求更改完毕后，使用如下命令检查所要部署的目标端口是否都可用：

::

    kolla-ansible prechecks
	
	
当镜像存于远程仓库时，可通过如下命令确认所需镜像是否都已准备好：

::

    kolla-ansible pull
	
	
若上述两步均无问题，运行如下命令开始进行部署：

::

    kolla-ansible deploy
	
	
部署过程大致会持续 20 分钟左右，主要依据部署环境而定。在成功部署后，使用如下命令，可创建出 openrc 文件，默认存放在 
``/etc/kolla/admin-openrc.sh`` 中。通过加载 openrc 中的环境变量，可在后台使用 OpenStack 命令行工具；或直接访问 OpenStack 控制面板。

::

    # 生成 openrc 文件
    kolla-ansible post-deploy
	
    # 加载环境变量
    source /etc/kolla/openrc.sh
	
	
