<h1> kubernetes 学习文档</h1>

# 安装(CentOS7)

## 切换root用户

```shell
sudo -s
```

## 关闭防火墙

```shell
systemctl disable firewalld
systemctl stop firewalld
#临时禁用SELinux
setenforce 0
#或者永久禁用
sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/sysconfig/selinux

# 临时关闭swap
# 永久关闭 注释/etc/fstab文件里swap相关的行
swapoff -a

# 开启forward
# Docker从1.13版本开始调整了默认的防火墙规则
# 禁用了iptables filter表中FOWARD链
# 这样会引起Kubernetes集群中跨Node的Pod无法通信

iptables -P FORWARD ACCEPT
```

## 配置yum源

```shell
# 配置源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 更新yum cache
yum makecache
yum list
# 安装
yum install -y kubelet kubeadm kubectl ipvsadm
```

## 启动docker 和 kubelet服务

```shell
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet
```

## 使用minikube运行

安装见https://kubernetes.io/docs/tasks/tools/install-minikube/

```shell
$ minikube version
#启动minikube
$ minikube start
There is a newer version of minikube available (v0.28.2).  Download it here:
https://github.com/kubernetes/minikube/releases/tag/v0.28.2

To disable this notification, run the following:
minikube config set WantUpdateNotification false
Starting local Kubernetes v1.10.0 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
#启动minikube需要kubectl
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.0", GitCommit:"fc32d2f3698e36b93322a3465f63a14e9f0eaead", GitTreeState:"clean", BuildDate:"2018-03-26T16:55:54Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.0", GitCommit:"fc32d2f3698e36b93322a3465f63a14e9f0eaead", GitTreeState:"clean", BuildDate:"2018-04-10T12:46:31Z", GoVersion:"go1.9.4", Compiler:"gc", Platform:"linux/amd64"}
#查看集群信息
$ kubectl cluster-info
Kubernetes master is running at https://172.17.0.45:8443
```

# kubectl命令

```shell
#查看版本
kubectl version
#查看节点
kubectl get nodes
#运行app，除dockerhub外的镜像地址需要全仓库的url
kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=8080
#获取deployment列表
kubectl get deployments
#在新的terminal开启代理，打通和pod内部交互
kubectl proxy
Starting to serve on 127.0.0.1:8001
#查看pod的版本
curl http://localhost:8001/version
{
  "major": "1",
  "minor": "10",
  "gitVersion": "v1.10.0",
  "gitCommit": "fc32d2f3698e36b93322a3465f63a14e9f0eaead",
  "gitTreeState": "clean",
  "buildDate": "2018-04-10T12:46:31Z",
  "goVersion": "go1.9.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}
#获取POD名字
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME
#调用Pod的请求
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-5c69669756-p9s7m | v=1
#查看pods 内部的运行情况
kubectl describe pods
#查看pod内部的打印
kubectl logs $POD_NAME
#查看环境变量
kubectl exec $POD_NAME env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubernetes-bootcamp-5c69669756-p9s7m
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
NPM_CONFIG_LOGLEVEL=info
NODE_VERSION=6.3.1
HOME=/root
#进入pods
kubectl exec -ti $POD_NAME bash
root@kubernetes-bootcamp-5c69669756-p9s7m:/# kubectl exec -ti $POD_NAME bash
#获取服务
kubectl get services
#创建一个服务
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
#查看service详细信息
kubectl describe services/kubernetes-bootcamp
#获取对外端口号
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
#测试连接
curl $(minikube ip):$NODE_PORT

#通过label寻找
kubectl get pods -l run=kubernetes-bootcamp
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME
#添加标签
kubectl label pod $POD_NAME app=v1
#查看标签
kubectl describe pods $POD_NAME
kubectl get pods -l app=v1
#删除服务
kubectl delete service -l run=kubernetes-bootcamp
```