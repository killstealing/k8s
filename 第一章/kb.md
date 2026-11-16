# Kubernetes 集群搭建与问题排查全记录

> 记录从零开始搭建 K8s 集群（含 containerd、Calico、MySQL、Tomcat、可视化工具）过程中遇到的所有问题及解决方案。

---

## 目录

1. [Ubuntu 上是否需要 yum](#1-ubuntu-上是否需要-yum)
2. [安装 kubeadm / kubelet / kubectl](#2-安装-kubeadm--kubelet--kubectl)
3. [切换到 root 用户](#3-切换到-root-用户)
4. [containerd 版本过旧导致 CRI 不可用](#4-containerd-版本过旧导致-cri-不可用)
5. [containerd 升级与配置修复](#5-containerd-升级与配置修复)
6. [kubeadm init preflight 卡在拉镜像](#6-kubeadm-init-preflight-卡在拉镜像)
7. [init-config.yaml 的 YAML 语法问题](#7-init-configyaml-的-yaml-语法问题)
8. [kubelet 无法启动控制平面（pause 镜像问题）](#8-kubelet-无法启动控制平面pause-镜像问题)
9. [kubectl 报 localhost:8080 拒绝连接](#9-kubectl-报-localhost8080-拒绝连接)
10. [worker 节点是否需要复制 admin.conf](#10-worker-节点是否需要复制-adminconf)
11. [节点 NotReady 与 Calico CNI 安装](#11-节点-notready-与-calico-cni-安装)
12. [Calico 镜像拉取失败的镜像源替换](#12-calico-镜像拉取失败的镜像源替换)
13. [kubectl apply 报 localhost:8080 拒绝连接](#13-kubectl-apply-报-localhost8080-拒绝连接)
14. [查看 MySQL Pod 日志](#14-查看-mysql-pod-日志)
15. [新版 K8s 用 docker ps 的替代命令](#15-新版-k8s-用-docker-ps-的替代命令)
16. [pause 容器为什么看不到](#16-pause-容器为什么看不到)
17. [Tomcat 镜像拉取失败（kubeguide/tomcat-app:v1）](#17-tomcat-镜像拉取失败kubeguidetomcat-appv1)
18. [NodePort 访问不通排查](#18-nodeport-访问不通排查)
19. [MySQL JDBC 连接失败（驱动与 MySQL 8 不兼容）](#19-mysql-jdbc-连接失败驱动与-mysql-8-不兼容)
20. [KubeSphere 简介与搭建方式](#20-kubesphere-简介与搭建方式)
21. [Kubernetes Dashboard 与 Headlamp](#21-kubernetes-dashboard-与-headlamp)
22. [HPA / PVC / PV 概念解释](#22-hpa--pvc--pv-概念解释)
23. [busybox 镜像使用注意](#23-busybox-镜像使用注意)
24. [Namespace + Pod 的部署方式](#24-namespace--pod-的部署方式)
25. [pause 容器在 Headlamp/KubeSphere 中搜不到](#25-pause-容器在-headlampkubesphere-中搜不到)
26. [外部机器连不上 MySQL Service](#26-外部机器连不上-mysql-service)
27. [Deployment apiVersion 写错](#27-deployment-apiversion-写错)
28. [Tomcat Deployment 一直不 Ready](#28-tomcat-deployment-一直不-ready)
29. [删除并重建 Deployment](#29-删除并重建-deployment)
30. [Tomcat 镜像 latest 标签 not found](#30-tomcat-镜像-latest-标签-not-found)

---

## 1. Ubuntu 上是否需要 yum

**结论：不需要，也不建议。**

- Ubuntu 属于 Debian 系，原生包管理器是 `apt`（处理 `.deb`）。
- `yum` 是为 Red Hat/CentOS 设计的（处理 `.rpm`）。
- 强行安装 yum 会遇到兼容性问题，甚至影响系统稳定性。

安装 K8s 组件应通过 `apt` 添加 Kubernetes 官方源来完成。

---

## 2. 安装 kubeadm / kubelet / kubectl

**推荐使用阿里云镜像源**（官方 `pkgs.k8s.io` 国内访问不稳定）。

```bash
# 创建密钥环目录
sudo mkdir -p /etc/apt/keyrings

# 下载并添加阿里云 GPG 密钥（以 v1.28 为例）
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 添加软件源
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# 安装
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 锁定版本
sudo apt-mark hold kubelet kubeadm kubectl
安装前必备：

禁用 Swap

安装容器运行时（containerd）

加载 br_netfilter、overlay 模块，开启 IP 转发

常见报错：Unable to locate package kubelet
→ 说明源没生效，需检查 kubernetes.list 内容并重新 apt-get update。

3. 切换到 root 用户
方式	命令	说明
推荐	sudo -i	用当前用户密码提权，加载 root 环境
需 root 密码	su -	Ubuntu 默认未设置 root 密码，需先 sudo passwd root
单条命令	sudo <命令>	不切换用户，仅提权执行
4. containerd 版本过旧导致 CRI 不可用
报错：

text
validate service connection: ... rpc error: code = Unimplemented
desc = unknown service runtime.v1.RuntimeService
根因：containerd 版本为 1.2.6，不支持 Kubernetes 所需的 CRI v1 接口（需 ≥ 1.5）。

解决：升级 containerd 到 1.6.x 或更高。

5. containerd 升级与配置修复
bash
# 添加 Docker 官方源（国内建议阿里云镜像）
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y containerd.io

# 重新生成配置
systemctl stop containerd
mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
containerd config default > /etc/containerd/config.toml

# 关键配置
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sed -i 's|sandbox_image = ".*"|sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"|' /etc/containerd/config.toml

systemctl daemon-reload
systemctl restart containerd

# 验证
crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock info
ctr plugin ls | grep cri   # 应显示 io.containerd.grpc.v1.cri 状态 ok
注意：apt 安装的 containerd 默认配置常带 disabled_plugins = ["cri"]，必须清空。

6. kubeadm init preflight 卡在拉镜像
原因：命令未加 --config，kubeadm 回退到默认 registry.k8s.io 源。

解决：所有 kubeadm 子命令都需显式指定配置文件：

bash
kubeadm init phase preflight --config=init-config.yaml
kubeadm config images pull --config=init-config.yaml
强制使用本地镜像：在 ClusterConfiguration 下加：

yaml
imagePullPolicy: Never
7. init-config.yaml 的 YAML 语法问题
报错：

text
key "imageRepository" already set in map
mapping values are not allowed in this context
原因：

imageRepository 字段重复出现。

某行写成 imageRepository: imageRepository: xxx，一行两个冒号。

修复：

yaml
imageRepository: registry.aliyuncs.com/google_containers
验证语法：

bash
kubeadm config print init-defaults --config=init-config.yaml > /dev/null
8. kubelet 无法启动控制平面（pause 镜像问题）
现象：

text
Failed to create sandbox for pod ... failed to pull image "registry.k8s.io/pause:3.6"
dial tcp ... i/o timeout
根因：containerd 的 sandbox_image 仍指向 registry.k8s.io/pause:3.6，国内拉不到。

修复：

bash
sed -i 's|sandbox_image = ".*"|sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"|' /etc/containerd/config.toml
systemctl restart containerd
crictl images | grep pause   # 确认本地有 pause:3.9
然后 kubeadm reset -f 再重新 kubeadm init。

9. kubectl 报 localhost:8080 拒绝连接
报错：

text
The connection to the server localhost:8080 was refused
根因：kubectl 找不到 kubeconfig，回退到默认 localhost:8080。

解决（在控制平面节点上）：

bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
10. worker 节点是否需要复制 admin.conf
结论：不需要，且不应这么做。

/etc/kubernetes/admin.conf 只存在于控制平面节点。

该文件拥有集群最高管理员权限，复制到 worker 节点有安全风险。

集群管理操作应在控制平面节点执行，worker 只需保证 kubelet 和 containerd 正常运行。

11. 节点 NotReady 与 Calico CNI 安装
现象：kubectl get nodes 显示两个节点均 NotReady。

根因：未安装网络插件（CNI），kubelet 报 cni plugin not initialized。

解决：安装 Calico（或 Flannel）：

bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
安装后等待 Calico Pod 全部 Running，节点会转为 Ready。

12. Calico 镜像拉取失败的镜像源替换
报错：

text
failed to pull image "docker.io/calico/cni:v3.25.0": dial tcp ... i/o timeout
失败过的镜像源：

docker.io（Docker Hub，国内不通）

docker.chenby.cn（域名已失效）

docker.xuanyuan.me / docker.m.daocloud.io（403 / 服务终止）

可用方案：华为云 SWR 同步站

bash
crictl pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/cni:v3.25.0
批量替换 calico.yaml 中的镜像地址：

bash
# 若原地址是 docker.io/calico
sed -i 's|docker.io/calico|swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico|g' calico.yaml

# 若原地址是 docker.chenby.cn/calico
sed -i 's|docker.chenby.cn/calico|swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico|g' calico.yaml

grep "image:" calico.yaml   # 确认替换结果
kubectl apply -f calico.yaml
注意：calico-node 是 DaemonSet，每个节点都需要能拉到镜像。worker 节点若网络不同，需单独 crictl pull 验证，或采用离线导入。

13. kubectl apply 报 localhost:8080 拒绝连接
原因：当前机器（如 worker 节点 ubuntu）没有 kubeconfig。

解决：

推荐：回到控制平面节点执行 kubectl apply。

若必须在 worker 上执行，从控制平面节点复制配置：

bash
mkdir -p $HOME/.kube
scp root@192.168.3.147:/etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
14. 查看 MySQL Pod 日志
报错：

text
container "mysql" in pod "mysql-xxx" is waiting to start: trying and failing to pull image
说明：容器未启动，没有日志可看。需先解决镜像拉取问题。

正常查看日志：

bash
kubectl get pods                       # 找到 Pod 名称
kubectl logs <pod-name>                # 一次性输出
kubectl logs -f <pod-name>             # 实时跟踪
kubectl logs --tail=100 <pod-name>     # 只看最后 100 行
kubectl logs <pod-name> --previous     # 看崩溃前日志
kubectl logs <pod-name> -n <ns>        # 指定命名空间
Pod 一直 Pending / 拉镜像失败：用 kubectl describe pod <pod-name> 看 Events。

15. 新版 K8s 用 docker ps 的替代命令
背景：K8s v1.24 起移除 dockershim，容器运行时为 containerd，docker 命令不可用。

书上命令	替代命令	说明
docker ps | grep mysql	crictl ps | grep mysql	节点级，操作 containerd
docker ps	kubectl get pods | grep mysql	集群级，推荐日常使用
docker logs	kubectl logs <pod>	查看 Pod 日志
docker inspect	kubectl describe pod <pod>	查看 Pod 详情
原则：作为集群使用者，优先用 kubectl；只有需要直接操作节点容器运行时才用 crictl。

16. pause 容器为什么看不到
结论：pause 容器没有消失，只是 crictl ps 默认不显示它。

pause 容器（sandbox 容器）由 kubelet 通过 containerd 创建，用于占住 Pod 的网络命名空间。

crictl ps 默认只显示业务容器，过滤掉 pause。

查看方式：

bash
ctr -n k8s.io containers ls | grep pause
crictl images | grep pause
17. Tomcat 镜像拉取失败（kubeguide/tomcat-app:v1）
报错：

text
Failed to pull image "kubeguide/tomcat-app:v1": dial tcp ... i/o timeout
失败源：

docker.xuanyuan.run/kubeguide/tomcat-app:v1 → 401 Unauthorized（需登录）

可用源（华为云 SWR）：

bash
crictl pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/kubeguide/tomcat-app:v1
修改 YAML：

yaml
image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/kubeguide/tomcat-app:v1
注意：worker 节点也需能拉到该镜像，否则对应 Pod 仍会 ImagePullBackOff。

18. NodePort 访问不通排查
现象：curl http://127.0.0.1:30001/demo/ 卡住无响应。

排查步骤：

本机 curl 测试：区分防火墙问题 vs K8s 转发问题。

检查 iptables 规则：

bash
sudo iptables -t nat -L KUBE-NODEPORTS -nv | grep 30001
有规则说明 kube-proxy 已生成入口。

检查 Endpoints：

bash
kubectl get endpoints myweb
若为 <none>，说明 Service selector 与 Pod labels 不匹配。

检查 Calico IPIP 模式（虚拟机环境常见问题）：

bash
kubectl get ippool -o yaml | grep ipipMode
若为 Always，同局域网节点可改为 Never：

bash
kubectl patch ippool default-ipv4-ippool --type=merge --patch '{"spec":{"ipipMode":"Never"}}'
防火墙：ufw 或 iptables 是否放行 NodePort 端口。

应用路径：老镜像可能没有 /demo/ 路径，先试根路径 /。

19. MySQL JDBC 连接失败（驱动与 MySQL 8 不兼容）
报错：

text
com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException:
Could not create connection to database server.
根因：老 Tomcat 镜像内置的 JDBC 驱动不认识 MySQL 8 的 caching_sha2_password 认证插件。

方案一（推荐）：MySQL 降级到 5.7

yaml
image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/library/mysql:5.7
方案二：修改 MySQL 8 用户认证方式

bash
kubectl exec -it <mysql-pod> -- mysql -uroot -p123456
sql
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
FLUSH PRIVILEGES;
20. KubeSphere 简介与搭建方式
定位：构建在 Kubernetes 之上的分布式云原生操作系统，提供可视化 Web 控制台。

路径一：从零安装（KubeKey，推荐新手）
bash
export KKZONE=cn                    # 国内镜像加速
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.13 sh -
chmod +x kk
./kk create config --with-kubernetes v1.27.4
# 编辑 config-sample.yaml 填入服务器信息
./kk create cluster -f config-sample.yaml
路径二：已有 K8s 集群，仅装控制台（Helm）
bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm upgrade --install -n kubesphere-system --create-namespace ks-core \
  oci://hub.kubesphere.com.cn/kse/ks-core \
  --debug --wait --version 1.2.2 --reset-values \
  --set extension.imageRegistry=swr.cn-north-9.myhuaweicloud.com/ks

kubectl get svc -n kubesphere-system ks-console
默认端口 30880，账号 admin，密码 P@88w0rd（首次登录强制修改）。

社区版限制：最多 128 vCPU，只能管理 1 个集群。

21. Kubernetes Dashboard 与 Headlamp
Dashboard 现状
官方项目已归档，不再积极维护。

仅支持 Helm 安装。

bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace --namespace kubernetes-dashboard

# 创建管理员 Token
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:dashboard-admin
kubectl -n kubernetes-dashboard create token dashboard-admin

# 访问
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
# 浏览器访问 https://localhost:8443
Headlamp（推荐替代）
官方 SIG 维护，活跃开发。

bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo update
helm install my-headlamp headlamp/headlamp --namespace kube-system

# 创建管理员 Token
kubectl -n kube-system create serviceaccount headlamp-admin
kubectl create clusterrolebinding headlamp-admin \
  --serviceaccount=kube-system:headlamp-admin \
  --clusterrole=cluster-admin
kubectl create token headlamp-admin -n kube-system

# 访问（端口转发）
kubectl -n kube-system port-forward svc/my-headlamp 8080:80
# 浏览器访问 http://localhost:8080

# 或改为 NodePort 供局域网访问
kubectl -n kube-system patch svc my-headlamp -p '{"spec":{"type":"NodePort"}}'
22. HPA / PVC / PV 概念解释
概念	全称	作用	类比
PV	PersistentVolume	集群管理员预先准备的存储资源，生命周期独立于 Pod	管理员划好的一块硬盘
PVC	PersistentVolumeClaim	用户对存储的申请（容量、读写权限），自动绑定满足条件的 PV	填写的"我要 10G 硬盘"申请单
HPA	Horizontal Pod Autoscaler	根据 CPU/内存等指标自动增减 Pod 副本数	自动调节人手的调度员
关系：PV 是供给方，PVC 是需求方，解耦应用配置与存储实现。

HPA 前提：需安装 Metrics Server，可用 kubectl top pods 验证。

注意：MySQL 未配 PVC 时数据写在容器内，Pod 删除重建即丢失。生产环境应为 /var/lib/mysql 配 PVC。

23. busybox 镜像使用注意
最小可用 Pod：

yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-test
spec:
  containers:
  - name: busybox
    image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/library/busybox:latest
    command: ["sleep", "3600"]     # 必须有前台命令，否则容器立即退出
验证 PVC 挂载：

yaml
spec:
  containers:
  - name: busybox
    image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/library/busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: <PVC名称>
进入容器写文件后删除重建 Pod，验证 /data 数据是否持久化。

24. Namespace + Pod 的部署方式
推荐：写在同一文件，用 --- 分隔，Pod 必须显式指定 namespace。

yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-ns
---
apiVersion: v1
kind: Pod
metadata:
  name: my-busybox
  namespace: my-ns          # 关键
spec:
  containers:
  - name: busybox
    image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/library/busybox:latest
    command: ["sleep", "3600"]
bash
kubectl apply -f my-pod.yaml
kubectl get pods -n my-ns
kubectl describe pod my-busybox -n my-ns
常见坑：Pod 不写 namespace 会悄悄创建到 default，kubectl get pods -n my-ns 看不到，误以为部署失败。

25. pause 容器在 Headlamp/KubeSphere 中搜不到
结论：正常现象。

pause 容器由 kubelet 直接通过 containerd 管理，Kubernetes API Server 不知道它的存在。

Headlamp、KubeSphere、kubectl get pods 都基于 API 获取资源，因此都看不到。

确认方式：登录节点执行

bash
ctr -n k8s.io containers ls | grep pause
crictl images | grep pause
26. 外部机器连不上 MySQL Service
根因：Service 类型为 ClusterIP，只在集群内部有效。

解决：改为 NodePort。

bash
kubectl patch svc mysql -p '{"spec":{"type":"NodePort"}}'
kubectl get svc mysql     # 查看 PORT(S) 列，如 3306:3xxxx/TCP
外部连接：

bash
mysql -h 192.168.3.147 -P 3xxxx -u root -p
注意：

必须用节点真实 IP，不能用 ClusterIP。

若报 Access denied，需授权远程访问：

sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
FLUSH PRIVILEGES;
检查防火墙是否放行 NodePort 端口。

27. Deployment apiVersion 写错
报错：

text
no matches for kind "Deployment" in version "v1"
原因：Deployment 属于 apps API 组，不是核心 v1。

修复：

yaml
apiVersion: apps/v1
kind: Deployment
附带问题：

matchLabels 与 matchExpressions 同时写且等价 → 冗余，可删 matchExpressions。

image: tomcat 会走 Docker Hub，国内拉不到 → 换华为云地址。

28. Tomcat Deployment 一直不 Ready
报错：

text
Failed to pull image "tomcat": dial tcp ... registry-1.docker.io ... i/o timeout
Error: ImagePullBackOff
原因：image: tomcat 短名称走 Docker Hub。

解决：

bash
# 先在节点上验证
crictl pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/tomcat:latest
修改 YAML：

yaml
image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/tomcat:latest
重新应用：

bash
kubectl apply -f tomcat-deployment.yaml
kubectl get pods -w
注意：用 kubectl apply 而非 create，避免 AlreadyExists。

29. 删除并重建 Deployment
适用场景：YAML 反复修改、旧资源状态混乱。

bash
kubectl delete deployment tomcat-deploy
kubectl get pods | grep tomcat      # 确认已清理
重建前：先在目标节点上 crictl pull 验证镜像地址可用，避免再次 ImagePullBackOff（kubelet 重试间隔会越来越长）。

bash
kubectl apply -f tomcat-deployment.yaml
kubectl get pods -w
30. Tomcat 镜像 latest 标签 not found
报错：

text
failed to resolve reference ".../library/tomcat:latest": not found
原因：镜像同步站通常只同步明确版本号的标签，不同步会变动的 latest。

解决：换带版本号的地址（注意路径可能不含 library/）。

bash
# 先验证
crictl pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/tomcat:9.0.75-jdk8-corretto
YAML：

yaml
image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/tomcat:9.0.75-jdk8-corretto
bash
kubectl apply -f tomcat-deployment.yaml
kubectl get pods -w
建议：不要用 latest 标签试错，去渡渡鸟镜像同步站（docker.aityp.com）搜索 tomcat，选标注"已同步"的版本号。

附：常用镜像源对照表
用途	可用地址前缀	备注
K8s 控制平面组件	registry.aliyuncs.com/google_containers	kubeadm 配置 imageRepository
pause sandbox	registry.aliyuncs.com/google_containers/pause:3.9	containerd sandbox_image
Calico / MySQL / Tomcat / busybox	swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/	华为云 SWR 同步站，已验证可用
Docker Hub	docker.io	国内基本不可用
轩辕镜像	docker.xuanyuan.run / docker.xuanyuan.me	需登录或已失效
DaoCloud	docker.m.daocloud.io	已停止服务
验证镜像可用性的通用方法：

bash
crictl pull <完整镜像地址>
# 看到 "Image is up to date" 即成功