# networking

Demonstrate how k8s networking work. Goals:

[All pods are joined a same pod network and the IP is get from Node CIDR](#1.)

[Pod can connect to another Pod via service](#service)

[Demo load balancer](demo-load-balcner)

## Up and running a k8s cluster with 2 nodes

``` bash
curl -O https://github.com/kubernetes-sigs/kubeadm-dind-cluster/releases/download/v0.1.0/dind-cluster-v1.13.sh
chmod +x dind-cluster-v1.13.sh
```

``` bash
./dind-cluster-v1.13.sh up
```

## All pods are joined a same pod network and the IP is get from Node CIDR

### Up and running cluster with 2 nodes on local

``` bash
./dind-cluster-v1.13.sh up
```

### Deploy the 2 container on 2 pods

``` bash
kubectl apply -f ping-deploy.yml
```

### Prove every pod is in the same pod network

Get pods IPs and node ip range

``` bash
kubectl get pods -o wide
kubectl get nodes
```

``` console
NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE          NOMINATED NODE   READINESS GATES
pingtest-6b78585c94-jjdps   1/1     Running   0          4m21s   10.244.2.3   kube-node-1   <none>           <none>
pingtest-6b78585c94-zqqmh   1/1     Running   0          4m21s   10.244.3.4   kube-node-2   <none>           <none>
```

``` console
NAME          STATUS   ROLES    AGE     VERSION
kube-master   Ready    master   5h41m   v1.13.0
kube-node-1   Ready    <none>   5h40m   v1.13.0
kube-node-2   Ready    <none>   5h40m   v1.13.0
```

``` note
Pod IPs: 10.244.2.3 and 10.244.3.4
```

Connect to a container in pod and ping other pod

``` bash
kubectl exec -it pingtest-6b78585c94-jjdps ash
ping 10.244.3.4
```

## Pod can connect to service

### Start a simple hello-world service with Node Port:30001 and Pod Port: 8080

``` bash
kubectl apply -f simple-web.yml
kubectl get services
```

``` output
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-svc    NodePort    10.98.17.194   <none>        8080:30001/TCP   54m
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP          5h45m
```

### Access via Pod Port 8080

The service name will resolved to Cluster-IP. Access the web service by service name or Cluster IP is equivalent

``` bash
kubectl exec -it pingtest-6b78585c94-jjdps ash
wget -qO- hello-svc:8080
wget -qO- 10.98.17.194:8080
```

### Access via Node Port 30001

Get node IP

``` bash
kubectl get node kube-node-1 -o=jsonpath='{.status.addresses[0].address}'
```

``` output
10.192.0.3
```

Connect to service via node port 30001 from a pod. If we deploy our cluster on cloud or minikube we can access from our local.

``` bash
kubectl exec -it pingtest-6b78585c94-jjdps ash
wget -qO- 10.192.0.3:30001


Output

``` console
<html><head><title>ACG loves K8S</title><link rel="stylesheet" href="http://netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css"/></head><body><div class="container"><div class="jumbotron"><h1>A Cloud Guru loves Kubernetes!!!</h1><p></p><p> <a class="btn btn-primary" href="https://www.amazon.com/Kubernetes-Book-Nigel-Poulton/dp/1521823634/ref=sr_1_3?ie=UTF8&amp;qid=1531240306&amp;sr=8-3&amp;keywords=nigel+poulton">The Kubernetes Book</a></p><p></p></div></div></body></html>
```

## Demo load balancer

```bash
kubectl apply -f lb.yml
kubectl get svc
```

Output

``` console
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-svc    NodePort       10.98.17.194    <none>        8080:30001/TCP   82m
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP          6h13m
lb-svc       LoadBalancer   10.109.78.144   <pending>     8080:31145/TCP   2m55s
```

The K8s cluster we are using doesn't support external load balancer then we have connect to a container to test the internal IP: 10.109.78.144

``` bash
wget -qO- 10.109.78.144:8080
```