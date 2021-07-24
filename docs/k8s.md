
#K8S简单操作命令
```shell
kubectl get pods -o wide #查看pods网络中的应用
kubectl get nodes #查看所有k8s节点
kubectl create depoloyment tomcat6 --image=tomcat.6.0.53-jre8 #启动tomcat镜像
kubectl expose deployment tomcat6 --port=80 --target-port=8080 --type=NodePort#暴露Tomcat端口
kubectl get svc -o wide #获取service信息
kubectl scale --replicas=3 deployment tomcat6 #扩容Tomcat6服务  副本数为3


kubectl get all #获取所有资源信息
kubectl delete [NAME] #按名称删除部署





```



kubectl文档：

```http
https://kubernetes.io/zh/docs/reference/kubectl/overview
```

 