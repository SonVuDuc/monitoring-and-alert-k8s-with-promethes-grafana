# Monitoring và Alerting Kubernetes với Prometheus-Grafana

## 1. Yêu cầu

3 VPS hoặc VM:

  + 1 Node Master
  + 2 Node Worker
  + 1 Node Rancher

Cấu hình: 2 CPU, 4GB RAM

Hệ điều hành: Ubuntu 18.04 Server


## 2. Cài đặt Kubernetes Cluster

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

Tạo namespace cho monitoring

```
kubectl create namespace monitoring
```
Cài đặt chart 

```
helm install --namespace monitoring prome-grafana prometheus-community/kube-prometheus-stack
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
prome-grafana                             ClusterIP   10.96.188.209    <none>        80:30988/TCP                 33h   app.kubernetes.io/instance=prome-grafana,app.kubernetes.io/name=grafana
prome-grafana-kube-prometh-alertmanager   ClusterIP   10.108.117.156   <none>        9093:30686/TCP               33h   alertmanager=prome-grafana-kube-prometh-alertmanager,app=alertmanager
prome-grafana-kube-prometh-operator       ClusterIP   10.98.19.54      <none>        8080/TCP,443/TCP             33h   app=kube-prometheus-stack-operator,release=prome-grafana
prome-grafana-kube-prometh-prometheus     ClusterIP   10.102.234.129   <none>        9090:32082/TCP               33h   app=prometheus,prometheus=prome-grafana-kube-prometh-prometheus
prome-grafana-kube-state-metrics          ClusterIP   10.97.255.190    <none>        8080/TCP                     33h   app.kubernetes.io/instance=prome-grafana,app.kubernetes.io/name=kube-state-metrics
prome-grafana-prometheus-node-exporter    ClusterIP   10.107.59.94     <none>        9100/TCP                     33h   app=prometheus-node-exporter,release=prome-grafana
prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     33h   app=prometheus
root@master:~#
```
Có 3 serive cần lưu ý là: 
  + **prome-grafana-kube-prometh-prometheus**
  + **prome-grafana**
  + **prome-grafana-kube-prometh-alertmanager**
  
Lần lượt là service của Prometheus, Grafana và Alert

Chỉnh sửa 3 service bằng lệnh edit svc

```
kubectl edit svc prome-grafana-kube-prometh-prometheus --namespace monitoring
```

```
kubectl edit svc prome-grafana --namespace monitoring
```

```
kubectl edit svc prome-grafana-kube-prometh-alertmanager --namespace monitoring
```

![Screenshot from 2020-10-10 23-43-52](https://user-images.githubusercontent.com/32956424/95660499-7e5a4100-0b52-11eb-9d09-168ba12d5afd.png)

Sửa mục **type** của 3 service từ ClusterIP thành NodePort và lưu lại


Dùng lệnh describe svc để xem NodePort của service đó

```
root@master:~# kubectl describe svc  prome-grafana-kube-prometh-prometheus --namespace monitoring
Name:                     prome-grafana-kube-prometh-prometheus
Namespace:                monitoring
Labels:                   app=kube-prometheus-stack-prometheus
                          app.kubernetes.io/managed-by=Helm
                          chart=kube-prometheus-stack-9.4.10
                          heritage=Helm
                          release=prome-grafana
                          self-monitor=true
Annotations:              field.cattle.io/publicEndpoints:
                            [{"addresses":["10.148.0.7"],"port":32082,"protocol":"TCP","serviceName":"monitoring:prome-grafana-kube-prometh-prometheus","allNodes":tru...
                          meta.helm.sh/release-name: prome-grafana
                          meta.helm.sh/release-namespace: monitoring
Selector:                 app=prometheus,prometheus=prome-grafana-kube-prometh-prometheus
Type:                     NodePort
IP:                       10.102.234.129
Port:                     web  9090/TCP
TargetPort:               9090/TCP
NodePort:                 web  32082/TCP
Endpoints:                192.168.189.91:9090
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
root@master:~# 

```
NodePort của service **prome-grafana-kube-prometh-prometheus** là 32082

```
root@master:~# kubectl describe svc  prome-grafana --namespace monitoring
Name:                     prome-grafana
Namespace:                monitoring
Labels:                   app.kubernetes.io/instance=prome-grafana
                          app.kubernetes.io/managed-by=Helm
                          app.kubernetes.io/name=grafana
                          app.kubernetes.io/version=7.2.0
                          helm.sh/chart=grafana-5.6.12
Annotations:              field.cattle.io/publicEndpoints:
                            [{"addresses":["10.148.0.7"],"port":30988,"protocol":"TCP","serviceName":"monitoring:prome-grafana","allNodes":true}]
                          meta.helm.sh/release-name: prome-grafana
                          meta.helm.sh/release-namespace: monitoring
Selector:                 app.kubernetes.io/instance=prome-grafana,app.kubernetes.io/name=grafana
Type:                     NodePort
IP:                       10.96.188.209
Port:                     service  80/TCP
TargetPort:               3000/TCP
NodePort:                 service  30988/TCP
Endpoints:                192.168.189.92:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
root@master:~# 
```
NodePort của service **prome-grafana** là 30988

```
root@master:~# kubectl describe svc  prome-grafana-kube-prometh-alertmanager --namespace monitoring
Name:                     prome-grafana-kube-prometh-alertmanager
Namespace:                monitoring
Labels:                   app=kube-prometheus-stack-alertmanager
                          app.kubernetes.io/managed-by=Helm
                          chart=kube-prometheus-stack-9.4.10
                          heritage=Helm
                          release=prome-grafana
                          self-monitor=true
Annotations:              field.cattle.io/publicEndpoints:
                            [{"addresses":["10.148.0.7"],"port":30686,"protocol":"TCP","serviceName":"monitoring:prome-grafana-kube-prometh-alertmanager","allNodes":t...
                          meta.helm.sh/release-name: prome-grafana
                          meta.helm.sh/release-namespace: monitoring
Selector:                 alertmanager=prome-grafana-kube-prometh-alertmanager,app=alertmanager
Type:                     NodePort
IP:                       10.108.117.156
Port:                     web  9093/TCP
TargetPort:               9093/TCP
NodePort:                 web  30686/TCP
Endpoints:                192.168.235.148:9093
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
root@master:~# 
```

NodePort của service **prome-grafana-kube-prometh-alertmanager** là 30686

#### Truy cập vào service

Chạy lệnh get pod để xem các service chạy trên Node nào

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

```
Service **prome-grafana-kube-prometh-prometheus** có Endpoint là 192.168.189.91:9090 -> node Worker2

Service **prome-grafana** có Endpoint là 192.168.189.92:3000 -> node Worker2

Service **prome-grafana-kube-prometh-alertmanager** có Endpoint là 192.168.235.148:9093 -> node Worker1

Truy cập vào địa chỉ http:// Extenal IP Node:NodePort
  
**Prometheus**

![Screenshot from 2020-10-10 23-57-04](https://user-images.githubusercontent.com/32956424/95660779-5370ec80-0b54-11eb-89cd-e587d6e3284f.png)


**Grafana**

![Screenshot from 2020-10-10 23-55-32](https://user-images.githubusercontent.com/32956424/95660759-1c9ad680-0b54-11eb-8721-c2355e5c6818.png)

**Alert Manager**

![Screenshot from 2020-10-10 23-58-11](https://user-images.githubusercontent.com/32956424/95660791-7a2f2300-0b54-11eb-96f8-f1e6c27a4d57.png)



## 4. Monitoring

Prometheus dùng các trình daemon cài sẵn trên các node để thu thập các thông tin cần thiết, giao tiếp với máy chủ quản lý monitor qua giao thức HTTP/HTTPs và lưu trữ data theo dạng time-series database (TSDB).

Tại Prometheus, người dùng có thể xem xét, kiểm tra các thông số của cluster bằng cách chọn các thông số muốn xem, chọn Excute

![Screenshot from 2020-10-11 00-05-23](https://user-images.githubusercontent.com/32956424/95660954-7e0f7500-0b55-11eb-9af0-006f2625e208.png)

Và có thể hiển thị kết quả dưới dạng biểu đồ 

![Screenshot from 2020-10-11 00-03-39](https://user-images.githubusercontent.com/32956424/95660923-3daff700-0b55-11eb-8237-12b7f71abbe1.png)

Tuy nhiên, sử dụng Grafana sẽ có giao diện đẹp hơn, chia thành nhiều mục để người dùng dễ dàng hơn trong việc đánh giá hiệu suất của cluster. Grafana sẽ lấy dữ liệu metric thu được từ Prometheus, phân tích và tạo ra dashboard mô tả trực quan các thông số như CPU, RAM, Disk, Network...

![Screenshot from 2020-10-11 00-10-27](https://user-images.githubusercontent.com/32956424/95661024-30dfd300-0b56-11eb-949d-e247930aaec9.png)

Thống kê chi tiết các dashboard của Grafana

Người dùng có thể chọn các dashboard của Grafana

![Screenshot from 2020-10-11 00-11-23](https://user-images.githubusercontent.com/32956424/95661040-51a82880-0b56-11eb-87e2-c8ebb9b4bd3c.png)

Tùy chọn chi tiết thời gian cập nhật, node cụ thể, namespace...

![Screenshot from 2020-10-11 00-11-23](https://user-images.githubusercontent.com/32956424/95661040-51a82880-0b56-11eb-87e2-c8ebb9b4bd3c.png)

## 5. Alerting

Alert giúp người dùng xác định những vấn đề bất thường xảy ra trong hệ thống. Bằng cách tùy chỉnh những thay đổi, người dùng có thể giảm thiểu sự gián đoạn trong hệ thống

Alert gồm 2 phần:
  + **Alert Rule**: Là điều kiện để Alert được kích hoạt, Alert Rule được người dùng định nghĩa bằng 1 hoặc nhiều điều kiện
  + **Notification channel**: Cách mà Alert được gửi đi, khi điều kiện của Alert được kích hoạt. Grafana sẽ gửi sẽ gửi thông báo cho channel. Notification channel có thế là E-mail, Telegram, Discord... 
  
Grafana 4.0 trở lên hỗ trợ cấu hình Alert

### Notification Channel Discord 

Discord là hệ thống trò chuyện bằng giọng nói và văn bản, cho phép giao tiếp với những người khác. Bất cứ ai cũng có thể tạo ra một máy chủ lưu trữ thảo luận với mọi người. Discord có mặt trên các hệ điều hành như Window, Ubuntu, Mac OS, Android...

Sau khi đăng ký tài khoản Discord, tiến hành tạo 1 server Discord - đây là nơi mà Alert từ Grafana sẽ gửi về. Đặt tên cho server

![Screenshot from 2020-10-20 13-28-26](https://user-images.githubusercontent.com/32956424/96551290-3c609600-12dc-11eb-822d-fb78b4fa9fa2.png)

Vào **Server Setting** -> **Integration**. Tạo 1 Webhook cho Server 

![Screenshot from 2020-10-20 13-29-09](https://user-images.githubusercontent.com/32956424/96551592-bf81ec00-12dc-11eb-87ec-69fb091c1f20.png)

Đặt tên cho Webhook và copy Webhook URL

![Screenshot from 2020-10-20 14-02-12](https://user-images.githubusercontent.com/32956424/96551680-dfb1ab00-12dc-11eb-8f1a-fa7010cddc49.png)

Truy cập vào phần Alert trên Grafana, chọn **New Channel** để tạo Notification Channel

Đặt tên cho Channel, chọn type Discord, paste Webhook URL của Discord đã copy trước đó vào phần Webhook URL

![Screenshot from 2020-10-20 14-04-33](https://user-images.githubusercontent.com/32956424/96551902-361ee980-12dd-11eb-95a3-78e930ba5ae8.png)

Để kiểm tra Notification có hoạt động hay không, click vào **Test**. Grafana sẽ gửi 1 Alert Test về Discord server

![Screenshot from 2020-10-20 14-07-33](https://user-images.githubusercontent.com/32956424/96552184-9f066180-12dd-11eb-951a-b2e7f2eaf3b0.png)

Test thành công












