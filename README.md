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

SSH đến node Master

## 3. Cài đặt Prometheus và Grafana
