# Kubernetes中部署Prometheus





## kube-prometheus

官方写好的yaml文件  和自己打镜像写yaml文件一样

包含的所有镜像要导入到公司内部的harbor上，把所有镜像名称过滤出来，然后科学下载、上传到公司。

```bash
git clone -b <选择兼容集群的版本> https://github.com/prometheus-operator/kube-prometheus.git
```



## 官方项目部署

### 1. 根据需要修改配置

```bash
cd  kube-prometheus/manifests/
#要修改yaml文件中的镜像地址
grep image: ./* -R

#官方提供networkpolicy(限制比较严格无法访问，修改或移走/删除)
ll *networkPolicy* 
mv ll *networkPolicy*.yaml  /opt/networkpolicy/
```



#### 1.1 创建资源对象

```bash
#根据个人实际情况修改service服务暴露
vim manifests/prometheus-service
vim manifests/grafana-service

#这个目录有一些大文件无法使用apply
kubectl create -f setup/

#新版本可以直接apply
kubectl apply --server-side -f manifests/setup #这一步是创建基础环境，比如namespace、CRD。
kubectl apply -f manifests
```



## 手动部署

yaml文件、



```bash 
#cadvisor
name: root
hostPath:
	path:/
name: run

hostPath:
	path:
	/var/run
name:sys
hostPath:
	path: /sys
name: docker
hostPath:
	#path:/var/lib/docker
	path: 
		/var/lib/containerd   #要改成自己的容器运行时
```

