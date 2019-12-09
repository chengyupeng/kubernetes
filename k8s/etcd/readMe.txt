etcd3.4部署
下载wget https://github.com/etcd-io/etcd/releases/download/v3.4.1/etcd-v3.4.1-linux-amd64.tar.gz
tar -zxvf etcd-v3.4.1-linux-amd64.tar.gz -C /usr/local/
解压 替换二进制文件即可


etcd集群搭建
环境初始化工作
环境准备 修改hostname /etc/hostname  /etc/sysconfig/network && hostnamectl set-hostname  etcd机器做好时间同步 配置hosts解析
vi /etc/hosts
禁用selinux sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux sed -i 's/enforcing/disabled/g' /etc/selinux/config
关闭swap  注释掉/etc/fstab 中关于swap 相关的行sed -i 's/\/dev\/mapper\/centos-swap/#\/dev\/mapper\/centos-swap/g' /etc/fstab
关闭防火墙 systemctl stop firewalld && systemctl disable firewalld
开启forward
iptables -P FORWARD ACCEPT
cat >> /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
EOF
加载系统参数
sysctl --system


磁盘参数
https://linux.die.net/man/1/ionice  


1.关闭kube-apiserver.service   systemctl stop kube-apiserver.service
2.查看kube-apiserver.service   systemctl status kube-apiserver.service
3.备份etcd节点
export ETCDCTL_API=3
etcdctl snapshot save /opt/etcddb/20191103sna.db
4.暂停etcd节点
systemctl stop etcd
5.查看etcd配置文件
systemctl status etcd
6.编辑/etc/systemd/system/etcd.service
--force-new-cluster  //将集群转变为单节点
7.集群搭建 .备份单节点文件
1.备份数据目录
cp -r /var/lib/etcd/etcd1 /opt/etcd/20191204data
etcdctl snapshot save /opt/etcddb/20191103sna.db
2.vim  /etc/systemd/system/etcd.service
ExecStart=/opt/kube/bin/etcd \
  --name=master \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls=https://172.16.18.2:2380 \
  --listen-peer-urls=https://172.16.18.2:2380 \
  --listen-client-urls=https://172.16.18.2:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://172.16.18.2:2379 \
  --initial-cluster-token=etcd-cluster-0 \  //备注多个集群 token 唯一
  --initial-cluster=master=https://172.16.18.2:2380,node3=https://172.16.18.19:2380 \
  --initial-advertise-peer-urls=https://172.16.18.2:2380 \
  --auto-compaction-mode=periodic \  //https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/configuration.md#configuration-flags 使用推荐的配置 
  --initial-cluster-state=existing \
  --auto-compaction-mode=periodic \
  --data-dir=/var/lib/etcd/etcd1 \
  --max-snapshots=5 \
  --max-wals=5 \
  --max-txn-ops=128 \
  --heartbeat-interval=100 \ //集群心跳 https://github.com/etcd-io/etcd/blob/master/Documentation/tuning.md 官方推荐  具体参见 http://172.16.18.2:3000/d/6q1lE0AWk/etcd-by-prometheus?orgId=1  指标https://github.com/etcd-io/etcd/blob/master/Documentation/metrics.md
  --election-timeout=500 \
  --auto-compaction-retention=12 \    //自动合并时间
  --max-request-bytes=33554432 \     //https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/limit.md
  --quota-backend-bytes=8589934592 \ //https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/limit.md
  --snapshot-count=50000 //默认值是100000  https://github.com/etcd-io/etcd/blob/master/Documentation/tuning.md   如果etcd的内存使用量和磁盘使用量过高，请尝试通过在命令行上设置以下内容来降低快照阈值 默认情况下，每10,000次更改后将创建快照
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

3.清空数据目录rm -rf /var/lib/etcd/etcd1
4.各个节点上执行  vim restore.sh 
#! /bin/bash
 ETCDCTL_API=3 etcdctl --endpoints=https://172.16.18.2:2379  --cacert=/etc/kubernetes/ssl/ca.pem  --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem  snapshot restore /opt/sh/back.db  --name=master  --initial-advertise-peer-urls=https://172.16.18.2:2380 --initial-cluster-token=etcd-cluster-0  --initial-cluster=master=https://172.16.18.2:2380,node3=https://172.16.18.19:2380  --data-dir=/var/lib/etcd/etcd1
 5.恢复etcd集群 ./restore.sh
 6.启动etcd集群
 7.启动apiserver
 8.验证etcd集群部署
 ETCDCTL_API=3 etcdctl --endpoints=https://172.16.18.2:2379  --cacert=/etc/kubernetes/ssl/ca.pem  --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem endpoint status -w table
 9.查看etcd集群中的数据
 ETCDCTL_API=3 etcdctl --endpoints=https://172.16.18.2:2379,https://172.16.18.19:2379  --cacert=/etc/kubernetes/ssl/ca.pem  --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem get   --keys-only=true --prefix /registry/pod
 
 


