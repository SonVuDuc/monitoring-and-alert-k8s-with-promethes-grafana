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


![Screenshot from 2020-10-10 22-58-21](https://user-images.githubusercontent.com/32956424/95659575-291b3100-0b4c-11eb-8e9e-de49a6de90e2.png)




## 3. Cài đặt Prometheus và Grafana

### Helm

### Expose Service

## 4. Monitoring
