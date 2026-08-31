---
title: Helm
date: 2026-05-09 17:25:00
categories: CI / CD
top_img: https://github.com/CHI-KEKE/pics/blob/main/CICD/feather.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/CICD/feather.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs Helm%}


<!-- tab Helm-->


Helm 是 Kubernetes 的「部署打包工具」，它把一堆 Kubernetes YAML 模板，加上環境的設定值，組成真正可以部署的 Kubernetes 資源。也就是，Kubernetes 真正看得懂的是 Deployment、Service、Ingress、ConfigMap 這些 YAML，而 Helm 幫我們產生這些 YAML


![F](https://github.com/CHI-KEKE/pics/blob/main/CICD/Helm/2_what_is_helm_for.png?raw=true)



Helm 是為了避免每個環境都手刻一大堆 Kubernetes YAML；它讓你用同一套部署模板，搭配不同 values，部署到 QA、Prod、TW、HK 等不同地方。可以把 Helm 想成 `template + values = Kubernetes YAML`，也就是說 `部署模板 + 環境設定 = 真正要丟給 Kubernetes 的部署檔`

假設沒有 Helm，要部署一個 API 到 Kubernetes，可能要自己寫很多檔案，而且 QA 一份、Prod 一份、TW 一份、HK 一份

```bash
deployment.yaml
service.yaml
ingress.yaml
configmap.yaml
secret.yaml
canary.yaml
```

久了會變成

```bash
deployment-tw-qa.yaml
service-tw-qa.yaml
ingress-tw-qa.yaml

deployment-tw-prod.yaml
service-tw-prod.yaml
ingress-tw-prod.yaml

deployment-hk-qa.yaml
service-hk-qa.yaml
ingress-hk-qa.yaml
```


![F](https://github.com/CHI-KEKE/pics/blob/main/CICD/Helm/1_duplicated_job.png?raw=true)



Helm 的做法是把它拆成兩種東西

1. templates - 固定的部署模板
2. values.yaml - 每個環境不同的設定值


例如 template 裡可能寫

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
namespace: "{{ .Values.namespace }}"
```

但這不是 Kubernetes 最終 YAML，只是 Helm template


然後 values 裡寫

```yaml
namespace: qa-ai-code-review
image:
  repository: docker-dev.build.91app.io/91app/nine1-ai-code-review-api
  tag: vLATEST-p-e5cceccd-dev-b20260510-1435
```


Helm 會把它們合在一起，變成 Kubernetes 看得懂的樣子

```yaml
image: "docker-dev.build.91app.io/91app/nine1-ai-code-review-api:vLATEST-p-e5cceccd-dev-b20260510-1435"
namespace: "qa-ai-code-review"
```

所以 Helm 在做的事情是把有變數的部署模板，變成真正的 Kubernetes YAML，然後部署到 Kubernetes



![F](https://github.com/CHI-KEKE/pics/blob/main/CICD/Helm/3_helm_chart.png?raw=true)


<!-- endtab -->


<!-- tab helm 實例-->


![F](https://github.com/CHI-KEKE/pics/blob/main/CICD/Helm/5_flow.png?raw=true)


例如 pipeline 前期會做 build 並產生 Docker image：vLATEST-p-e5cceccd-dev-b20260510-1435

接著 deploy job 會拿到這些資訊

```bash
namespace: qa-ai-code-review
release name: nine1-ai-code-review-api
image tag: vLATEST-p-e5cceccd-dev-b20260510-1435
values file: values-tw-qa.yaml
APP_ROLE: api
```

然後 pipeline 會拿公司共用的 API 部署模板複製到你我們的 charts/nine1-ai-code-review-api 裡

```bash
cp -r n1_pipelines/assets/helmcharts/api/templates charts/nine1-ai-code-review-api
```


接著 Helm 出場，他拿這些資訊組出 Kubernetes YAML

```bash
charts/nine1-ai-code-review-api/templates

+

charts/nine1-ai-code-review-api/values-tw-qa.yaml

+

APP_VERSION / CONFIG_VERSION / RELEASE_NAME

```


最後執行 helm upgrade

```bash
helm upgrade --install nine1-ai-code-review-api ./charts/nine1-ai-code-review-api \
  -n qa-ai-code-review \
  -f values-tw-qa.yaml
```

這邊意思是請 Helm 部署 **nine1-ai-code-review-api**，如果 Kubernetes 裡已經有這個 release，就更新它。如果還沒有，就第一次安裝它。這次部署到 **qa-ai-code-review namespace**，而這次使用 **values-tw-qa.yaml** 的設定。


![F](https://github.com/CHI-KEKE/pics/blob/main/CICD/Helm/6_upgrade_install.png?raw=true)


<!-- endtab -->


<!-- tab Helm 與 Kubernetes-->


Kubernetes 是真正跑服務的地方，Helm 是整理和送出部署檔的工具


Docker image → 你的程式被包成可以執行的 image

Kubernetes → 負責把 image 跑起來，變成 Pod / Service / Ingress

Helm → 負責產生 Kubernetes YAML，並送去 Kubernetes


所以真正跑 API 的是 Kubernetes 裡面的 Pod


![F](https://github.com/CHI-KEKE/pics/blob/main/CICD/Helm/4_docker_helm_k8s.png?raw=true)


<!-- endtab -->


<!-- tab Helm chart-->

Helm chart 就是一包部署模板，裡面放著 Chart.yaml、values.yaml、templates。一個 Helm chart 大概長這樣，Helm 就是拿這包 chart 去部署

```yaml
charts/nine1-ai-code-review-api/
  Chart.yaml ## 這包 Helm chart 的基本資料
  values-tw-qa.yaml ## TW QA 這次部署要用的設定值
  templates/ ## Kubernetes YAML 模板
    deployment.yaml
    service.yaml
    ingress.yaml
    canary.yaml
```



<!-- endtab -->


<!-- tab values.yaml-->

values.yaml 是這次部署的變數表，它決定同一套 template 在不同環境會長成什麼樣。例如 QA 跟 Prod 都用同一套 template，但 values 不同

```bash
## QA

namespace = qa-ai-code-review
replicas = 1
image tag = dev 版本
domain = qa.91dev.tw

## Prod

namespace = prod-ai-code-review
replicas = 3
image tag = release 版本
domain = 正式站 domain
```

所以不用複製兩份 deployment.yaml，只要換 values

<!-- endtab -->


<!-- tab templates-->

templates 是還沒填值的 Kubernetes YAML，Helm 會把 values 塞進去，template 可能長這樣

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.fullnameOverride }}
spec:
  template:
    spec:
      containers:
        - name: {{ .Values.app.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

它裡面的 `.Values.xxx` 就是等 Helm 填值的地方

<!-- endtab -->

<!-- tab helm upgrade --install-->


## upgrade


upgrade 是把已經存在的 Helm release 更新成新的 chart / values / image 版本，例如原本 Kubernetes 跑的是 image tag = v1 這次 pipeline build 出  image tag = v2，helm upgrade 就會把 Deployment 更新成 v2

## --install


--install 是補保險，如果 release 不存在，就不要報錯，直接幫我建立新的，如果只跑

```bash
helm upgrade nine1-ai-code-review-api ./charts/nine1-ai-code-review-api
```

但這個 release 第一次部署，還不存在，Helm 會失敗。

加上 `--install` 就變成第一次部署也可以跑、第 N 次部署也可以跑


<!-- endtab -->


<!-- tab kubectl apply-->


![F](https://github.com/CHI-KEKE/pics/blob/main/CICD/Helm/7_kubectl_apply.png?raw=true)


kubectl apply 是直接套 Kubernetes YAML，而 helm upgrade --install 是先用 chart + values 產生 YAML，再幫我們管理版本。Helm 多了幾個能力

- template render
- values 管理
- release name
- revision
- rollback
- upgrade history

所以 log 會看到 REVISION: 8 這是 Helm 記的部署版本


helm upgrade --install 就是讓 Helm 幫你把 chart + values 變成 Kubernetes YAML，然後部署到 K8s，release 存在就更新，不存在就建立

<!-- endtab -->


<!-- tab 整條 pipeline-->

```bash
GitLab Pipeline
│
├─ pipeline-requirements:prepare
│   └─ 下載公司共用 pipeline：n1_pipelines
│
├─ version:prepare
│   └─ 產生版本資訊：
│      - service version = LATEST-p-1da13d95
│      - config version = LATEST-p-1da13d95
│      - build id = 20260508-0622
│
├─ qa:config:build
│   └─ 產生 QA config artifact
│
├─ qa:config:validate
│   └─ 驗證 config，發現缺 Dify，但 job 仍成功
│
├─ dev:build
│   └─ build application image：
│      docker-dev.build.91app.io/91app/nine1-ai-code-review-api:vLATEST-p-1da13d95-dev-b20260508-0622
│
├─ dev:prepare-deploy
│   └─ 產生 Helm deploy job：
│      tw-qa:nine1-ai-code-review-api:deploy
│      extends .dev:helm-deploy
│      variables:
│        - NAMESPACE = qa-ai-code-review
│        - HELM_VALUES_FILE_NAME = values-tw-qa
│        - APP_ROLE = api
│        - APP_VERSION = vLATEST-p-1da13d95-dev-b20260508-0622
│        - CONFIG_VERSION = LATEST-p-1da13d95
│        - RELEASE_NAME = nine1-ai-code-review-api
│
└─ 下一步：
    tw-qa:nine1-ai-code-review-api:deploy
    └─ 真正開始：
       - 使用 kubectl-helm image
       - 複製 Helm templates
       - 補 Chart.yaml
       - 讀 values-tw-qa.yaml
       - 執行 helm upgrade/install
       - Canary 檢查

```

<!-- endtab -->


<!-- tab Summary-->


![F](https://github.com/CHI-KEKE/pics/blob/main/CICD/Helm/helm.png?raw=true)



<!-- endtab -->


{% endtabs %}