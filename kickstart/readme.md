# 1. 架构说明

## 1.1 组件说明

| 组件 | 作用 | 协议/端口 |
|------|------|-----------|
| **dnsmasq** | DHCP（提供 PXE 参数，分配 IP）+ TFTP 服务 | UDP 67（DHCP）、UDP 69（TFTP） |
| **nginx** | 提供 HTTP 服务：存放 iPXE 脚本、内核、initrd、Kickstart 文件 | TCP 8080 |
| **iPXE** | 网络引导固件（`undionly.kpxe` 用于 BIOS，`ipxe.efi` 用于 UEFI） | TFTP 下载引导文件，HTTP 下载脚本/内核 |
| **Fedora 安装源** | 使用中科大镜像站（外网），同步到本地仓库 | HTTPS |

## 1.2 工作流程

1. 客户端 PXE 启动 → 广播 DHCP 请求。
2. dnsmasq（DHCP）响应，根据客户端架构返回对应 iPXE 引导文件（BIOS 用 `undionly.kpxe`，UEFI 用 `ipxe.efi`）。
3. 客户端下载并执行引导文件 → 进入 iPXE 环境。
4. iPXE 重新发起 DHCP 请求（带 `user-class=iPXE`）。
5. dnsmasq 识别到 iPXE 客户端，返回 HTTP 脚本 URL。
6. iPXE 执行 HTTP 脚本，从 nginx 下载 `vmlinuz` 和 `initrd.img`，并传递 `inst.repo`、`inst.ks` 等参数。
7. 内核启动，读取 Kickstart 文件，从镜像源自动安装 Fedora。

---

# 2. 服务器准备（Fedora Server）

| 项目 | 值 |
|------|-----|
| 服务器操作系统 | Fedora 44 Server |
| 路由器/网关 | 192.168.1.5 |
| PXE 服务器 IP | 192.168.1.23 |

## 2.1 服务器配置

```bash
# selinux会组织nginx读取文件，为了方便关闭 fedora 默认开启的 selinux
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
sudo reboot now

# 防火墙放行
sudo firewall-cmd --permanent --add-service=http --add-service=tftp --add-service=dhcp
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

## 2.2 安装软件包

```bash
sudo dnf makecache
sudo dnf install -y dnsmasq nginx tftp
```

## 2.3 创建所需目录

```bash
sudo mkdir -p /srv/tftp          # TFTP 根目录
sudo mkdir -p /var/www/html/autoinstall/fedora-mirror   # HTTP 内核/initrd 存放目录
sudo mkdir -p /var/www/html/autoinstall/ks         # Kickstart 存放目录
sudo mkdir -p /var/www/html/autoinstall/ipxe       # HTTP iPXE 脚本目录
```

---

# 3. 准备文件

| 目录 | 文件 |
|------|-----|
| /srv/tftp | ipxe固件 |
| /var/www/html/autoinstall/fedora-mirror/ | fedora镜像源 |
| /var/www/html/autoinstall/ipxe  | ipxe脚本文件 |
| /var/www/html/autoinstall/ks  | kickstart配置文件 |

## 3.1 下载 iPXE 引导文件

下载地址：https://github.com/ipxe/ipxe/releases （下载 `ipxeboot.tar.gz`）

```bash
cd /srv/tftp
sudo wget https://github.com/ipxe/ipxe/releases/download/v2.0.0/ipxeboot.tar.gz
sudo tar -xvf ipxeboot.tar.gz
sudo rm ipxeboot.tar.gz
```

## 3.2 同步镜像源

```bash
sudo rsync -avzP --delete \
    rsync://mirrors.ustc.edu.cn/fedora/releases/44/Server/x86_64/os/ \
    /var/www/html/autoinstall/fedora-mirror/
```

## 3.3 创建 iPXE 启动脚本 `boot.ipxe`

```bash
sudo nano /var/www/html/autoinstall/ipxe/boot.ipxe
```

内容如下（根据实际修改 IP 和文件位置）：

```bash
#!ipxe

set server-ip http://192.168.1.23:8080
set fedora-mirror-url ${server-ip}/fedora-mirror

kernel ${fedora-mirror-url}/images/pxeboot/vmlinuz \
    inst.repo=${fedora-mirror-url} \
    inst.ks=${server-ip}/ks/leza.ks \
    inst.stage2=${fedora-mirror-url}

initrd ${fedora-mirror-url}/images/pxeboot/initrd.img
boot
```

## 3.4 创建 Kickstart 文件 `leza.ks`

可以从已安装的 Fedora 系统获取模板：`sudo cat /root/anaconda-ks.cfg`

```bash
sudo nano /var/www/html/autoinstall/ks/leza.ks
```

参考内容（密码哈希需另行生成）：

```bash
# 安装源：使用本地同步的镜像源
url --url="http://192.168.1.23:8080/fedora-mirror"

# 文本模式安装（避免图形界面等待）
text

# 键盘布局
keyboard --vckeymap=us --xlayouts='us'

# 系统语言
lang en_GB.UTF-8

# 网络配置：DHCP 自动获取（无人值守必须自动）
network --bootproto=dhcp --device=link --activate

# 时区
timezone Asia/Shanghai --utc

# 磁盘分区（自动分区，并且清除所有现有分区）
zerombr
clearpart --all --initlabel
autopart --type=lvm

# 用户与密码
rootpw --lock
user --groups=wheel --name="用户名" --password="密码" --iscrypted --gecos="用户名"

# 授权配置（保留指纹选项）
authselect enable-feature with-fingerprint

# 安装后重启
reboot

# 首次启动时是否运行设置代理（关闭可彻底无人）
firstboot --disable

# 要安装的软件包组
%packages
@^server-product-environment
nano
wget
%end

# 后置脚本（举例：添加 SSH 公钥）
%post --log=/root/kickstart-post.log
mkdir -p /home/leza/.ssh
echo "公钥" > /home/leza/.ssh/authorized_keys
chown -R leza:leza /home/leza/.ssh
chmod 700 /home/leza/.ssh
chmod 600 /home/leza/.ssh/authorized_keys
%end
```

生成密码哈希：

```bash
mkpasswd -m yescrypt "密码"
```

# 4 配置 nginx

## 4.1 设置文件权限

```bash
sudo chown -R nginx:nginx \
    /var/www/html/autoinstall/fedora/44 \
    /var/www/html/autoinstall/ks \
    /var/www/html/autoinstall/ipxe
```
## 4.2 创建所需目录

```bash
sudo mkdir /etc/nginx/sites-available
sudo mkdir /etc/nginx/sites-enabled

sudo nano /etc/nginx/nginx.conf
# 在 include /etc/nginx/conf.d/*.conf; 下一行
# 添加 include /etc/nginx/sites-enabled/*;
```

## 4.3 添加站点配置

```bash
sudo vi /etc/nginx/sites-available/autoinstall-server.conf
```

配置内容：

```bash
server {
    listen 192.168.1.23:8080;
    server_name _;

    root /var/www/html/autoinstall;

    charset utf-8;

    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
}
```

## 4.4 软链接到 sites-enabled

```bash
cd /etc/nginx/sites-enabled
sudo ln -s /etc/nginx/sites-available/autoinstall-server.conf autoinstall-server.conf
```

## 4.5 应用配置

```bash
sudo nginx -t
sudo nginx -s reload
```

## 4.6 验证

```bash
curl http://192.168.1.23:8080/ipxe/boot.ipxe
```

---

# 5. 配置 dnsmasq（DHCP + TFTP）

## 5.1 创建配置文件

创建 `/etc/dnsmasq.d/proxy-pxe.conf`：

```bash
sudo nano /etc/dnsmasq.d/proxy-pxe.conf
```

## 5.2 配置文件内容

```bash
# 监听网卡（替换为实际网卡名）
interface=ens18
bind-dynamic

dhcp-range=192.168.1.190,192.168.1.200,255.255.255.0,12h
dhcp-option=option:router,192.168.1.5
dhcp-option=option:dns-server,192.168.1.5,8.8.8.8

# 启用 TFTP 服务
enable-tftp
tftp-root=/srv/tftp

# ------------------------------------------------------------
# 第一阶段：为原生 PXE 客户端（BIOS/UEFI）提供 iPXE 引导文件
# ------------------------------------------------------------
# BIOS (Arch:00000)
dhcp-match=set:bios,option:client-arch,0
dhcp-boot=tag:bios,ipxeboot/undionly.kpxe

# UEFI x86-64 (Arch:00007 和 00009)
dhcp-match=set:efi64,option:client-arch,7
dhcp-match=set:efi64,option:client-arch,9
dhcp-boot=tag:efi64,ipxeboot/ipxe.efi

# ------------------------------------------------------------
# 第二阶段：为已经是 iPXE 的客户端提供 HTTP 脚本
# ------------------------------------------------------------
# 识别 iPXE 客户端（User-Class 包含 "iPXE"）
dhcp-userclass=set:ipxe,iPXE
# 直接返回 HTTP URL（跳过 TFTP 脚本传输）
dhcp-boot=tag:ipxe,http://192.168.1.23:8080/ipxe/boot.ipxe

# 日志（调试用）
log-dhcp
```

## 5.3 `dhcp-boot` 语法说明

```bash
dhcp-boot=[<filename>],[<tftp_server>],[<client_arch>],[<bootserver_hostname>]
```

| 参数位置 | 参数名 | 是否必需 | 说明 |
|---------|--------|---------|------|
| 第1个 | `filename` | **推荐** | 引导文件名（如 `pxelinux.0`、`ipxe.efi`） |
| 第2个 | `tftp_server` | 可选 | TFTP 服务器地址（IP 或主机名），不写则用 dnsmasq 自己的地址 |
| 第3个 | `client_arch` | 可选 | 客户端架构代码，用于匹配特定类型的 PXE 客户端 |
| 第4个 | `bootserver_hostname` | 可选 | 引导服务器主机名（极少使用） |

> ⚠️ **注意**：dhcp 代理模式下，iPXE 会用主 DHCP 覆盖 filename 和 next-server，忽略代理配置。

## 5.4 启动服务并验证

```bash
# 测试配置
sudo dnsmasq --test

# 启动服务
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

## 5.5 验证 TFTP

```bash
curl tftp://192.168.1.23/ipxeboot/undionly.kpxe -o undionly.kpxe
curl tftp://192.168.1.23/ipxeboot/ipxe.efi -o ipxe.efi
```