k8s 二进制升级
二进制学习 https://zhangguanzhang.github.io/page/4/
k8s 1.11.0 升级到1.12.3
1 下载二进制文件 wget https://storage.googleapis.com/kubernetes-release/release/v1.12.1/kubernetes-server-linux-amd64.tar.gz
2.k8s高可用集群平滑升级 v1.11.x 到v1.12.x
 kubectl: https://dl.k8s.io/v1.12.1/kubernetes-client-linux-amd64.tar.gz
 kubernetes-server:  https://dl.k8s.io/v1.12.1/kubernetes-server-linux-amd64.tar.gz
 /opt/k8s/bin目录：kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy
 https://blog.csdn.net/zhonglinzhang/article/details/77223027 k8s 二进制搭建过程
 
 https://v1-12.docs.kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade-1-12/  集群联邦
 
下载地址
https://storage.googleapis.com/kubernetes-release/release/${KUBE_VERSION}/kubernetes-server-linux-amd64.tar.gz > kubernetes-server-linux-amd64.tar.gz

二进制升级到https://blog.51cto.com/goome/2424921   1.12  到1.15

升级指南https://github.com/easzlab/kubeasz/blob/master/docs/op/upgrade.md

https://gitee.com/weilaizhou/K8s 


1.备份kubectl 文件
which kubectl  --替换kubectl
mkdir -pv /opt/k8s/old/client
cp /opt/kube/bin/kubectl kubectl
cp /opt/k8s/kubernetes/client/bin/kubectl /opt/kube/bin
替换线上的客户端 

2.替换线上的二进制的服务端 排除master
systemctl stop kubelet
cp  /opt/k8s/kubernetes/server/bin/kubelet  /opt/kube/bin 重启二进制文件
systemctl start kubelet
systemctl stop kube-proxy.service
cp /opt/k8s/kubernetes/server/bin/kube-proxy /opt/kube/bin/
systemctl start kubelet
systemctl start kube-proxy

matsrer节点执行
cp  /opt/k8s/kubernetes/server/bin/kube-apiserver  /opt/kube/bin
cp  /opt/k8s/kubernetes/server/bin/kube-controller-manager  /opt/kube/bin
cp  /opt/k8s/kubernetes/server/bin/kube-scheduler  /opt/kube/bin

[root@k8s-ceshi-master k8s]# systemctl start kube-apiserver
[root@k8s-ceshi-master k8s]# systemctl start kube-controller-manager.service 
[root@k8s-ceshi-master k8s]# systemctl start kube-scheduler.service 
[root@k8s-ceshi-master k8s]# systemctl start kubelet.service 
[root@k8s-ceshi-master k8s]# systemctl start kube-proxy.service 

查看升级后的状态
kubectl version






