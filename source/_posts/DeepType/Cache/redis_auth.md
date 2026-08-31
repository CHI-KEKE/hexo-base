---
title: Redis 的存取控制（Authentication 與 ACL）
date: 2026-01-17 11:19:50
categories: 落葉下的存檔
top_img: https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---

{% tabs Cache - Redis 的存取控制（Authentication 與 ACL）%}

<!-- tab 存取控制-->

![存取控制](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_landing.png)

Redis 的權限設定，本質是在「界定誰可以對資料做哪些事」，避免任何能連線的人都變成系統管理員，目的是為了防止「連線成功 = 全權控制」這種過於危險的設計被直接暴露在現實世界，尤其 Redis 常用於儲存重要的暫存資料、session、token 等，一旦外洩後果嚴重，在預設情況下，啟用的是 6379 port，若未設定密碼（requirepass）或防火牆規則，任何人只要知道伺服器 IP，就可以直接連線存取資料
存取控制（Access Control）

- 最小授權原則（Least Privilege）被忽略：沒有設定 ACL 或密碼等限制，導致所有人都能以最高權限操作 Redis
- 資料與系統間的橋樑風險：攻擊者一旦能控制 Redis，就能利用 Redis 的指令把惡意內容寫進 .ssh 資料夾，進而用 SSH 登入伺服器，取得完整控制權

<!-- endtab -->

<!-- tab 情境 A：故意做一個「未設密碼、可直接連線」的 Redis-->

這個階段會綁定 0.0.0.、關掉 protected-mod、不設密碼，Redis 在「完全未設防」狀態下，任何能連到它的人都等同管理者

![no_auth](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth_1.png)

<br>

#### 1-1. 啟動不設密碼的 Redis

為了可以直接從本機連得到，我們會讓它監聽 0.0.0.0，並關掉 `protected-mode`
```bash
docker run -d --name redis-open -p 6379:6379 redis:7 redis-server --bind 0.0.0.0 --protected-mode no
```

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test1_1.png)

<br>

#### 1-2. 非破壞性檢查：PING

接著做內部健檢，用 docker exec 進入啟動的 Redis 容器，然後用 redis-cli 驗證服務是否正常運作，直接從服務內部驗證 Redis 是否真的活著，排除「服務根本沒起來」這種低階錯誤，先確認 Redis 正常，再談安全性，不然後面全是假象
```bash
docker exec -it redis-open redis-cli PING

## PONG
##（代表不用 AUTH 就能打指令）
```

重點來了，我們起一個新的 container，內部是 Reids-cli 連線端，用來模擬外部連線
```bash
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 PING

## PONG
```

`-h host.docker.internal` 是整行指令的靈魂，`host.docker.internal`，是 Docker 提供的特殊 DNS 名稱，讓 container 可以找到「宿主機本身」，所以我們是在 container 裡，回頭打 host 的 6379 port

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test1_2.png)

<br>

#### 1-3. 檢查 requirepass

一樣用「另一個容器」讀取 Redis 的密碼設定（requirepass），站在外部使用者的角度，確認 Redis 現在是不是完全沒設防，這個 CONFIG GET requirepass 就是在問 Redis：「你現在有沒有設定連線密碼？」，如果有，會回傳密碼，如果沒有，會回空值，如果能成功執行，表示已經被 Redis 視為「管理者」了!
```bash
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 CONFIG GET requirepass

## requirepass

## 如果有設密碼
## 1) "requirepass"
## 2) "my-real-plain-text-password"
```

![取得管理權限](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test1_3.png)

<br>

#### 1-4. 嘗試用錯密碼 AUTH（驗證「是否強制」）

```bash
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 AUTH wrongpassword

# ERR AUTH <password> called without any password configured for the default user. Are you sure your configuration is correct
```
重點是它不會變成「需要密碼才能操作」，因為根本沒設定密碼!

<br>

#### 為什麼不用本機的 redis-cli 就好？

因為本機能連，不代表「跑在容器裡的服務」也能連，container 有自己的 network namespace、DNS、路由、firewall 都可能不同，這行指令是在測如果是在一個容器，連不連得到，但host.docker.internal 的限制與陷阱，但 host.docker.internal 是為了開發方便而生，不代表跨平台一定能 work


現在我們用 Docker 起了一個 Container，內部住著一座 Redis，這個實驗處理的是 `-p 6379:6379` + Redis 開放這件事，如果真的要對外開放，我們的宿主機也要開放對外的 pulic IP，並且讓 6379 這個 port 可以放行 

<!-- endtab -->

<!-- tab 情境 B：設 requirepass-->

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth_test2_4.png)

#### 2-1. 停掉前一個容器
```bash
docker rm -f redis-open
```

<br>

#### 2-2. 啟動有密碼的 Redis

```bash
docker run -d --name redis-pass -p 6379:6379 redis:7 redis-server --bind 0.0.0.0 --protected-mode no --requirepass "MyStrongPass123"
```

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test2_1.png)

<br>

#### 2-3. 不帶密碼先 PING（應該被擋）

```bash
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 PING

## NOAUTH Authentication required.
```

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test2_2.png)

<br>

#### 2-4. 正確 AUTH 後再 PING

```bash
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 -a "MyStrongPass123" PING

# Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
# PONG
```

<br>

#### 2-5. 這時候 CONFIG GET requirepass（要帶密碼才查得到）

```bash
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 -a "MyStrongPass123" CONFIG GET requirepass

# Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
# requirepass
# MyStrongPass123
```

<!-- endtab -->

<!-- tab 情境 C：Redis 6+ ACL（ACL USERS / ACL GETUSER default）-->

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test3_1.png)

#### 3-1. 先用密碼進去看現有 users

```bash
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 -a "MyStrongPass123" ACL USERS

# default
```

<br>

#### 3-2. 看 default user 權限

```bash
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 -a "MyStrongPass123" ACL GETUSER default

# flags
# on
# sanitize-payload
# passwords
# 22ed3807425be27ec599a9212e4f7f6f57b4c8f0a26e5fac26e3e862904ad5eb
# commands
# +@all
# keys
# ~*
# channels
# &*
# selectors
```

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test3_2.png)

<br>

#### 3-3. 建一個「只允許 PING / GET / SET」的低權限 user

```bash
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 -a "MyStrongPass123" ACL SETUSER appuser on -@all +ping +get +set "~*" ">appPass!123"

# Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
# OK

docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 -a "MyStrongPass123" ACL GETUSER appuser


# flags
# on
# sanitize-payload
# passwords
# 7a97f6078acd1f3f8aef6366bb447b228f7df8415978f3070db610a01b0fc319
# commands
# -@all +ping +get +set
# keys
# ~*
# channels

# selectors
```

-@all 先全部禁止，再用 +ping +get +set 放行你要的指令

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test3_3.png)

<br>

#### 3-4. 用 appuser 登入測試

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test3_5.png)

```bash
# 測試 PING
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 --user appuser --pass "appPass!123" PING

# PONG

#測試危險指令
docker run --rm redis:7 redis-cli -h host.docker.internal -p 6379 --user appuser --pass "appPass!123" CONFIG GET requirepass

# NOPERM User appuser has no permissions to run the 'config|get' command
```

![redis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__test3_4.png)

<br>

#### 清理（避免你之後本機 6379 被佔用）

```bash
docker rm -f redis-pass
```

<!-- endtab -->

<!-- tab 結果整理-->

![con](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Redis/redis_auth__conclude.png)

<!-- endtab -->

{% endtabs %}