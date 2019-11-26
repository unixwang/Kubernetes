# 安装Kubernetes HA集群
  k8s通过容器的方式，把我们应用的所有内容，也就是应用的所有生命周期全部给管理起来
> sealos一键安装：https://github.com/miaocunfa/iKubernetes/blob/master/Sealos%E5%AE%89%E8%A3%85Kubernetes%20v1.16.0%20HA%E9%9B%86%E7%BE%A4.md
> KubeSphere一键安装：https://www.cnblogs.com/kubesphere/p/11939357.html
> kubeadm一键安装：https://www.cnblogs.com/qianjingchen/diary/2019/03/25/10418567.html
# 初始化脚本 init.sh
``` shell
#!/bin/bash
#!/bin/bash
#desc: this script must run in centos7
#author: by wyy
#
soft_path="/usr/local/src"
Yum_path="/etc/yum.repos.d"
docker_version="docker-ce-18.06.1.ce-3.el7"
k8s_version="kubelet-1.16.0"
kadm_version=="kubeadm-1.16.0"
kubctl_version="kubectl-1.16.0"
kubelet_config="/etc/sysconfig/kubelet"
ssh_pass="wyy123"
master_ip="192.168.56.11"
node_ip1="192.168.56.12"
node_ip2="192.168.56.13"
k8s_version="v1.16.0"

function docker_install() {
    [ -f ${Yum_path}/docker-ce.repo ] || wget -P ${Yum_path} https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    sleep 1
    yum install -y yum-utils device-mapper-persistent-data lvm2
    yum -y install ${docker_version}
    systemctl enable docker && systemctl start docker
    #修改docker Cgroup Driver为systemd
    sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service
    curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io #设置docker镜像,提高docker镜像下载速度和稳定性
    systemctl daemon-reload
    systemctl restart docker

}

function kubernetes_install() {
cat > ${Yum_path}/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
        yum install -y nfs-utils  #必须先安装nfs-utils才能挂载nfs网络存储
	yum install -y ${k8s_version} ${kadm_version} ${kubctl_version} ipvsadm
	systemctl enable kubelet && systemctl start kubelet

}
function kubelet_set() {
	echo 'KUBELET_EXTRA_ARGS="--fail-swap-on=false"' >${kubelet_config}
        /usr/sbin/swapoff -a && sed -i 's/^\/dev\/mapper\/centos-swap/#&/g' /etc/fstab
        #sed -i 's/.*swap.*/#&/' /etc/fstab
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_local_port_range = 10000 65000
fs.file-max = 2000000
vm.swappiness = 0
EOF
        echo "1" > /proc/sys/net/ipv4/ip_forward
 	sysctl --system && sysctl -p /etc/sysctl.d/k8s.conf

}
function sealos_install() {
	wget -P ${soft_path} https://github.com/fanux/sealos/releases/download/v2.0.7/sealos
	chmod +x ${soft_path}/sealos && mv ${soft_path}/sealos /usr/bin/	
}
function init_k8s() {
        sealos init --passwd ${ssh_pass}  --master ${master_ip} --node ${node_ip1} --node ${node_ip2} --pkg-url https://sealyun.oss-cn-beijing.aliyuncs.com/37374d999dbadb788ef0461844a70151-1.16.0/kube1.16.0.tar.gz  --version ${k8s_version}
}
function main() {
	docker_install
        kubernetes_install
	kubelet_set
        sealos_install
        init_k8s
}
main

```
