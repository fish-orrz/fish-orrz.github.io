---
date: '2026-11-14T11:06:41+08:00'
draft: false
title: 'Networkpolicy'
---

## NetworkPolicy 是什麼?

K8s 預設情況下,叢集裡所有 Pod 都可以互相連線,就像一間**沒有隔間、沒有門禁**的辦公室。

**NetworkPolicy 就是 Pod 的門禁規則**:決定「誰可以連進來」和「可以連出去哪裡」。

三個一定要記住的觀念:

1. **沒有任何 NetworkPolicy** → 全部都通(預設)。
2. **Pod 只要被某一條規則選中** → 該方向(進來 / 出去)就變成**全擋**,只有規則裡寫的才放行(也就是白名單)。
3. NetworkPolicy **只有「允許」,沒有「拒絕」**。多條規則是疊加放行,不會互相打架。

> 注意:NetworkPolicy 需要網路外掛(CNI)支援才會生效,例如 Calico、Cilium。如果叢集用的外掛不支援,規則建了也不會有作用。

---

## K8s 網路流量怎麼走(從 LB VIP 到 Pod)

先搞清楚流量走哪一條路,才知道 NetworkPolicy 管的是哪一關。

```text
使用者
  │  https://web.example.com
  ▼
LB VIP(固定入口 IP,例如 192.168.1.100)
  │
  ▼
Node 節點(NodePort,例如 30443)
  │
  ▼
Ingress Controller(Pod,例如 ingress-nginx)  ← 依 Ingress 規則「看網址」分流
  │
  ▼
Service(固定的內部窗口)                     ← 依標籤找到 Pod
  │
  ▼
Pod(role: web)                              ← NetworkPolicy 守在這一關
```

| 關卡 | 白話說明 |
|---|---|
| **LB VIP** | 對外的固定入口 IP,把流量分給多台節點 |
| **Node(節點)** | 跑 Pod 的主機,對外開一個 port(NodePort)接流量 |
| **Ingress Controller** | 像「櫃台」,看網址或路徑,決定交給哪個 Service |
| **Ingress** | 櫃台的「分流規則表」(只是設定,真正執行的是 Controller) |
| **Service** | Pod 前面的固定窗口。Pod 重建後 IP 會變,Service 的位址不變 |
| **Pod** | 真正跑程式的地方 |

> 有些環境的 Ingress Controller 直接佔用節點的 80/443,就不會經過 NodePort,實際以自己的環境為準。

> **NetworkPolicy 管的是最後一關:Pod 的門口。** LB、Ingress、Service 都是「帶路」的,NetworkPolicy 才是 Pod 門口的警衛。Pod 看到的來源,是**直接把流量交給它的那一個**(通常是 Ingress Controller 的 Pod),不是最前面的使用者。

---

## 標籤(label):`role: web` 是什麼意思

標籤就是貼在 Pod 身上的**便利貼**,格式是 `名稱: 值`。

- 名稱和值都是自己取的,K8s 不懂 `role` 或 `web` 的意思,只是拿來「找人」。
- 也可以取 `app: web`、`tier: frontend` 之類的,只要全部統一就好。
- `role: web` 就是「這個 Pod 的角色是 web」。

標籤是在 Deployment 裡面貼上去的:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy              # Deployment 自己的名字
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      role: web                 # Deployment 管理「標籤是 role=web」的 Pod
  template:
    metadata:
      labels:
        role: web               # ★ 真正貼在 Pod 身上的標籤
    spec:
      containers:
        - name: web
          image: nginxinc/nginx-unprivileged:1.27   # 監聽 8080
          ports:
            - containerPort: 8080
```

同一個標籤,會被三個地方拿來「找人」:

| 誰在用 | 寫在哪 | 目的 |
|---|---|---|
| Deployment | `spec.selector` | 管理哪些 Pod |
| Service | `spec.selector` | 流量轉給哪些 Pod |
| NetworkPolicy | `spec.podSelector` / `from.podSelector` | 保護或放行哪些 Pod |

> 標籤要貼在 `template.metadata.labels`(Pod 身上),只貼在 Deployment 最上面的 `metadata` 是沒用的,NetworkPolicy 看的是 Pod 的標籤。

確認標籤:

```bash
kubectl get pods -n demo --show-labels
kubectl get pods -n demo -l role=web
```

---

## 常見的 Service 與 Ingress 設定

### Service:Pod 的固定窗口

Service 用 `selector` 的標籤找到 Pod,再把流量分給它們。三個 port 不要搞混:

- `port`:Service 自己對外的 port
- `targetPort`:Pod 實際監聽的 port
- `nodePort`:節點上開的 port(範圍 30000–32767,只有 NodePort 類型才有)

**ClusterIP(預設):只有叢集內部連得到**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: demo
spec:
  type: ClusterIP
  selector:
    role: web            # 轉給標籤是 role=web 的 Pod
  ports:
    - port: 80           # 叢集內連 web-svc:80
      targetPort: 8080   # 轉給 Pod 的 8080
```

**NodePort:每台節點都開一個 port,外部可用「節點IP:30080」連進來**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  namespace: demo
spec:
  type: NodePort
  selector:
    role: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080    # 不寫會自動配一個
```

**LoadBalancer:向 LB 要一個對外 IP(需要環境有提供 LB,例如 MetalLB、雲端 LB)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  namespace: demo
spec:
  type: LoadBalancer
  selector:
    role: web
  ports:
    - port: 443
      targetPort: 8080
```

| 類型 | 誰連得到 | 常見用途 |
|---|---|---|
| ClusterIP | 叢集內部 | 服務之間互相呼叫(最常用) |
| NodePort | 能連到節點 IP 的人 | 測試,或給外部 LB 當後端 |
| LoadBalancer | 透過外部 IP 的人 | 直接對外提供服務 |

### Ingress:依網址分流的櫃台

一個入口,依網址導到不同 Service,不用每個服務都開一個對外 IP。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: demo
spec:
  ingressClassName: nginx        # 交給哪個 Ingress Controller 處理
  tls:
    - hosts:
        - web.example.com
      secretName: web-tls        # 存放憑證的 Secret
  rules:
    - host: web.example.com      # 網址
      http:
        paths:
          - path: /              # 路徑
            pathType: Prefix
            backend:
              service:
                name: web-svc    # 交給哪個 Service
                port:
                  number: 80
```

> Ingress 只是一份規則,叢集裡要有 Ingress Controller(例如 ingress-nginx)才會真的運作。

---

## 先看一份基本範例

情境:`demo` 這個 namespace 裡有前端 Pod(web)和資料庫 Pod(db),我們只想讓 **web 連到 db 的 3306**,其他人都不行。

回顧一下,Pod 身上貼的標籤(label)就是 NetworkPolicy 找人的依據(`role: web` 的意思見上一節):

```yaml
# 資料庫 Pod 的標籤
labels:
  role: db

# 前端 Pod 的標籤
labels:
  role: web
```

接著是 NetworkPolicy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-web          # ① 這條規則自己的名字(隨便取,方便辨識)
  namespace: demo             # ② 這條規則放在哪個 namespace
spec:
  podSelector:                # ③ 保護誰?(被保護的 Pod)
    matchLabels:
      role: db
  policyTypes:
    - Ingress                 # ④ 管「進來」的流量
  ingress:
    - from:                   # ⑤ 誰可以進來?
        - podSelector:
            matchLabels:
              role: web
      ports:                  # ⑥ 可以連哪個 port?
        - protocol: TCP
          port: 3306
```

### 每個欄位對應什麼

| 編號 | 欄位 | 白話意思 | 範例中的值 |
|---|---|---|---|
| ① | `metadata.name` | 規則的名字,只是個代號,**不影響效果** | `db-allow-web` |
| ② | `metadata.namespace` | 規則生效的 namespace | `demo` |
| ③ | `spec.podSelector` | **被保護的對象**(誰是門內的人) | 標籤 `role: db` |
| ④ | `policyTypes` | 管進來(`Ingress`)還是出去(`Egress`) | `Ingress` |
| ⑤ | `ingress.from` | **允許進來的對象**(誰有門卡) | 標籤 `role: web` |
| ⑥ | `ingress.ports` | 允許連的 port | TCP 3306 |

### 最容易搞混的地方

- ①**規則名字**、③**被保護的 Pod**、⑤**允許的來源**是三件不同的事,不要混在一起。
- NetworkPolicy **不認 Pod 名字,也不認 Deployment 名字,只認標籤(label)**。
- `podSelector` 在 ③ 的意思是「保護誰」,在 ⑤ 的意思是「放行誰」,位置不同意思不同。

### 這份規則用白話念一遍

> 在 `demo` 裡,標籤是 `role=db` 的 Pod,只接受標籤是 `role=web` 的 Pod 連進來,而且只能連 TCP 3306,其他進來的全部擋掉。

---

## 同一個 namespace 的常見設定

以下範例都放在 `demo` 這個 namespace。

### 1. 全擋(擋所有進來的流量)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: demo
spec:
  podSelector: {}        # {} 代表「這個 namespace 的所有 Pod」
  policyTypes:
    - Ingress            # 只寫了 Ingress,卻沒寫任何 ingress 規則 = 一個都不放行
```

### 2. 全擋(進來、出去都擋)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: demo
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### 3. 全通(允許所有進來的流量)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: demo
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - {}                 # 一條空規則 = 誰都可以進來
```

> 沒有建任何 NetworkPolicy 本來就是全通。這份通常用在「想明確寫出全通」的時候。

### 4. 白名單(只放行指定的對象)

就是最前面的 `db-allow-web` 範例。想放行多個來源,在 `from` 底下多加幾項即可:

```yaml
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: web
        - podSelector:
            matchLabels:
              role: batch      # 另一種 Pod 也允許
      ports:
        - protocol: TCP
          port: 3306
```

### 5. 只允許同一個 namespace 內互通

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: demo
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}    # 同 namespace 的所有 Pod;其他 namespace 的進不來
```

### 6. 限制「出去」的流量(Egress 白名單)

情境:web Pod 只能連 db 的 3306,其他都不准連。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-egress-limit
  namespace: demo
spec:
  podSelector:
    matchLabels:
      role: web
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              role: db
      ports:
        - protocol: TCP
          port: 3306
    - ports:                 # 一定要放行 DNS,否則用網域名稱連線會失敗
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

> 擋「出去」時,最常忘記的就是 **DNS(port 53)**。忘了的話,Pod 連服務名稱會直接失敗。

### 7. 只允許特定 IP 範圍(ipBlock)

來源不是 Pod,而是公司內網或特定機器時,用 IP 範圍來放行:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-office-ip
  namespace: demo
spec:
  podSelector:
    matchLabels:
      role: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 10.10.0.0/16        # 允許這個網段
            except:
              - 10.10.5.0/24          # 但這個子網段除外
      ports:
        - protocol: TCP
          port: 8080
```

### 建議的做法

先建一份**全擋**(上面第 1 或第 2 份),再一條一條開**白名單**。這樣最安全,也最容易看出誰被開了哪些權限。

---

## 跨 namespace:ns1 要連 ns2 的 443

情境:`ns1` 裡的 Pod 要連 `ns2` 裡的服務,只開 **TCP 443**,其他 port 都不行。

跨 namespace 要想成**兩道門**:

- **ns2 這一側(門的裡面)**:要允許 ns1 進來。**這份一定要建**。
- **ns1 這一側(門的外面)**:如果 ns1 有擋「出去」的流量,還要允許它出去到 ns2。**沒擋出去就不用建**。

### ns2:允許 ns1 連進來(必要)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ns1-443
  namespace: ns2                 # 規則放在被連線的那一側
spec:
  podSelector: {}                # ns2 的所有 Pod;只想開特定服務就改成 matchLabels
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:     # 用「標籤」找 namespace
            matchLabels:
              kubernetes.io/metadata.name: ns1
      ports:
        - protocol: TCP
          port: 443
```

namespace 也是靠標籤來辨識。K8s 會自動幫每個 namespace 貼上 `kubernetes.io/metadata.name=<namespace 名稱>`,所以直接用就好。可以用這個指令確認:

```bash
kubectl get ns ns1 --show-labels
```

### ns1:允許連出去到 ns2(ns1 有擋 Egress 才需要)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-ns2-443
  namespace: ns1
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ns2
      ports:
        - protocol: TCP
          port: 443
    - ports:                     # DNS 一樣要放行
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

> 小提醒:`port` 寫的是 **Pod 實際監聽的 port**,不是 Service 對外的 port。如果 Service 是 443 轉到 Pod 的 8443,規則要寫 8443。

### 想再縮小:只允許 ns1 的特定 Pod

差一個 `-`,意思完全不同:

```yaml
# 「且」:必須是 ns1 裡面、而且標籤是 app=client 的 Pod
- from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ns1
      podSelector:
        matchLabels:
          app: client
```

```yaml
# 「或」:ns1 的所有 Pod,或是「本 namespace」裡標籤是 app=client 的 Pod
- from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ns1
    - podSelector:
        matchLabels:
          app: client
```

第二種寫法會開得比預期還大,是初學者最常踩的坑。

---

## 實際情境:全擋之後,網站為什麼打不開?

`demo` 做了「全擋」之後,使用者從 Ingress 進來的流量也會被擋。因為對 Pod 來說,**來源是 Ingress Controller 的 Pod**(在另一個 namespace),必須明確放行它。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-allow-ingress-controller
  namespace: demo
spec:
  podSelector:
    matchLabels:
      role: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx   # Ingress Controller 所在的 namespace
      ports:
        - protocol: TCP
          port: 8080                                       # Pod 的 port,不是 Service 的 80
```

- Ingress Controller 的 namespace 名稱因環境而異(可能是 `ingress-nginx` 或 `kube-system`),用這個指令找:

```bash
kubectl get pods -A | grep ingress
```

- 如果是用 NodePort 或 LoadBalancer **直接**連到 Pod(沒經過 Ingress),來源可能是節點 IP 或使用者 IP,依環境而異,這時改用前面的 `ipBlock` 比較合適。

---

## 如何確認規則有生效

```bash
# 查看規則
kubectl get networkpolicy -n ns2
kubectl describe networkpolicy allow-from-ns1-443 -n ns2

# 從 ns1 的 Pod 測試連線(443 應該通,其他 port 應該不通)
kubectl exec -n ns1 <pod名稱> -- curl -sk -m 5 https://<service名稱>.ns2.svc.cluster.local
```

---

## 重點整理

| 想做的事 | 作法 |
|---|---|
| 全通 | 不建規則,或 `ingress: [- {}]` |
| 全擋進來 | `podSelector: {}` + `policyTypes: [Ingress]`,不寫 `ingress` |
| 全擋進出 | `policyTypes` 同時寫 `Ingress` 和 `Egress` |
| 白名單 | 先全擋,再用 `from` / `to` + `ports` 一條一條開 |
| 跨 namespace | 用 `namespaceSelector`,規則放在**被連線的那一側** |
| 擋出去 | 別忘了放行 DNS(53) |
| 放行特定 IP | 用 `ipBlock` 寫 `cidr` |
| 經過 Ingress 的網站 | 放行 **Ingress Controller 所在的 namespace**,port 寫 Pod 的 port |
| 標籤 | `role: web` 只是便利貼,Deployment、Service、NetworkPolicy 用同一個標籤找人 |
