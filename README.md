# Kubernetes-ansible

## 全部重构,步骤和kubeadm一样,九成九配置文件和路径一样,管理组件扣成了systemd二进制,并且强制bind网卡支持多网卡部署

## 更新
 * 2018/06/06 - 解决keepalived无法添加某些段的vip的错误
 * 2018/06/08 - 解决2.5.3的ansible使用脚本报错
 * 2018/06/09 - 添加增加node剧本
 * 2019/03/08 - 推倒重来,仿照kubeadm,但是支持多网卡下部署,etcd支持单独的机器跑

## ansible部署Kubernetes

系统可采用`Ubuntu 16.x`与`CentOS 7.x`
本次安裝的版本：
> * Kubernetes v1.13.4 (HA高可用)
> * CNI plugins v0.7.1
> * Etcd v3.3.12
> * Calico v3.0.4
> * Docker CE 18.06.03


**hosts文件写ip来支持多网卡部署**

**因为kubeadm扣的步骤,路径九成九一致,剧本里路径不给自定义**

**HA是基于VIP,无法用于云上(测试过青云默认解除ip和mac绑定可以飘VIP,阿里不行),阿里四层SLB也有问题**

安装过程是参考的[Kubernetes v1.13.4 HA全手动苦工安装教学](https://zhangguanzhang.github.io/2019/03/03/kubernetes-1-13-4)

**下面是我的配置,电脑配置低就一个Node节点**

| IP    | Hostname   |  CPU  |   Memory | 
| :----- |  :----:  | :----:  |  :----:  |
| 192.168.126.6 |K8S-M1|  2   |   2G    |
| 192.168.126.7 |K8S-M2|  2   |   2G    |
| 192.168.126.8 |K8S-M3|  2   |   2G    |
| 192.168.126.9 |K8S-N1|  2   |   2G    |

# 使用前提和注意事项
> * 每台主机端口和密码最好一致(不一致最好懂点ansible修改hosts文件)


# 使用(在master1的主机上使用且master1安装了ansible,剧本也可以支持非部署的机器运行剧本,后续测试下)

centos通过yum安装ansible或者pip
```
#可以yum安装指定版本或者离线安装 yum install -y  https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.7.8-1.el7.ans.noarch.rpm
yum install -y epel-release && yum install -y python-pip git sshpass
pip install ansible
# 嫌弃慢就下面的
# pip install --no-cache-dir ansible -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
```
#上面安装提示失败的话请先下载下来用yum解决依赖
```
yum install wget -y 1 > /dev/null
wget https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.7.8-1.el7.ans.noarch.rpm
yum localinstall ansible-2.5.4-1.el7.ans.noarch.rpm -y
```

**1 git clone**
```
git clone https://github.com/zhangguanzhang/Kubernetes-ansible.git
cd Kubernetes-ansible
```

**2 配置脚本属性**

 * 修改当前目录ansible的`hosts`分组成员文件

 * 修改`group_vars/all.yml`里面的参数
 1. ansible_ssh_pass为ssh密码(如果每台主机密码不一致请注释掉`all.yml`里的`ansible_ssh_pass`后按照的`hosts`文件里的注释那样写上每台主机的密码）
 2. `VIP`为高可用HA的虚ip,和master在同一个网段没有被使用过的ip即可,`NETMASK`为VIP的掩码
 3. `INTERFACE_NAME`为各机器的ip所在网卡名字Centos可能是`ens33`,看情况修改
 4. 其余的参数按需修改,不熟悉最好别乱改
 5. 运行下`ansible all -m ping`测试连通性
----------

**3 开始运行安装，下面是用法**
 * -----   setup.yml     -------
 * setup: 机器设置(关闭swap安装一些依赖+ntp)+内核升级并重启,例如`ansible-playbook setup.yml`,带上`-e 'kernel=false'`不会升级内核
 * -----    deploy.yml   -------
 * docker: 安装docker,默认是18.06,其他版本自行修改`group_vars/all.yml`, 运行命令为`ansible-playbook deploy.yml --tags docker`
 * 这步不是标签,手动运行`bash get-binaries.sh all`: 通过docker下载k8s和etcd的二进制文件还有cni插件,觉得不信任可以自己其他方式下载,cni插件可能不好下载
 * tls: 生成证书和管理组件的kubeconfig,kubeconfig生成依赖kubectl命令,此步确保已经下载有.运行命令为`ansible-playbook deploy.yml --tags tls`,下面的也是tag改tags后面标签为下面的即可
 * etcd: 部署etcd
 * HA: keepalived+haproxy
 * master: 管理组件
 * bootstrap: 给kubelet注册用
 * node: kubelet
 * addon: kube-proxy,flannel,coredns.metrics-server，flannel镜像拉取慢,推荐运行前使用命令拉取`ansible Allnode -m shell -a 'curl -s https://zhangguanzhang.github.io/bash/pull.sh | bash -s -- quay.io/coreos/flannel:v0.11.0-amd64'`

运行方式为下面,xxx为上面标签
ansible-playbook deploy.yml --tags xxx  
运行到master后查看下管理组件状态
```
$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
```

运行完node后`kubectl get node -o wide`查看注册上来否
运行完addon后查看node状态是否为ready,以及kube-system命名空间下pod是否运行,

![k8s](https://github.com/zhangguanzhang/Image-Hosting/blob/master/k8s/kube-ansible.png)

**5 后续添加Node节点(未完成)**
 1. 在当前的ansible目录改hosts,添加[newNode]分组写上成员
 2. 后执行以下命令添加node
 3. 然后查看是否添加上
```
$ kubectl get node -o wide
NAME     STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
k8s-m1   Ready    <none>   27h   v1.13.4   172.16.1.4    <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
k8s-m2   Ready    <none>   27h   v1.13.4   172.16.1.5    <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
k8s-m3   Ready    <none>   27h   v1.13.4   172.16.1.8    <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
k8s-n1   Ready    <none>   27h   v1.13.4   172.16.1.12   <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
k8s-n2   Ready    <none>   27h   v1.13.4   172.16.1.13   <none>        CentOS Linux 7 (Core)   5.0.0-2.el7.elrepo.x86_64   docker://18.6.3
```

不想master跑可以按照输出的命令打污点
```
kubectl taint nodes ${node_name} node-role.kubernetes.io/master="":NoSchedule
```
master的`ROLES`字段默认是none,它显示的值是来源于一个label
```
kubectl label node ${node_name} node-role.kubernetes.io/master=""
kubectl label node ${node_name} node-role.kubernetes.io/worker=worker

```
后面的一些Extraaddon后续更新
