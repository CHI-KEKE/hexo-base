---
title: EKS
date: 2026-05-18 08:28:00
categories: kubernetes
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/k8s2.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/k8s2.png
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs EKS%}


<!-- tab EKS-->

Elastic Kubernetes Service 就是 AWS 幫我們 Kubernetes 的核心管理層顧好，讓我們把力氣放在「部署服務」和「管理應用」，不要一開始就被cluster維護、升級、控制平面穩定性卡住。先理解 Kubernetes 是用來管理容器化應用的系統，例如幫你處理服務部署、擴容、重啟、流量導向等事情。而 AWS 上的 Kubernetes 讓我們想在 AWS 裡跑 Kubernetes 時，可以自己從零架一套，也可以使用 AWS 提供的託管服務。Kubernetes 裡最關鍵、最麻煩維護的控制平面，例如 API Server、cluster 狀態管理等，會由 AWS 負責。使用 EKS 時，開發者或 DevOps 團隊通常把重點放在部署 Pod、Service、Ingress、Node、IAM、網路設定與應用維運上。

假設我們有一個 API 服務，原本只跑在一台 EC2 上

當流量變大時，可能會遇到這些問題

- 要怎麼自動開更多服務？
- 某個服務掛掉時，要怎麼自動重啟？
- 要怎麼平滑部署新版本？
- 要怎麼讓不同服務互相溝通？
- 要怎麼管理多台機器上的容器？


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/EKS/1_why_eks.png)




這時 Kubernetes 可以處理這些容器管理問題。自己架 Kubernetes 當然可以，但需要處理很多事情，例如

- 控制平面的高可用
- etcd 備份與還原
- Kubernetes 版本升級
- 憑證管理
- 網路插件設定
- 安全修補
- 故障排查

但如果是使用 EKS 的情境，我們只要告訴 AWS「我要一個 Kubernetes cluster。」AWS 就會建立並管理 Kubernetes 控制平面，接著把自己的應用部署進去 例如：


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
        - name: backend-api
          image: my-backend-api:latest
          ports:
            - containerPort: 3000
```


這段設定的意思是「我要跑 3 份 backend-api 容器，如果有一份掛掉，Kubernetes 要幫我補回來。」而 EKS 則提供這個 Kubernetes 環境，讓這段設定可以在 AWS 上運作，但使用 EKS 之後，團隊仍然需要理解並管理

- Pod 怎麼部署
- Service 怎麼暴露
- Ingress 怎麼接外部流量
- IAM 權限怎麼設
- VPC 和子網路怎麼規劃
- Node Group 怎麼選
- Log 和 Metric 怎麼收
- 成本怎麼控制

所以 EKS 不是「不用懂 Kubernetes」，而是「不用自己從零維護 Kubernetes 控制平面」



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/EKS/2_k8s_control_isdeep.png)


<!-- endtab -->


<!-- tab control management-->


Kubernetes 控制平面主要負責

- 接收 kubectl apply
- 決定 Pod 要排到哪台機器
- 監控 Pod 有沒有掛掉
- 管理 Deployment、Service、Node 等狀態
- 對外提供 Kubernetes API


平常下這種指令背後都要打到 K8S API Server

```bash
kubectl get pods
kubectl apply -f deployment.yaml
kubectl scale deployment api --replicas=5
```


如果自己在 EC2 上架會怎樣控制平面可能長這樣

```bash
EC2-1: api-server
EC2-2: controller-manager
EC2-3: scheduler
```

假設 API Server 那台 EC2 掛掉，可能會遇到 `kubectl get pods` 結果 `Unable to connect to the server`，此時不是應用程式一定掛了，而是「無法控制 Kubernetes」


但若使用 EKS，不用自己決定 API Server 要放幾台、要不要跨 AZ、哪台壞掉要怎麼補、控制平面機器容量不夠怎麼擴


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/EKS/3_aws_is_brain.png)


<!-- endtab -->


<!-- tab etcd 備份與還原-->

etcd 是 Kubernetes 的記憶庫；它壞掉不是少幾個 Pod 而已，而是整個 cluster 可能忘記自己原本長什麼樣子。例如這些東西都存在 etcd 裡

- 有哪些 Deployment
- 每個 Deployment 要幾個 replicas
- 有哪些 Service
- 有哪些 ConfigMap
- 有哪些 Secret
- 哪些 Node 加入cluster
- Pod 目前狀態

例如建立一個 Deployment

```bash
kubectl apply -f api-deployment.yaml
```

Kubernetes 會把「我要 3 個 api Pod」這個期望狀態記進 etcd


## 自己架會怎樣？

自己架 Kubernetes，就要處理：

- etcd 要跑幾台
- etcd 資料怎麼備份
- 備份多久做一次
- 備份放哪裡
- etcd 壞掉怎麼還原
- 還原後資料會不會不一致

假設某天 etcd 資料壞了，你可能會看到 Deployment 查不、Service 查不到、Secret 查不到、cluster 狀態混亂，更慘的是，容器可能還在某些 Node 上跑，但 Kubernetes 已經不知道它們是誰、由誰管理、要不要重建。而 EKS 控制平面包含 etcd cluster，而且 AWS 文件提到 etcd server nodes 也跑在 Auto Scaling Group，並且跨三個 AZ，以提高耐用性，所以你我們不用自己維護控制平面裡的 etcd cluster



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/EKS/4_etcd.png)


<!-- endtab -->

<!-- tab etcd Kubernetes 版本升級-->

Kubernetes 升級最麻煩的地方是要確認舊設定、外掛、API 版本、Node 都不會在升級後爆掉，當一直推出新版本，例如

```bash
1.29
1.30
1.31
1.32
```

每個版本可能會有新功能、安全修補、API 行為改變、舊 API 被移除、元件相容性問題



## 自己架會怎樣？

如果自己架 Kubernetes，升級時要自己處理

- 升級 API Server
- 升級 Controller Manager
- 升級 Scheduler
- 升級 etcd
- 升級 kubelet
- 升級 kube-proxy
- 升級 CNI 外掛
- 確認舊 API 有沒有被移除

假設原本有一個舊版 Ingress

```bash
apiVersion: extensions/v1beta1
kind: Ingress

```


但新版 Kubernetes 已不支援這個 API，升級後可能會發生

- Ingress 無法套用
- 部署流程失敗
- CI/CD pipeline 爆掉
- 外部流量進不來


## EKS

EKS 可以幫我們升級控制平面版本，但自己還是要規劃升級流程。AWS 文件也有明確把升級分成更新控制平面、更新 cluster 元件等步驟，另外，EKS 對 Kubernetes 版本有標準支援與延伸支援週期，官方文件提到標準支援是 14 個月，延伸支援是接著 12 個月，合計最多 26 個月。使用 EKS 時，我們大致會做

1. 檢查目前 cluster 有沒有 deprecated API
2. 升級 EKS control plane
3. 升級 managed node group
4. 升級 add-ons，例如 VPC CNI、CoreDNS、kube-proxy
5. 測試服務是否正常

EKS 幫你省掉的是

1. 自己 SSH 到控制平面機器
2. 自己升級 API Server
3. 自己升級 Scheduler
4. 自己升級 Controller Manager
5. 自己處理控制平面 HA 升級順序

但它不會幫你保證 YAML、Helm chart、應用程式完全相容新版 Kubernetes



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/EKS/5_k8s_update.png)



<!-- endtab -->

<!-- tab 憑證管理-->

Kubernetes 裡很多元件彼此溝通都靠憑證驗身分，憑證過期或配錯，整個 cluster 會像公司門禁壞掉，自己人也進不去


```bash
kubectl -> API Server
kubelet -> API Server
controller-manager -> API Server
scheduler -> API Server
API Server -> etcd
```

它們溝通時會用到 TLS 憑證。可以把它想成每個元件都要拿識別證，才能跟 Kubernetes 大腦講話


## 自己架會怎樣？

如果自己架 Kubernetes，要處理

1. CA 憑證
2. API Server 憑證
3. kubelet client certificate
4. etcd certificate
5. 憑證到期時間
6. 憑證輪替
7. 憑證權限


假設憑證過期，可能會看到

```bash
x509: certificate has expired or is not yet valid
```

然後 `kubectl get nodes` 失敗，Node 也可能無法跟 API Server 回報狀態


## EKS

EKS 管理控制平面，所以控制平面內部元件所需的憑證管理、輪替與維護，大多由 AWS 負責。，我們不用自己登入控制平面主機去更新 API Server 憑證



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/EKS/6_certificate.png)


<!-- endtab -->

<!-- tab 網路插件設定-->


Kubernetes 裡的 Pod 需要網路， Pod 要拿 IP、服務要互通、外部流量要進來，任何一段配錯都會變成很難查的連線問題

例如

- frontend Pod 要打 backend Pod
- backend Pod 要打 database
- 外部使用者要進到 ingress
- Pod 要能連 DNS


在 EKS 裡，常見的網路元件包括：

1. Amazon VPC CNI
2. CoreDNS
3. kube-proxy
4. Security Group
5. Subnet
6. Route Table
7. Load Balancer Controller


其中 CNI 負責讓 Pod 可以拿到網路位址並和 VPC 網路整合


## 自己架會怎樣？

如果自己架 Kubernetes，要自己選 CNI，例如

- Calico
- Cilium
- Flannel
- Weave

然後處理

1. Pod CIDR 怎麼規劃
2. Node 網路怎麼通
3. Service 網路怎麼轉發
4. DNS 怎麼解析
5.  網路政策怎麼套
6. 外部 Load Balancer 怎麼接
7. CNI 版本怎麼升級

網路問題最討厭的是，它常常不是直接噴明顯錯誤，而是可能只會看到

- Pod A curl Pod B timeout
- 某些 Node 上的 Pod 可以連
- 某些 Node 上的 Pod 不行
- DNS 偶爾解析失敗
- 新 Pod 一直 Pending
- EKS 幫你做什麼？

EKS 會提供與 AWS VPC 整合的網路方案，並且有 EKS add-ons 可以管理一些常見元件版本，例如 VPC CNI、CoreDNS、kube-proxy，但我們仍要懂：


- VPC CIDR 夠不夠
- Subnet IP 夠不夠
- Security Group 有沒有放行
- Pod 要不要跨 AZ
- Load Balancer 要放 public 還是 private subnet


例如建立了一個 EKS  cluster ，然後部署很多 Pod，突然新 Pod 都卡在 Pending，查事件看到類似


```bash
failed to assign an IP
failed to assign an IP address to container
```

可能原因是

- Subnet IP 不夠了
- Node 可分配的 Pod IP 滿了
- VPC CNI 設定不適合目前規模

這種情況下，AWS 幫你提供 EKS 網路整合能力，但你的 VPC/Subnet/IP 規劃還是要自己負責，所以這一項比較精準的講法是 AWS 幫我們提供並維護 EKS 常用網路元件的整合方式，但網路架構設計與參數設定仍然是自己的責任


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/EKS/7_network.png)



<!-- endtab -->


<!-- tab summary-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/EKS/8_what_we_do.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/k8s/EKS/9_coopration.png)




<!-- endtab -->


{% endtabs %}



