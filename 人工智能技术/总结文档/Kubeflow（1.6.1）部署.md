# Kubeflow（1.6.1）部署

# 前言

安装前请注意捋清楚版本关系，如kubeflow版本对应的K8S版本及其相关工具版本等等

我们此处使用的是是kubeflow-1.6.1和K8s-v1.22.8

# 单机部署

## 部署K8S

### 初始化Linux

#### 1.关闭selinux

```bash
 setenforce 0 && sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

#### 2.关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```

#### 3.设置hostname

```bash
hostnamectl set-hostname ai-node
```

#### 4.关闭swap

```bash
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab 
```

#### 5.修改内核参数和模块

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
#使内核参数配置生效
sysctl --system
modprobe br_netfilter
lsmod | grep br_netfilter
```

#### 6.更新系统及内核（可选）

[升级centos7及其内核](https://blog.csdn.net/weixin_45661908/article/details/123377496)

### 安装docker

#### 1.安装docker-ce

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum -y install docker-ce
```

#### 2.替换国内镜像

```bash
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}

```

#### 3.启动docker-ce

```bash
systemctl start docker
systemctl enable docker
```

### 安装kubernetes

#### 1.配置yum源

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
# repo_gpgcheck要设置为0，如设置为1会导致后面在install kubelet、kubeadm、kubectl的时候报[Errno -1] repomd.xml signature could not be verified for kubernetes Trying other mirror.
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 2.安装kubernetes基础服务

```bash
yum install -y kubelet-1.22.8 kubeadm-1.22.8 kubectl-1.22.8
systemctl start kubelet
systemctl enable kubelet.service
```

#### 3.初始化K8S

```bash
# apiserver-advertise-address指定master的interface，版本号与安装的K8S版本要一致，pod-network-cidr指定Pod网络的范围，这里使用flannel网络方案。
# 安装成功之后，会打印kubeadm join的输出，记得要保存下来，后面需要这个命令将各个节点加入集群中
kubeadm init --apiserver-advertise-address=192.168.0.240 --kubernetes-version v1.22.8 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16
 
## 如果初始化过程中出现错误，就reset之后重新init
# kubeadm reset
# rm -rf $HOME/.kube/config​
 
# 查看是否所有的pod都处于running状态
kubectl get pod -n kube-system -o wide
```

#### 4.初始化kubectl

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 5.设置kubectl自动补充

```bash
source <(kubectl completion bash)
```

可以加入~/.bashrc中以便在新的session中不需要手动加载

#### 6.网络插件

> 比较常用的时flannel和calico，flannel的功能比较简单，不具备复杂网络的配置能力，calico是比较出色的网络管理插件，单具备复杂网络配置能力的同时，往往意味着本身的配置比较复杂，所以相对而言，比较小而简单的集群使用flannel，考虑到日后扩容，未来网络可能需要加入更多设备，配置更多策略，则使用calico更好

**以下网络插件二选一即可**

##### 6.1 安装calico网络插件

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

##### 6.2 安装flannel网络插件

For Kubernetes v1.17+

```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

#### 7.解除master限制

默认k8s的master节点是不能跑pod的业务，需要执行以下命令解除限制

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

##  部署kubeflow

### 下载安装脚本

官方仓库地址：https://github.com/kubeflow/manifests

### 安装kustomize

> kustomize 是一个通过 kustomization 文件定制 kubernetes 对象的工具，它可以通过一些资源生成一些新的资源，也可以定制不同的资源的集合。

```bash
wget https://github.com/kubernetes-sigs/kustomize/releases/download/v3.2.0/kustomize_3.2.0_linux_amd64
mv kustomize_3.2.0_linux_amd64 kustomize
chmod +x kustomize
mv kustomize /usr/bin/
```

### 镜像同步

#### 1.说明

因为kubeflow的镜像存储在google镜像仓库，国内被墙，因此正常安装方式是不会安装成功的，此时提供两种途径

1.使用国内的同步镜像

2.自己从google仓库同步镜像

第一种方式显然比较简单，但是问题也和明显，镜像很多时候同步并不及时，因此很多镜像都是老版本的，如果想用全新版本安装，想要找到一个合适的镜像仓库，还是比较费劲的

第二种方式就一个要求：使用科技上网【不懂的话还是选第一种方式吧】

因为我们当前需要安装的kubeflow-1.6.1是最新版本，国内的同步仓库目前没发现最新版本，因此，我们选择第二种方式

#### 2.同步

2.1 网络问题搞定后【可以科技上网】，将刚才下载的manifests-1.6.1.tar.gz包解压

```bash
tar -zxvf manifests-1.6.1.tar.gz
```

2.2 进入目录

```bash
cd manifests-1.6.1
```

获取gcr镜像，因为我的网络只无法获取gcr.io, quay.io正常，可以根据需求修改

```bash
kustomize build example |grep 'image: gcr.io'|awk '$2 != "" { print $2}' |sort -u 
```

检查一下如果有镜像不带tag，说明提取的时候有问题，将awk去掉后仔细看看，gcr.io仓库下载是需要带tag的，换句话说，好像没有latest

2.3 使用脚本将以上获取的镜像同步至指定仓库，可以是dockerhub，也可以是私有镜像仓库

脚本配置,此脚本是网友编写，源码地址：https://github.com/kenwoodjw/sync_gcr

```bash
# tree sync_gcr
sync_gcr/
├── images.txt
├── load_image.py
├── README.md
└── sync_image.py

```

将步骤2.2获取的镜像列表放到images.txt中

修改sync_image.py中的镜像仓库及相关登录信息（如果是public仓库，则不需要login）

```python
# coding:utf-8
import subprocess, os
def get_filename():
    with open("images.txt", "r") as f:
        lines = f.read().split('\n')
        # print(lines)
        return lines


def pull_image():
    name_list= get_filename()
    for name in name_list:
        if 'sha256' in name:
            print(name)
            sha256_name = name.split("@")
            new_name = sha256_name[0].split("/")[-1]
            tag = sha256_name[-1].split(":")[-1][0:6]
            #此处为了加载镜像速度，我放在内网的私有镜像仓库中
            image = "192.168.8.38:9090/grc-io/" + new_name + ":"+ tag
            cmd = "docker tag {0}   {1}".format(name, image)
            subprocess.call("docker pull {}".format(name), shell=True)
            subprocess.run(["docker", "tag", name, image])
            #subprocess.call("docker login -u user -p passwd", shell=True)
            subprocess.call("docker push {}".format(image), shell=True)
        else:
            new_name = "192.168.8.38:9090/grc-io/" + name.split("/")[-1]
            cmd = "docker tag {0}   {1}".format(name, new_name)
            subprocess.call("docker pull {}".format(name), shell=True)
            subprocess.run(["docker", "tag", name, new_name])
            #subprocess.call("docker login -u user -p passwd", shell=True)
            subprocess.call("docker push {}".format(new_name), shell=True)
        
if __name__ == "__main__":
    pull_image()

```

2.4 执行脚本

```bash
python sync_image.py
```

这是一个漫长的过程，主要是需要从gcr.io下载镜像，镜像有大有小，数量较多，下载时间完全看个人网速，慢慢等待

在等待的过程中可以同步修改一下安装文件，即2.5

2.5修改部署文件

因为我们将镜像同步到自己的仓库，所以需要修改一下镜像地址

在manifests-1.6.1目录下，打开配置文件

```bash
vim example/kustomization.yaml
```

新增内容【依据版本不同，镜像版本也不同，请不要无脑照抄】



```yaml
images:
- name: gcr.io/arrikto/istio/pilot:1.14.1-1-g19df463bb
  newName:  192.168.8.38:9090/grc-io/pilot
  newTag: "1.14.1-1-g19df463bb"
- name: gcr.io/arrikto/kubeflow/oidc-authservice:28c59ef
  newName:  192.168.8.38:9090/grc-io/oidc-authservice
  newTag: "28c59ef"
- name: gcr.io/knative-releases/knative.dev/eventing/cmd/controller@sha256:dc0ac2d8f235edb04ec1290721f389d2bc719ab8b6222ee86f17af8d7d2a160f
  newName:  192.168.8.38:9090/grc-io/controller
  newTag: "dc0ac2"
- name: gcr.io/knative-releases/knative.dev/eventing/cmd/mtping@sha256:632d9d710d070efed2563f6125a87993e825e8e36562ec3da0366e2a897406c0
  newName:  192.168.8.38:9090/grc-io/mtping
  newTag: "632d9d"
- name: gcr.io/knative-releases/knative.dev/eventing/cmd/webhook@sha256:b7faf7d253bd256dbe08f1cac084469128989cf39abbe256ecb4e1d4eb085a31
  newName:  192.168.8.38:9090/grc-io/webhook
  newTag: "b7faf7"
- name: gcr.io/knative-releases/knative.dev/net-istio/cmd/controller@sha256:f253b82941c2220181cee80d7488fe1cefce9d49ab30bdb54bcb8c76515f7a26
  newName:  192.168.8.38:9090/grc-io/controller
  newTag: "f253b8"
- name: gcr.io/knative-releases/knative.dev/net-istio/cmd/webhook@sha256:a705c1ea8e9e556f860314fe055082fbe3cde6a924c29291955f98d979f8185e
  newName:  192.168.8.38:9090/grc-io/webhook
  newTag: "a705c1"
- name: gcr.io/knative-releases/knative.dev/serving/cmd/activator@sha256:93ff6e69357785ff97806945b284cbd1d37e50402b876a320645be8877c0d7b7
  newName:  192.168.8.38:9090/grc-io/activator
  newTag: "93ff6e"
- name: gcr.io/knative-releases/knative.dev/serving/cmd/autoscaler@sha256:007820fdb75b60e6fd5a25e65fd6ad9744082a6bf195d72795561c91b425d016
  newName:  192.168.8.38:9090/grc-io/autoscaler
  newTag: "007820"
- name: gcr.io/knative-releases/knative.dev/serving/cmd/controller@sha256:75cfdcfa050af9522e798e820ba5483b9093de1ce520207a3fedf112d73a4686
  newName:  192.168.8.38:9090/grc-io/controller
  newTag: "75cfdc"
- name: gcr.io/knative-releases/knative.dev/serving/cmd/domain-mapping@sha256:23baa19322320f25a462568eded1276601ef67194883db9211e1ea24f21a0beb
  newName:  192.168.8.38:9090/grc-io/domain-mapping
  newTag: "23baa1"
- name: gcr.io/knative-releases/knative.dev/serving/cmd/domain-mapping-webhook@sha256:847bb97e38440c71cb4bcc3e430743e18b328ad1e168b6fca35b10353b9a2c22
  newName:  192.168.8.38:9090/grc-io/domain-mapping-webhook
  newTag: "847bb9"
- name: gcr.io/knative-releases/knative.dev/serving/cmd/queue@sha256:14415b204ea8d0567235143a6c3377f49cbd35f18dc84dfa4baa7695c2a9b53d
  newName:  192.168.8.38:9090/grc-io/queue
  newTag: "14415b"
- name: gcr.io/knative-releases/knative.dev/serving/cmd/webhook@sha256:9084ea8498eae3c6c4364a397d66516a25e48488f4a9871ef765fa554ba483f0
  newName:  192.168.8.38:9090/grc-io/webhook
  newTag: "9084ea"
- name: gcr.io/kubebuilder/kube-rbac-proxy:v0.8.0
  newName:  192.168.8.38:9090/grc-io/kube-rbac-proxy
  newTag: "v0.8.0"
- name: gcr.io/ml-pipeline/api-server:2.0.0-alpha.5
  newName:  192.168.8.38:9090/grc-io/api-server
  newTag: "2.0.0-alpha.5"
- name: gcr.io/ml-pipeline/cache-server:2.0.0-alpha.5
  newName:  192.168.8.38:9090/grc-io/cache-server
  newTag: "2.0.0-alpha.5"
- name: gcr.io/ml-pipeline/frontend:2.0.0-alpha.5
  newName:  192.168.8.38:9090/grc-io/frontend
  newTag: "2.0.0-alpha.5"
- name: gcr.io/ml-pipeline/metadata-writer:2.0.0-alpha.5
  newName:  192.168.8.38:9090/grc-io/metadata-writer
  newTag: "2.0.0-alpha.5"
- name: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance
  newName:  192.168.8.38:9090/grc-io/minio
  newTag: "RELEASE.2019-08-14T20-37-41Z-license-compliance"
- name: gcr.io/ml-pipeline/mysql:5.7-debian
  newName:  192.168.8.38:9090/grc-io/mysql
  newTag: "5.7-debian"
- name: gcr.io/ml-pipeline/persistenceagent:2.0.0-alpha.5
  newName:  192.168.8.38:9090/grc-io/persistenceagent
  newTag: "2.0.0-alpha.5"
- name: gcr.io/ml-pipeline/scheduledworkflow:2.0.0-alpha.5
  newName:  192.168.8.38:9090/grc-io/scheduledworkflow
  newTag: "2.0.0-alpha.5"
- name: gcr.io/ml-pipeline/viewer-crd-controller:2.0.0-alpha.5
  newName:  192.168.8.38:9090/grc-io/viewer-crd-controller
  newTag: "2.0.0-alpha.5"
- name: gcr.io/ml-pipeline/visualization-server:2.0.0-alpha.5
  newName:  192.168.8.38:9090/grc-io/visualization-server
  newTag: "2.0.0-alpha.5"
- name: gcr.io/ml-pipeline/workflow-controller:v3.3.8-license-compliance
  newName:  192.168.8.38:9090/grc-io/workflow-controller
  newTag: "v3.3.8-license-compliance"
- name: gcr.io/tfx-oss-public/ml_metadata_store_server:1.5.0
  newName:  192.168.8.38:9090/grc-io/ml_metadata_store_server
  newTag: "1.5.0"
- name: gcr.io/ml-pipeline/metadata-envoy:2.0.0-alpha.5
  newName:  192.168.8.38:9090/grc-io/metadata-envoy
  newTag: "2.0.0-alpha.5"
```

具体位置

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
# Cert-Manager
- ../common/cert-manager/cert-manager/base
- ../common/cert-manager/kubeflow-issuer/base
# Istio
- ../common/istio-1-14/istio-crds/base
- ../common/istio-1-14/istio-namespace/base
- ../common/istio-1-14/istio-install/base
# OIDC Authservice
- ../common/oidc-authservice/base
# Dex
- ../common/dex/overlays/istio
# KNative
- ../common/knative/knative-serving/overlays/gateways
- ../common/knative/knative-eventing/base
- ../common/istio-1-14/cluster-local-gateway/base
# Kubeflow namespace
- ../common/kubeflow-namespace/base
# Kubeflow Roles
- ../common/kubeflow-roles/base
# Kubeflow Istio Resources
- ../common/istio-1-14/kubeflow-istio-resources/base


# Kubeflow Pipelines
- ../apps/pipeline/upstream/env/cert-manager/platform-agnostic-multi-user
# Katib
- ../apps/katib/upstream/installs/katib-with-kubeflow
# Central Dashboard
- ../apps/centraldashboard/upstream/overlays/kserve
# Admission Webhook
- ../apps/admission-webhook/upstream/overlays/cert-manager
# Jupyter Web App
- ../apps/jupyter/jupyter-web-app/upstream/overlays/istio
# Notebook Controller
- ../apps/jupyter/notebook-controller/upstream/overlays/kubeflow
# Profiles + KFAM
- ../apps/profiles/upstream/overlays/kubeflow
# Volumes Web App
- ../apps/volumes-web-app/upstream/overlays/istio
# Tensorboards Controller
-  ../apps/tensorboard/tensorboard-controller/upstream/overlays/kubeflow
# Tensorboard Web App
-  ../apps/tensorboard/tensorboards-web-app/upstream/overlays/istio
# Training Operator
- ../apps/training-operator/upstream/overlays/kubeflow
# User namespace
- ../common/user-namespace/base

# KServe
- ../contrib/kserve/kserve
- ../contrib/kserve/models-web-app/overlays/kubeflow
images:
- name: gcr.io/arrikto/istio/pilot:1.14.1-1-g19df463bb
  newName:  192.168.8.38:9090/grc-io/pilot
  newTag: "1.14.1-1-g19df463bb"
- name: gcr.io/arrikto/kubeflow/oidc-authservice:28c59ef
  newName:  192.168.8.38:9090/grc-io/oidc-authservice
  newTag: "28c59ef"
- name: gcr.io/knative-releases/knative.dev/eventing/cmd/controller@sha256:dc0ac2d8f235edb04ec1290721f389d2bc719ab8b6222ee86f17af8d7d2a160f
  newName:  192.168.8.38:9090/grc-io/controller
  newTag: "dc0ac2"
........
```

2.6 创建PV（可选）

如果你的单机集群没有任何Provisioner，那就需要手动创建pv，如果已经安装了相关的Provisioner和StorageClass，那么这一步可以省略。

2.6.1 手工创建

kubeflow中有四个服务是statefulsets，因此需要挂在卷，因为我们是单机安装，所以直接创建local存储即可，PV大小请自行配置

```bash
##创建存储卷目录
mkdir -p /opt/kubeflow/{pv1,pv2,pv3,pv4}
# 创建pv的yaml
cat pv.yaml 
```



```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-001
spec:
  capacity:
    storage: 80Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/opt/kubeflow/pv1"

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-002
spec:
  capacity:
    storage: 80Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/opt/kubeflow/pv2"

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-003
spec:
  capacity:
    storage: 80Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/opt/kubeflow/pv3"

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-004
spec:
  capacity:
    storage: 80Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/opt/kubeflow/pv4"
```

执行创建

```bash
kubectl apply -f pv.yaml
```

2.6.2 OpenEBS 实现 Local PV 动态持久化存储

OpenEBS(https://openebs.io) 是一种模拟了 [AWS](https://so.csdn.net/so/search?q=AWS&spm=1001.2101.3001.7020) 的 EBS、阿里云的云盘等块存储实现的基于容器的存储开源软件。具体描述大家可以访问官网，此处进行一下安装教程（简单）

如果大家选择默认安装，则直接执行

```bash
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
```

如果需要进行一些自定义，则可以直接将yaml文件下载下来，修改后再部署，比如默认pv存储路径是/var/openebs/local，一般系统盘较小，如果有单独数据盘，我们可以修改一下`OPENEBS_IO_LOCALPV_HOSTPATH_DIR`的参数。

同时要修改的还有openebs-hostpath的`- name: BasePath`的value值，改成跟`OPENEBS_IO_LOCALPV_HOSTPATH_DIR`相同即可。

查看pod启动情况

```bash
kubectl get pods -n openebs 
```

默认情况下 OpenEBS 还会安装一些内置的 StorageClass 对象：

```bash
kubectl get sc
```

设置openebs-hostpath为default sc：

```bash
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

![image-20221119140651428](C:\Users\wangjb\AppData\Roaming\Typora\typora-user-images\image-20221119140651428.png)



> 使用动态存储的好处是，部署完kubeflow后，使用过程中很多任务都需要pv，比如notebook的创建等。如果还是手工创建pv，那么每次创建新的需要持久化的目标时，就需要手工操作一下pv创建，比较麻烦。动态存储就解决了这些问题

2.7 等镜像全部同步完成后，执行部署

依然在`manifests-1.6.1`目录下

```bash
while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done
```

查看pod部署情况

```bash
kubectl get pods --all-namespaces
```

```bash
[root@ai-node manifests-1.6.1]# kubectl get pods --all-namespaces 
NAMESPACE                   NAME                                                     READY   STATUS             RESTARTS        AGE
auth                        dex-559dbcd758-wmf57                                     1/1     Running            2 (21h ago)     46h
cert-manager                cert-manager-7b8c77d4bd-8jjmd                            1/1     Running            2 (21h ago)     46h
cert-manager                cert-manager-cainjector-7c744f57b5-vmgws                 1/1     Running            2 (21h ago)     46h
cert-manager                cert-manager-webhook-fcd445bc4-rspjk                     1/1     Running            2 (21h ago)     46h
istio-system                authservice-0                                            1/1     Running            0               5h7m
istio-system                cluster-local-gateway-55ff4696f4-ddjzl                   1/1     Running            0               5h24m
istio-system                istio-ingressgateway-6668f9548d-zh6tp                    1/1     Running            0               5h24m
istio-system                istiod-64bd848cc4-8ktxh                                  1/1     Running            0               5h24m
knative-eventing            eventing-controller-55c757fbf4-z5z8b                     1/1     Running            1 (21h ago)     22h
knative-eventing            eventing-webhook-78dccf77d-7xf2c                         1/1     Running            1 (21h ago)     22h
knative-serving             activator-f6fbdbdd7-kgxjk                                2/2     Running            0               21h
knative-serving             autoscaler-5c546f654c-w8rmv                              2/2     Running            0               21h
knative-serving             controller-594bc5bbb9-dn7nx                              2/2     Running            0               21h
knative-serving             domain-mapping-849f785857-8h9hx                          2/2     Running            0               21h
knative-serving             domainmapping-webhook-5954cfd85b-hrdcl                   2/2     Running            0               21h
knative-serving             net-istio-controller-655fd85bc4-gqgrk                    2/2     Running            0               21h
knative-serving             net-istio-webhook-66c78c9cdb-6lwtl                       2/2     Running            0               21h
knative-serving             webhook-6895c68dfd-h58l4                                 2/2     Running            0               21h
kube-system                 calico-kube-controllers-796cc7f49d-ldvfp                 1/1     Running            2 (21h ago)     46h
kube-system                 calico-node-fdqqp                                        1/1     Running            2 (21h ago)     46h
kube-system                 coredns-7f6cbbb7b8-8lwrc                                 1/1     Running            2 (21h ago)     46h
kube-system                 coredns-7f6cbbb7b8-cgqkc                                 1/1     Running            2 (21h ago)     46h
kube-system                 etcd-ai-node                                             1/1     Running            2 (21h ago)     46h
kube-system                 kube-apiserver-ai-node                                   1/1     Running            2 (21h ago)     46h
kube-system                 kube-controller-manager-ai-node                          1/1     Running            2 (21h ago)     46h
kube-system                 kube-proxy-4xzxn                                         1/1     Running            2 (21h ago)     46h
kube-system                 kube-scheduler-ai-node                                   1/1     Running            2 (21h ago)     46h
kubeflow-user-example-com   ml-pipeline-ui-artifact-69cc696464-mh4pb                 2/2     Running            0               3h8m
kubeflow-user-example-com   ml-pipeline-visualizationserver-64d797bd94-xn74q         2/2     Running            0               3h9m
kubeflow                    admission-webhook-deployment-79d6f8c8fb-qkssk            1/1     Running            0               5h24m
kubeflow                    cache-server-746dd68dd9-qff72                            2/2     Running            0               5h23m
kubeflow                    centraldashboard-f64b457f-6npr2                          2/2     Running            0               5h23m
kubeflow                    jupyter-web-app-deployment-576c56f555-th492              1/1     Running            0               5h23m
kubeflow                    katib-controller-75b988dccc-d5xsz                        1/1     Running            0               5h23m
kubeflow                    katib-db-manager-5d46869758-9khlf                        1/1     Running            0               5h23m
kubeflow                    katib-mysql-5bf95ddfcc-p55d6                             1/1     Running            0               5h23m
kubeflow                    katib-ui-766d5dc8ff-vngv2                                1/1     Running            0               5h24m
kubeflow                    kserve-controller-manager-0                              2/2     Running            0               5h23m
kubeflow                    kserve-models-web-app-5878544ffd-s8d84                   2/2     Running            0               5h23m
kubeflow                    kubeflow-pipelines-profile-controller-5d98fd7b4f-765x7   1/1     Running            0               5h23m
kubeflow                    metacontroller-0                                         1/1     Running            0               5h23m
kubeflow                    metadata-envoy-deployment-5b96bc6fd6-rfhwj               1/1     Running            0               3h25m
kubeflow                    metadata-grpc-deployment-59d555db46-gbgn9                2/2     Running            5 (5h22m ago)   5h23m
kubeflow                    metadata-writer-76bbdb799f-kftpv                         2/2     Running            1 (5h22m ago)   5h23m
kubeflow                    minio-86db59fd6-jv5xv                                    2/2     Running            0               5h23m
kubeflow                    ml-pipeline-8587f95f8f-2xl4w                             2/2     Running            2 (5h20m ago)   5h24m
kubeflow                    ml-pipeline-persistenceagent-568f4bddb5-6bqxk            2/2     Running            0               5h24m
kubeflow                    ml-pipeline-scheduledworkflow-5c74f8dff4-wwpr5           2/2     Running            0               5h24m
kubeflow                    ml-pipeline-ui-5c684875c-5dm98                           2/2     Running            0               5h23m
kubeflow                    ml-pipeline-viewer-crd-748d77b759-wz5r2                  2/2     Running            1 (5h21m ago)   5h23m
kubeflow                    ml-pipeline-visualizationserver-5b697bd55d-97xwt         2/2     Running            0               3h25m
kubeflow                    mysql-77578849cc-dtx42                                   2/2     Running            0               5h23m
kubeflow                    notebook-controller-deployment-68756676d9-zdbr4          2/2     Running            1 (5h21m ago)   5h23m
kubeflow                    profiles-deployment-79d49b8648-xd7pg                     3/3     Running            1 (5h21m ago)   5h23m
kubeflow                    tensorboard-controller-deployment-6f879dd7f6-s7mk7       3/3     Running            1 (5h23m ago)   5h23m
kubeflow                    tensorboards-web-app-deployment-6849d8c9bc-bs52h         1/1     Running            0               5h23m
kubeflow                    training-operator-6c9f6fd894-qsxcg                       1/1     Running            0               5h23m
kubeflow                    volumes-web-app-deployment-5f56dd78-22jvh                1/1     Running            0               5h23m
kubeflow                    workflow-controller-686dd58c95-5cmxg                     2/2     Running            1 (5h23m ago)   5h23m
```

等所有容器都起来后，查看80的映射端口是哪个

```bash
kubectl -n istio-system get svc istio-ingressgateway 
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   NodePort   10.1.109.168   <none>        15021:30965/TCP,80:31714/TCP,443:30626/TCP,31400:32535/TCP,15443:32221/TCP   46h

```

从输出可以看到80映射的是31714，因此打开kubeflow页面的地址就是http://服务器IP:31714/

![image-20221027151829241](C:\Users\wangjb\AppData\Roaming\Typora\typora-user-images\image-20221027151829241.png)

登录的账号一般默认是user@example.com，密码：12341234，如果不对，则在dex的configmaps中查看

```bash
kubectl -n auth get configmaps dex -o yaml
```



```
apiVersion: v1
data:
  config.yaml: |
    issuer: http://dex.auth.svc.cluster.local:5556/dex
    storage:
      type: kubernetes
      config:
        inCluster: true
    web:
      http: 0.0.0.0:5556
    logger:
      level: "debug"
      format: text
    oauth2:
      skipApprovalScreen: true
    enablePasswordDB: true
    staticPasswords:
    - email: user@example.com
      hash: $2y$12$4K/VkmDd1q1Orb3xAt82zu8gk7Ad6ReFR4LCP9UeYE90NLiN9Df72
      # https://github.com/dexidp/dex/pull/1601/commits
      # FIXME: Use hashFromEnv instead
      username: user
      userID: "15841185641784"
    staticClients:
    # https://github.com/dexidp/dex/pull/1664
    - idEnv: OIDC_CLIENT_ID
      redirectURIs: ["/login/oidc"]
      name: 'Dex Login Application'
      secretEnv: OIDC_CLIENT_SECRET
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"config.yaml":"issuer: http://dex.auth.svc.cluster.local:5556/dex\nstorage:\n  type: kubernetes\n  config:\n    inCluster: true\nweb:\n  http: 0.0.0.0:5556\nlogger:\n  level: \"debug\"\n  format: text\noauth2:\n  skipApprovalScreen: true\nenablePasswordDB: true\nstaticPasswords:\n- email: user@example.com\n  hash: $2y$12$4K/VkmDd1q1Orb3xAt82zu8gk7Ad6ReFR4LCP9UeYE90NLiN9Df72\n  # https://github.com/dexidp/dex/pull/1601/commits\n  # FIXME: Use hashFromEnv instead\n  username: user\n  userID: \"15841185641784\"\nstaticClients:\n# https://github.com/dexidp/dex/pull/1664\n- idEnv: OIDC_CLIENT_ID\n  redirectURIs: [\"/login/oidc\"]\n  name: 'Dex Login Application'\n  secretEnv: OIDC_CLIENT_SECRET\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"dex","namespace":"auth"}}
  creationTimestamp: "2022-10-25T08:51:08Z"
  name: dex
  namespace: auth
  resourceVersion: "2956"
  uid: a0716cd6-f259-4a49-bee3-783c1069e8d2

```

staticPasswords.email就是用户名

staticPasswords.hash就是密码

生成代码（python）:

python console下执行，可生成hash密码

```python
from passlib.hash import bcrypt
import getpass
print(bcrypt.using(rounds=12, ident="2y").hash(getpass.getpass()))
```



### 报错修复

### 1.authservice-0 pod 启动失败

istio-system下的authservice-0启动失败，查看日志返现没有权限操作目录/var/lib/authservice/data.db

```bash
authservice-0 pod 启动失败：Error opening bolt store: open /var/lib/authservice/data.db: permission denied
```

解决方案：

修改`common/oidc-authservice/base/statefulset.yaml`, 添加以下内容

```yaml
  initContainers:
  - name: fix-permission
    image: busybox
    command: ['sh', '-c']
    args: ['chmod -R 777 /var/lib/authservice;']
    volumeMounts:
    - mountPath: /var/lib/authservice
      name: data
```

修改后的文件如下`vim common/oidc-authservice/base/statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
 name: authservice
spec:
 replicas: 1
 selector:
   matchLabels:
     app: authservice
 serviceName: authservice
 template:
   metadata:
     annotations:
       sidecar.istio.io/inject: "false"
     labels:
       app: authservice
   spec:
     initContainers:
     - name: fix-permission
       image: busybox
       command: ['sh', '-c']
       args: ['chmod -R 777 /var/lib/authservice;']
       volumeMounts:
       - mountPath: /var/lib/authservice
         name: data
     containers:
     - name: authservice
       image: gcr.io/arrikto/kubeflow/oidc-authservice:6ac9400
       imagePullPolicy: Always
       ports:
       - name: http-api
         containerPort: 8080
       envFrom:
         - secretRef:
             name: oidc-authservice-client
         - configMapRef:
             name: oidc-authservice-parameters
       volumeMounts:
         - name: data
           mountPath: /var/lib/authservice
       readinessProbe:
           httpGet:
             path: /
             port: 8081
     securityContext:
       fsGroup: 111
     volumes:
       - name: data
         persistentVolumeClaim:
             claimName: authservice-pvc
```

### 2.kubeflow-user-example-com镜像失败

kubeflow-user-example-com下两个镜像拉取失败

ml-pipeline-ui-artifact、ml-pipeline-visualizationserver

查看后发现拉取的还是gcr.io的镜像，还没来得及分析具体在哪个配置文件中修改，但是镜像跟kubeflow中的是相同的，因此只需要修改两个deploment的镜像地址即可，等有时间再仔细研究下在哪个部署文件修改

# 集群部署

K8S按照集群部署

kubeflow部署方式不变