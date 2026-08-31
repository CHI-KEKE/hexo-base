---
title: WAF
date: 2026-04-10 07:59:11
categories: 蒼き盾
top_img: https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
tags:
    - 蒼き盾
toc:
toc_number:
comments :
---

{% tabs WAF%}

<!-- tab WAF 動作模式 count block-->

count 跟 block 是兩種不同的 WAF 動作模式


## count 模式

意思是規則有命中，會記錄、會統計、會看到 log，但不會真的擋掉請求，用途通常是上線前先觀察規則準不準、看看會不會誤傷正常流量、蒐集資料後再決定要不要正式攔，可以把它理解成先裝監視器，不急著鎖門，因為規則只要看錯一點點，block 擋掉的就可能是自己人，所以很多團隊都會先 count，等確認沒誤殺再下手



## block 模式

意思是規則有命中，請求會被直接拒絕或攔下，流量進不到後面的服務，用途通常是已經確認這類流量就是不該進來，規則驗證過了，要正式啟用防護可以把它理解成監視器看夠了，現在直接鎖門


![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/3_count_block.png)


如果一開始就 block，風險是

- 合法來源也被擋掉
- 第三方串接突然失敗
- 測試人員以為系統掛了，其實是 WAF 擋掉
- 排查會變麻煩，因為問題不是程式本身，而是入口規則


![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/4_why_not_block_init.png)



所以常見流程會是先掛規則、設成 count、觀察命中來源、修正白名單或條件、再切成 block



## 情境

QA 環境只給公司內部測試，公司有一個 QA 網站：tw-qa-shop.example.com

這個環境平常只有

- 工程師測試
- QA 驗證
- PM 看功能


![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/6_qa_normal.png)


照理說只有公司內網或 VPN 才能打，但發現最近晚上常常有奇怪請求

- 一直掃 /admin
- 一直打 /login
- 嘗試一些不存在的 API 路徑

這時你加了一條 WAF 規則：來源不是公司內網或 VPN 就命中，先用 count

看了兩天 log 後發現

- 大部分是國外掃描 IP
- 有少數是公司同事在家裡直接連 QA，沒走 VPN
- 也有一台自動化測試機器在雲端跑，IP 沒加進白名單


![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/7_what_count_see.png)



這時如果一開始就 block

- 同事會突然進不去 QA
- 自動化測試會失敗
- 大家會以為系統壞掉

所以先修規則

- 要求同事走 VPN
- 把測試機 IP 加到白名單
- 再切成 block

都確認好之後，才改成 block

這就是最典型的 count → block 流程



![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/5_count_to_block.png)



<!-- endtab -->

<!-- tab CDN / WAF 流量管控策略-->


## CDN / WAF 流量管控策略


避免 QA 環境 CDN 背後的服務收到 非允許來源 的流量，會將所有 QA 環境 的 CDN 都掛上 WAF

大 QA (tw-qa) 與 QA 14 配合全家串接，會掛上允許第三方 IP 的 WAF 規則
其他 QAn 環境因只有公司內部開發用，會掛上通用 WAF 規則



![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/8_qa_cdn_split.png)


<!-- endtab -->


<!-- tab 檢查網路的方式-->

1. 手動 health-check


```bash
curl http://notification-template-api-internal.qa1.hk.91dev.tw/_hc
```

2. nslookup

```bash
nslookup notification-template-api-internal.qa1.hk.91dev.tw
```

<!-- endtab -->


<!-- tab 檢查 ip-->

https://www.abuseipdb.com/check/202.182.127.147
https://www.virustotal.com/gui/ip-address/202.182.127.147

https://ipinfo.io/202.182.127.147?lookup_source=search-bar



![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/9_tools.png)


<!-- endtab -->


<!-- tab Web ACL-->


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/WebACL.png)


網頁存取控制清單

它是 AWS WAF 裡面用來保護網站或 API 流量的規則集合，常拿來擋這些東西

- SQL Injection
- XSS
- 惡意 Bot
- 異常請求
- 特定 IP / 國家來源流量



這個 WAF 規則目前綁定到哪些 AWS 資源上

資源名稱：private-proxy-eks-ingress-alb
資源類型：Application Load Balancer
區域：Asia Pacific (Tokyo)

這個名叫 TW-Prod-Regional-NKP-Private-Ingress-WebACL 的 WAF 防護規則，目前掛在一台 ALB（Application Load Balancer） 上，那台 ALB 名叫 private-proxy-eks-ingress-alb



![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/1_web_acl.png)




<!-- endtab -->



<!-- tab proxy 的 WAF-->

504 error 畫面


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/proxy-WAF.png)


看起來... 是 proxy 的 WAF 擋到了

掛在 proxy 前面的流量檢查器，用來先過濾危險請求，再把正常流量放進去，因為 proxy 是流量入口，入口不先擋，後面的服務就得自己扛風險、扛垃圾流量、扛攻擊。

1. 使用者或外部系統送出 HTTP/HTTPS 請求
2. 請求先到 ALB / Ingress，也就是你這邊的 proxy 入口
3. 因為這個入口綁了 WAF Web ACL，所以請求會先被 WAF 檢查
4. WAF 會依照規則判斷這個請求要不要放行
5. 如果符合惡意特徵，像是 SQL Injection、XSS、異常路徑、惡意 IP，WAF 就會擋掉。如果看起來正常，才會繼續往後送到 EKS 裡面的 service / pod



![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/WAF/2_504.png)


<!-- endtab -->



{% endtabs %}