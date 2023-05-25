## k8s部署

### 准备

1）在控制节点做免密认证

```shell
#!/bin/bash
ssh-keygen -t rsa  -C "test key" -P ''
for IP in  { 10 11 20 21 22 30 31  32 40 41 42 50 51 };do
        sshpass -p 123456 ssh-copy-id   -o StrictHostKeyChecking=no   12.0.0.$IP
done

```

2）配置ansible

apt update && apt install -y  ansible

root@master01:/etc/ansible# cat hosts  
[slb]
slb01 ansible_host=12.0.0.10
slb02 ansible_host=12.0.0.11

[master]
master01 ansible_host=12.0.0.20
master02 ansible_host=12.0.0.21
master03 ansible_host=12.0.0.22

[node]
node01 ansible_host=12.0.0.30
node02 ansible_host=12.0.0.31
node03 ansible_host=12.0.0.32

[etcd]
etcd01 ansible_host=12.0.0.40
etcd02 ansible_host=12.0.0.41
etcd03 ansible_host=12.0.0.42

[harbor]
harbor01 ansible_host=12.0.0.50
harbor02 ansible_host=12.0.0.51

ansible-inventory --list -y 可以使用此命令检查主机列表

3）各个主机之间做hosts解析

ansible  all  -m copy  -a     "src=/etc/hosts dest=/etc/hosts mode=0644"

4）时区及其时间的修改

```yaml
root@master01:~/initialize/playbook# cat timeconfig.yml 
---
- hosts: all
  gather_facts: no
  tasks:
  - name: Set timezone
    ansible.builtin.file:
      src:  /usr/share/zoneinfo/Asia/Shanghai
      dest: /etc/localtime
      state: link
  - name: 24-hour system 
    lineinfile:
      path: /etc/default/locale 
      regexp: '^LC_TIME'
      line: LC_TIME=en_DK.UTF-8
      #匹配以LC_TIME开头的行，若不存在则在文本最后一行添加

```

5）时间同步服务

*/5 * * * * /usr/sbin/ntpdate time1.aliyun.com &> /dev/null && hwclock -w &> /dev/null     #提供NTP server端配置

```yaml
# cat chrony.yaml
---
- hosts: all
  gather_facts: no
  tasks:
  - name: Install chrony
    ansible.builtin.apt:
      name: chrony
      state: latest 
      update_cache: no 
  - name: delete  line include centos.pool
    lineinfile:
      path: /etc/chrony/chrony.conf
      regexp: 'ubuntu.pool'
      state: absent
  - name: add  line
    lineinfile:
      path: /etc/chrony/chrony.conf
      insertafter: 'ntp.ubuntu.com'
      state: present
      line: 'pool 12.0.0.20             iburst maxsources 1'
  - name:
    ansible.builtin.systemd:
      name: chrony.service
      state: restarted
      enabled: yes

```

6）harbor安装

1.主机harbor01，通过 APT Docker 存储库安装 docker（apt-cache madison 可以列出指定的版本）。 参阅清华大学镜像官网

2.安装docker-compose  

3.安装harbor及其配置 参见官网

```shell

1)生成CA证书,如果使用 FQDN 连接您的 Harbor 主机，则必须将其指定为公用名 ( CN) 属性。
生成CA证书的私钥。
openssl genrsa -out ca.key 4096
生成CA证书
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/CN=ks.harbor.com" \
 -key ca.key \
 -out ca.crt
 
2)生成服务器证书
生成私钥。
openssl genrsa -out ks.harbor.com.key 4096
生成证书签名请求 (CSR)。如果使用 FQDN 连接您的 Harbor 主机，则必须将其指定为公用名称 ( CN) 属性并在密钥和 CSR 文件名中使用它。

openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/CN=ks.harbor.com" \
    -key ks.harbor.com.key \
    -out ks.harbor.com.csr
生成 x509 v3 扩展文件。无论是使用 FQDN 还是 IP 地址连接到您的 Harbor 主机，都必须创建此文件，以便为你的 Harbor 主机生成符合主题备用名称 (SAN) 和 x509 v3 的证书扩展要求。替换DNS条目以反映你的域。

cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=ks.harbor.com
DNS.2=ks.harbor
DNS.3=harbor01
EOF
使用该v3.ext文件为您的 Harbor 主机生成证书。

将yourdomain.comCRS 和 CRT 文件名中的 替换为 Harbor 主机名。

openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in ks.harbor.com.csr \
    -out ks.harbor.com.crt

```

```yaml
4)harbor的主配置文件的修改
hostname: ks.harbor.com
http:
  port: 80
https:
  port: 443
  certificate: /apps/harbor/certs/ks.harbor.com.crt
  private_key: /apps/harbor/certs/ks.harbor.com.key
harbor_admin_password: 123456
5)安装harbor

```

7）测试harbor的使用

在harbor02的主机上测试，容器运行为containerd，安装参见https://0xzx.com/2022091003502621716.html

若要使得容器具有网络功能可以使用cni提供的插件https://github.com/containernetworking/plugins（参见附录）

containerd（containerd是一个实现了CRI接口的容器运行时）与harbor服务的对接以及配置文件的修改根据需要请参考https://github.com/containerd/containerd/blob/main/docs/cri/config.md#registry-configuration

https://github.com/containerd/containerd/blob/main/docs/cri/config.md

```shell
root@harbor02:/etc/containerd/certs.d/ks.harbor.com# cat hosts.toml 
server = "https://ks.harbor.com"

[host."https://ks.harbor.com"]
  ca = "/etc/certs/ks.harbor.com.crt"     #此处的证书是从harbor服务证书的拷贝
#ansible控制端需要执行（因为k8s的每台工作节点都需要认证到harbor服务器上）
root@master01:~/palybooks# cat crt-file.yaml 
---
- hosts: all
  gather_facts: no
  tasks:
  - name: mkdir directory1 
    file:
      path: /etc/containerd/certs.d/ks.harbor.com
      state: directory
      mode: 755
  - name:  mkdir directory2
    file:
      path: /etc/certs
      state: directory
      mode: 755
  - name: copy file1
    copy:
      src: /root/palybooks/hosts.toml
      dest: /etc/containerd/certs.d/ks.harbor.com/hosts.toml
      mode: '0644'
  - name: copy file2
    copy:
      src: /root/palybooks/ks.harbor.com.crt
      dest: /etc/certs/ks.harbor.com.crt
      mode: '0644'

```

containerd自带的ctr客户端默认是不使用配置config.toml文件，当我们从私有仓库pull或者push的时候，需要做下列指定

ctr -n k8s.io  images push    --platform  linux/amd64     --user admin:123456  --hosts-dir "/etc/containerd/certs.d"  ks.harbor.com/basic/busybox:latest

8）配置Api-server的负载均衡器(slb01和slb02),各种客户端程序通过vip访问api-server，利用keepalived提供的虚拟VIP地址，通过haproxy bind这个vip，就可以通过vip代理至后端的api-server

```shell
root@slb01:~# apt-get   install     keepalived   haproxy    （slb02同样需要安装）
root@slb01:/etc/keepalived# cp   /usr/share/doc/keepalived/samples/keepalived.conf.vrrp   /etc/keepalived/
root@slb01:/etc/keepalived# mv  keepalived.conf.vrrp keepalived.conf
root@slb01:/etc/keepalived# cat keepalived.conf 
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    garp_master_delay 10
    smtp_alert
    virtual_router_id 51
    priority 100                  #区分主备的优先级
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        12.0.0.188/24 dev  ens33 label ens33:0
        12.0.0.189/24 dev  ens33 label ens33:1
        12.0.0.190/24 dev  ens33 label ens33:2
        12.0.0.191/24 dev  ens33 label ens33:3
    }
}
slb01和slb02同样的配置,但是没有vip监听的地址，需要修改内核参数,允许非本机的端口绑定
# cat /etc/sysctl.conf
net.ipv4.ip_nonlocal_bind = 1
net.ipv6.ip_nonlocal_bind = 1

root@slb01:~# cat /etc/haproxy/haproxy.cfg 
.....
listen  k8s-6443
        bind      12.0.0.188:6443
        mode      tcp
        server    k8s1 12.0.0.20:6443 check inter 3s fall 3 rise 5
        server    k8s2 12.0.0.21:6443 check inter 3s fall 3 rise 5
        server    k8s3 12.0.0.22:6443 check inter 3s fall 3 rise 5
root@slb01:~# systemctl   restart haproxy.service                                                        
root@slb01:~# systemctl   enable  haproxy.service 


```

### K8S部署

https://github.com/easzlab/kubeasz

主要的配置文件修改如下

```shell
oot@master01:/etc/kubeasz/clusters/k8s-01# grep -v  ^#  hosts 
[etcd]
12.0.0.40
12.0.0.41
12.0.0.42

[kube_master]
12.0.0.20
12.0.0.21
12.0.0.22

[kube_node]
12.0.0.30
12.0.0.31
12.0.0.32

[harbor]

[ex_lb]
12.0.0.11 LB_ROLE=backup EX_APISERVER_VIP=12.0.0.188 EX_APISERVER_PORT=6443
12.0.0.10 LB_ROLE=master EX_APISERVER_VIP=12.0.0.188 EX_APISERVER_PORT=6443

[chrony]

[all:vars]
SECURE_PORT="6443"

CONTAINER_RUNTIME="containerd"

CLUSTER_NETWORK="calico"

PROXY_MODE="ipvs"

SERVICE_CIDR="10.100.0.0/16"

CLUSTER_CIDR="10.200.0.0/16"

NODE_PORT_RANGE="30000-65000"

CLUSTER_DNS_DOMAIN="wukui.local"

bin_dir="/usr/local/bin/"

base_dir="/etc/kubeasz"

cluster_dir="{{ base_dir }}/clusters/k8s-01"

ca_dir="/etc/kubernetes/ssl"

config.yml配置根据自己的需要进行修改
```

之后的步骤按照项目的步骤进行安装就可以

```shell
#名词解释
CRI是一个实现了kubelet与各种容器运行时对接的接口，无需重新编译集群组件，从containerd1.1开始cri插件内置于二进制文件中
OCI(Open Container Initiative 开放容器倡议）开放容器倡议是一个开放的治理结构，想要表达的目标是围绕容器格式和容器运行时创建开放的行业标准也就是将一个符合容器镜像规范的镜像运行在特殊的文件系统中，这个文件系统满足统一的规范，用来解包和运行镜像。kubernetes推出的runC就是这个文件系统（容器运行时规范）的一种实现，而要与runC进行对接（对容器运行时API的调用）就需要通过容器运行时接口（CRI)来实现，只要实现了CRI接口的容器运行时就可以对接到k8s的kubelet组件中
Containerd或者dockerd是用来维持通过runc创建的容器的运行状态。即runc用来创建和运行容器，containerd作为常驻进程管理容器的生命周期
低级容器运行时：实现名称空间的隔离以及容器的创建和运行，例如runC
高级容器运行时:用来管理由低级容器运行时创建的容器，例如dockerd和containerd,我们可以理解为高级容器运行时提供了一个GRPC Server,与kubelet进行对接
其中我们可以在高级容器运行时与runC之前增加一个shim,允许runc在创建和运行容器之后退出, 并将shim作为容器的父进程, 而不是containerd或者dockerd作为父进程
#说明
containerd的容器日志的位置是由kubelet来决定的（例如/var/log/pods/default_my-nginx_7a3a9db7-d9e2-43fd-8e59-c4f44d494685/my-nginx)
#命令行客户端，containerd安装好之后，命令行客户端ctr二进制也会被下载
参考：https://cloud.tencent.com/developer/article/1852346
```

### DNS的部署

yaml部署清单可以在ks8对应的二进制文件的附件中找到，根据自己的需要修改其中的变量配置

(在k8s项目下的对应版本的CHANGELOG下，下载二进制文件，解压后的addons中有各种清单)

（其中在coredns.yaml的清单中修改CLUSTER_DNS_DOMAIN="wukui.local"）

其中镜像地址可以修改，但是coredns.yaml文件中serviceaccount在拉取docker镜像时候可能会需要用户名和密码报权限禁止的问题

ctr  -n  k8s.io images   push  --platform  linux/amd64   --user native900:wukui@731++   docker.io/native900/coredns:v1.9.3

### 附录

```shell
# - 将busybox容器实现网络通信
root@slb02:/opt/cni/bin/cni-1.1.2/scripts# ctr -n k8s.io run -d docker.io/library/busybox:latest busybox   （创建一个容器并生成task）
root@slb02:/opt/cni/bin/cni-1.1.2/scripts# ctr -n k8s.io task ls
TASK       PID     STATUS    
busybox    1428    RUNNING
root@slb02:/opt/cni/bin/cni-1.1.2/scripts# pid=$(ctr -n k8s.io t ls|grep busybox|awk '{print $2}')
root@slb02:/opt/cni/bin/cni-1.1.2/scripts# netnspath=/proc/$pid/ns/net
root@slb02:/opt/cni/bin/cni-1.1.2/scripts# CNI_PATH=/root/cni-plugins ./exec-plugins.sh add $pid $netnspath

随后进入busybox容器我们将会发现其新增了一张网卡并可以实现外部网络访问：
root@slb02:/opt/cni/bin/cni-1.1.2/scripts# ctr -n k8s.io task exec --exec-id $RANDOM -t busybox  sh -
/ # ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 3a:1b:7b:9f:58:90 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.6/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::381b:7bff:fe9f:5890/64 scope link 
       valid_lft forever preferred_lft forever
/ # ping -c2  223.5.5.5
PING 223.5.5.5 (223.5.5.5): 56 data bytes
64 bytes from 223.5.5.5: seq=0 ttl=127 time=11.494 ms
64 bytes from 223.5.5.5: seq=1 ttl=127 time=16.847 ms

--- 223.5.5.5 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 11.494/14.170/16.847 ms

接下来以这种方式创建的所有容器都可以通过mynet0桥实现互通，并且分配的地址为10.0.0.0/24
```

我们知道containerd的客户端ctr很难用，containerd仓库下还有一个项目nerdctl，根据提示进行使用

```shell
当我们在运行容器后,直接使用了cni-plugins的桥接网络
root@slb02:~/cni-plugins# nerdctl --cni-path   /root/cni-plugins  run -it --rm alpine    
/ # ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 92:9c:d7:92:94:66 brd ff:ff:ff:ff:ff:ff
    inet 10.4.0.3/24 brd 10.4.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::909c:d7ff:fe92:9466/64 scope link 
       valid_lft forever preferred_lft forever
```

使用独立版的buildkit来构建镜像（官方项目下都有明确说明）

```shell
1）下载二进制文件，解压到指定目录下
2）利用systemd来管理buildkit服务端
root@slb02:/lib/systemd/system# cat   /lib/systemd/system/buildkit.service 
[Unit]
Description=BuildKit
Requires=buildkit.socket
After=buildkit.socket
Documentation=https://github.com/moby/buildkit

[Service]
Type=notify
ExecStart=/usr/local/bin/buildkitd --oci-worker=false --containerd-worker=true

[Install]
WantedBy=multi-user.target
root@slb02:/lib/systemd/system# cat   /lib/systemd/system/buildkit.socket 
[Unit]
Description=BuildKit
Documentation=https://github.com/moby/buildkit

[Socket]
ListenStream=%t/buildkit/buildkitd.sock
SocketMode=0660

[Install]
WantedBy=sockets.target
systemctl  enable  --now buildkit.service
```

之后nerdctl的使用基本与docker的客户端一致，更高级的用法参加官方

### k8s升级

#### 二进制升级

root@master01:/etc/kube-lb/conf# cat kube-lb.conf 
user root;
worker_processes 1;

error_log  /etc/kube-lb/logs/error.log warn;

events {
    worker_connections  3000;
}

stream {
    upstream backend {
        server 12.0.0.31:6443    max_fails=2 fail_timeout=3s;
        server 12.0.0.32:6443    max_fails=2 fail_timeout=3s;
        server 12.0.0.33:6443    max_fails=2 fail_timeout=3s;
    }

```json
server {
    listen 127.0.0.1:6443;
    #有一个代理程序NGINX监听在本地的6443端口，kublet和kube-proxy,会代理到后端的upstream,实现Node节点上对api-server的高可用
    proxy_connect_timeout 1s;
    proxy_pass backend;
}
```
}

先从每一个Node节点上把nginx代理的其中的一个matser删除，重启NGINX，再升级master的配置

node升级的话需要停kubelet及其kube-proxy,替换二进制

1）注释upstream的其中一个api-server地址，重启systemctl   restart kube-lb.service 

2）master升级

root@master01:/etc/kubeasz/bin# kube  #这些都需要升级，升级之前需要停止服务
kube-apiserver           kube-controller-manager  kubectl                  kubelet                  kube-proxy               kube-scheduler 

root@master01:/etc/kubeasz# systemctl stop  kube
kube-apiserver.service                                            kubelet.service                                                   kubepods-burstable.slice                                          kube-scheduler.service
kube-controller-manager.service                                   kubepods-besteffort.slice                                         kubepods.slice                                                    
kube-lb.service                                                   kubepods-burstable-pod398d5e6b_17f3_413f_bac5_c3462b86c1e3.slice  kube-proxy.service                                                

下载二进制文件解压（client, node,server,kubernetes),解压之后各种二进制文件都在kubernets/server/bin,将此目录下的二进制文件拷贝到:/usr/local/bin/（部署的时候可以设定）

3）node升级

停止服务

scp kubernets/server/bin 下的Kubelet kube-proxy 到/usr/local/bin/

## Etcd的备份与恢复

io要求高

write ahead log 在执行真正的写操作之前先预写一个日志，记录数据的变化历程

root@etcd01:~# ls /var/lib/etcd/                         数据目录   可通过/etc/systemd/system/etcd.service修改
member                        

root@etcd01:~# ETCDCTL_API=3 etcdctl   get / --prefix   --keys-only       |grep pods     |wc -l  
8

root@etcd01:~# ETCDCTL_API=3 etcdctl     snapshot  save   etcd-2022-07-19.db

root@etcd01:~# ETCDCTL_API=3 etcdctl     snapshot  restore    etcd-2022-07-19.db    --data-dir=/tmp/etcd

DATE=`date +%Y-%m-%d-%H-%M-%S`

在恢复的时候可以停止相关的服务，对其etcd的写入
