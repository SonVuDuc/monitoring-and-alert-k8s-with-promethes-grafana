# Monitoring và Alerting Kubernetes với Prometheus-Grafana

## 1.Yêu cầu

3 VPS hoặc VM:

  + 1 Node Master
  + 2 Node Worker
  + 1 Node Rancher

Cấu hình: 2 CPU, 4GB RAM

Hệ điều hành: Ubuntu 18.04 Server


## 2.Cài đặt Kubernetes Cluster

### Cài đặt cơ bản

#### Cài đặt docker

SSH lần lượt đến các VPS và cài đặt docker thông qua script

```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

#### Cài đặt Kubeadm, Kubectl và Kubelet

Chạy lệnh:

```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```


### Master Node

Cài đặt Kubernetes Cluster với network plugin **Calico**

SSH đến VPS Master và chạy lệnh sau để khởi tạo node Master

```
kubeadm init --pod-network-cidr=192.168.0.0/16
```
Đợi quá trình khởi tạo hoàn tất, chạy lệnh sau:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Cài đặt plugin Calico:

```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Chạy lệnh sau để lấy **join-command**
```
kubeadm token create --print-join-command
```

Master node sẽ trả về 1 join command dùng để chạy trên Worker để tham gia vào cluster

```
root@master:~# kubeadm token create --print-join-command
kubeadm join 10.148.0.6:6443 --token o7gqha.c89gpgn8kt3jgckt     --discovery-token-ca-cert-hash sha256:6401eb108c8d75e558dd2d6207b1d88b1eca8602d8e7eea71f08da2643a27f19 
root@master:~# 

```

### Worker Node

SSH đến Worker Node và nhập lệnh join command từ Master Node trước đó

Đợi tầm vài phút, chạy lệnh sau trên Master Node để kiểm tra các Worker Node đã tham gia cluster chưa:
```
root@master:~# kubectl get nodes
NAME      STATUS   ROLES    AGE    VERSION
master    Ready    master   6d6h   v1.19.2
worker1   Ready    <none>   6d5h   v1.19.2
worker2   Ready    <none>   6d5h   v1.19.2
root@master:~# 
```

### Rancher Node

SSH đến Rancher Node và chạy lệnh sau để cài đặt Rancher:
```
docker run -d --restart=unless-stopped \
    -p 80:80 -p 443:443 \
    -v /opt/rancher:/var/lib/rancher \
    rancher/rancher:latest
```

Đợi vài phút để cài đặt xong. Truy cập vào giao diện web của Rancher đã cài đặt thông qua IP của node Rancher. Đăng nhập với **username** và **password** là **admin**

![Screenshot from 2020-10-10 22-55-32](https://user-images.githubusercontent.com/32956424/95659517-d04b9880-0b4b-11eb-910b-8d3456fdd968.png)

Thêm K8s cluster vào Rancher

![Screenshot from 2020-10-10 22-58-11](https://user-images.githubusercontent.com/32956424/95659567-23255000-0b4c-11eb-838d-0ba3bd881fa6.png)


Chạy các lệnh trên node Master để import cluster vào Rancher

![Screenshot from 2020-10-10 22-58-21](https://user-images.githubusercontent.com/32956424/95659575-291b3100-0b4c-11eb-8e9e-de49a6de90e2.png)




## 3. Cài đặt Prometheus và Grafana

### Helm

Helm là package manager của Kubernetes, cài đặt với Helm sẽ đơn giản hơn thay vì viết từng file yaml

#### Cài đặt Helm 

Trên Master node, chạy lệnh sau để cài đặt Helm

```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

```
Kiểm tra version của Helm bằng lệnh

```
helm version
```

### Cài đặt Prometheus và Grafana bằng Helm

Thêm repo 

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
```
Cài đặt chart 

```
helm install prome-grafana prometheus-community/kube-prometheus-stack
```

### Expose Service

Để sử dụng được dịch vụ của Prometheus và Grafana, cần expose dịch vụ ra ngoài internet. Sử dụng Node Port

Dùng lệnh get pod để xem các pod của Promthes chạy trên node nào

```
root@master:~# kubectl get pods --namespace monitoring -o wide
NAME                                                     READY   STATUS    RESTARTS   AGE   IP                NODE      NOMINATED NODE   READINESS GATES
alertmanager-prome-grafana-kube-prometh-alertmanager-0   2/2     Running   4          33h   192.168.235.148   worker1   <none>           <none>
prome-grafana-7c7fc9ddff-vhsl5                           2/2     Running   4          33h   192.168.189.92    worker2   <none>           <none>
prome-grafana-kube-prometh-operator-fd6495577-bqjfj      2/2     Running   5          33h   192.168.235.147   worker1   <none>           <none>
prome-grafana-kube-state-metrics-7958b79b99-xzn9q        1/1     Running   3          33h   192.168.189.90    worker2   <none>           <none>
prome-grafana-prometheus-node-exporter-98jbw             1/1     Running   2          33h   10.148.0.8        worker1   <none>           <none>
prome-grafana-prometheus-node-exporter-lckzr             1/1     Running   2          33h   10.148.0.6        master    <none>           <none>
prome-grafana-prometheus-node-exporter-r5b49             1/1     Running   2          33h   10.148.0.7        worker2   <none>           <none>
prometheus-prome-grafana-kube-prometh-prometheus-0       3/3     Running   7          33h   192.168.189.91    worker2   <none>           <none>
root@master:~# 
```

Dùng lệnh get service để xem các dịch vụ

```
root@master:~# kubectl get svc --namespace monitoring -o wide
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   33h   app=alertmanager
prome-grafana                             ClusterIP    10.96.188.209    <none>        80:30988/TCP                 33h   app.kubernetes.io/instance=prome-grafana,app.kubernetes.io/name=grafana
prome-grafana-kube-prometh-alertmanager   ClusterIP    10.108.117.156   <none>        9093:30686/TCP               33h   alertmanager=prome-grafana-kube-prometh-alertmanager,app=alertmanager
prome-grafana-kube-prometh-operator       ClusterIP   10.98.19.54      <none>        8080/TCP,443/TCP             33h   app=kube-prometheus-stack-operator,release=prome-grafana
prome-grafana-kube-prometh-prometheus     ClusterIP    10.102.234.129   <none>        9090:32082/TCP               33h   app=prometheus,prometheus=prome-grafana-kube-prometh-prometheus
prome-grafana-kube-state-metrics          ClusterIP   10.97.255.190    <none>        8080/TCP                     33h   app.kubernetes.io/instance=prome-grafana,app.kubernetes.io/name=kube-state-metrics
prome-grafana-prometheus-node-exporter    ClusterIP   10.107.59.94     <none>        9100/TCP                     33h   app=prometheus-node-exporter,release=prome-grafana
\prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     33h   app=prometheus
root@master:~# 
```





## 4. Monitoring
