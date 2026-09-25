# 第 1 章：从零扩展到百万用户

[English](./Readme.md)

## 引言
把系统扩展到能支撑百万用户，是一条需要反复打磨和优化的迭代之路。本章说明如何从单机起步，再一步步把架构扩到百万用户规模。

---

## 第 1 节：单机部署
一开始，所有组件（Web 应用、数据库、缓存）都跑在同一台服务器上。

<div style="margin-left:3rem">
   <img src="./images/single-server.png" width="400" />
</div>

### 请求流程
1. 用户通过域名（例如 `api.mysite.com`）访问应用，域名经 DNS 解析为 IP 地址。
2. Web 服务器的 IP 返回给浏览器或移动 App。
3. HTTP 请求发到 Web 服务器，服务器返回 HTML 或 JSON。

浏览器和 App 最终必须连 **IP**（例如 `15.125.23.214`），网络层不认域名。域名只是给人记的别名，所以每次访问都要先问：这个名字现在对应哪台机器？单机阶段 Web、数据库、缓存都在同一台机器上，DNS 记的就是这台机器的公网 IP。

整条链路：

```
用户输入 api.mysite.com
        ↓
   DNS：名字 → IP（A 记录直接给 IP；CNAME 则先换成另一个名字再查 IP）
        ↓
客户端拿到 IP
        ↓
TCP 连上这台机器（HTTPS 再做 TLS 握手：SNI 选证书、校验证书）
        ↓
HTTP 请求（Host 仍是域名）
        ↓
同一台机器上的 Web 处理
        ↓
返回 HTML（网页）或 JSON（接口）
```

三层分工可以记成：

| 是什么 | 解决什么 |
|--------|----------|
| **IP** | 送到哪台机器 |
| **端口** | 这台机器上的哪类服务（443 是 HTTPS / Web，3306 是 MySQL，不是「哪一个网站」） |
| **Host（以及 TLS 的 SNI）** | 这个 Web 服务里的哪一个站点 |

对用户来说，地址栏输入域名后页面会出来，不必自己填 IP 或 Host。DNS、TCP、证书校验、`Host` 都是浏览器自动完成的。

#### DNS：A 记录与 CNAME

DNS 负责把名字翻译成 IP，这一步**还不访问你的业务服务器**，只是在问门牌号。常见记录：

| 类型 | 含义 | 例子 |
|------|------|------|
| **A** | 名字直接对应 IPv4 | `api.mysite.com` → `15.125.23.214` |
| **AAAA** | 名字直接对应 IPv6 | `api.mysite.com` → `2001:db8::1` |
| **CNAME** | 当前名字是另一个名字的别名，不直接给 IP | `api.mysite.com` → `lb.mysite.com`，再查 `lb` 的 A 记录得到 IP |

CNAME（Canonical Name）不是另一种连接方式。解析过程只是多问一次：你问 `api` 住哪，答案是「跟 `lb` 住一块」，再问 `lb` 才拿到门牌。

**CNAME 不是访问网站的必要条件。** 全程只用 A 记录（域名直接写 IP）完全能打开页面。没有 CNAME 时可以这样：

```
www.mysite.com     A    10.10.10.10
api.mysite.com     A    10.10.10.10
admin.mysite.com   A    10.10.10.10
```

站点能用，只是每个名字都要单独维护 IP。有 CNAME 时，多个子域名都指向 `lb.mysite.com`，机器换 IP 只改 `lb` 这一条 A 记录，不容易漏。

根域名（`mysite.com` 本身）通常不能配 CNAME，多用 A，或云厂商的 ALIAS/ANAME（对外像 A，背后仍跟另一个名字走）。

##### 为什么 CDN、对象存储、GitHub Pages、云负载均衡常用 CNAME

这些服务的共同点是：**真正提供内容的机器不在你机房里，IP 还经常变、且是一堆 IP。** 他们只给你一个「请把你的域名指到这个名字」的入口，这个指法就是 CNAME。

若自己写死 A 记录（`www.mysite.com A 1.2.3.4`），会有这些问题：

1. IP 不是一台、也不是长期不变（CDN 全球很多边缘节点；云负载均衡后面一排机器会扩缩容）。
2. 同一域名在不同地区应解析到不同 IP 时，一条写死的 A 做不到。
3. 你不该、也没权限维护别人机房的 IP 列表；对方改了，你的 A 就过期。

所以厂商给你的是一个**他们控制的主机名**，例如 Cloudflare 的 `xxx.cdn.cloudflare.net`、AWS ALB 的 `xxx.elb.amazonaws.com`、对象存储的 `bucket.cos.ap-guangzhou.myqcloud.com`、GitHub Pages 的 `yourname.github.io`。你把自家域名 CNAME 到这个名字，「这个名字现在是哪些 IP」由**他们的 DNS** 回答。

CNAME 只改「去哪台机器」，不改浏览器里的域名。证书、SNI、Host 仍是 `www.mysite.com` / `static.tcamp.qq.com`，所以 CDN/存储/LB 上还要配你的自定义域名和证书。

四种服务并不都是「解析时返回离用户最近的 IP」：

| 服务 | 解析时会不会「给最近的 IP」 |
|------|------------------------------|
| **CDN** | 会。这是它的核心能力：厂商 DNS 做 GeoDNS，或用 Anycast，把用户送到近处的边缘节点。 |
| **云负载均衡** | 一般是你选的那个区域（如广州）里的 LB IP，**不是全球就近**；同区域多 IP 是为了高可用。 |
| **对象存储** | 多半是桶所在地域的入口 IP。要就近，通常要在前面再套一层 CDN。 |
| **GitHub Pages** | 通常就是 GitHub 的入口，**不是**按用户位置全球选节点。 |

CDN 的解析过程（用户在北京打开 `www.mysite.com`，且已 CNAME 到 `xxx.cdn.example.net`）：

```
浏览器问 DNS：www.mysite.com 的 IP？
    → 你的 DNS：这是 CNAME，去问 xxx.cdn.example.net
    → CDN 自己的 DNS：按「谁在问」（大体是用户/运营商 DNS 所在地）返回近处节点 IP
浏览器连该 IP:443（SNI / Host 仍是 www.mysite.com）
    → 节点缓存命中就直接返回；未命中再回源
```

东京用户查同一个 CDN 名字，拿到的往往是另一组 IP。就近发生在「解析厂商那个域名」的时候，由 CDN 做，不是你自己的 A 记录做的。

##### 例子：腾讯云 COS 绑定 `static.tcamp.qq.com`

把项目静态域名 CNAME 到腾讯云，**不等于**默认就会返回离用户最近的 IP。要看 CNAME 指向的是 COS 还是 CDN。

**只绑 COS（没开 CDN）：** 桶在广州时，默认域名类似 `xxx-123.cos.ap-guangzhou.myqcloud.com`：

```
static.tcamp.qq.com  CNAME  xxx-123.cos.ap-guangzhou.myqcloud.com
```

用户访问时跟到的是「广州地域 COS 接入」的 IP，再去广州读这个桶。北京、上海的用户通常也还是连广州这套入口。CNAME 的作用是自定义域名交给腾讯、IP 由他们维护，**不是做就近调度。**

**COS 前面再开 CDN：** 控制台给该自定义域名开启 CDN 加速后，CNAME 目标会变成 CDN 的域名（不再是 `*.cos.ap-guangzhou.myqcloud.com`）。这时才是：CDN 的 DNS 返回离用户近的边缘节点 IP；节点有缓存就直接回，没有再回源到广州的 COS。就近发生在 CDN 解析，COS 只当源站。

控制台里看 CNAME 目标就能分清：带 `cos.地域.myqcloud.com` 是存储入口；带 `cdn` 一类才是就近加速。

#### 已经连上 IP，为什么 HTTP 里还要 Host

TCP 连的是 **IP + 端口**（如 `15.125.23.214:443`）。这只说明数据包到了这台机器上的某个进程。一台 Nginx 上经常同时挂 `www`、`api`、`admin`，它们**共用同一个 IP、同一个 443**。端口只能区分「这是 Web 而不是数据库」，不能再区分「Web 里的哪一个站」。

HTTP 请求必须告诉服务器要哪套站点配置：

```http
GET /users/1 HTTP/1.1
Host: api.mysite.com
```

`Host` 写成 IP、或写错域名，可能落到默认站、404，或 HTTPS 证书对不上。HTTP/1.1 规定必须带 `Host`，浏览器每次都会从你输入的 URL 里自动带上。

**单站点时**（例如只在 `10.10.10.10` 上部署一个前端，只绑定 `tcamp.qq.com`）：访问 `https://tcamp.qq.com` 确实就是这个前端。此时不靠 `Host` 也能找到唯一服务，`Host` 看起来像多余的。它仍然会被带上，只是服务器几乎不用它来「选站」。**多站点共用同一 IP 和 443 时**，才必须靠 `Host` 区分。

不要用「每个网站一个端口」来替代 Host：浏览器默认只认 443，用户还得记 `:444`、`:445`，证书和防火墙也会很难做。现实是无数个 HTTPS 站点共用 443，用域名区分。

#### HTTPS 仍然需要 Host，另外还有 SNI

HTTPS 是加密后的 HTTP，请求内容没变，里面照样有 `Host`。握手时域名会出现两次，职责不同：

| 阶段 | 带域名的字段 | 干什么 |
|------|----------------|--------|
| TLS 握手 | **SNI** | 按域名选出证书，把加密建起来（进门亮证件） |
| HTTP 请求 | **Host** | 选出网站、返回对应页面（告诉前台要哪个房间） |

证书证明「你是不是真的这个域名」（防冒充）。`Host` 决定「加密建好后，这台机器上要哪一个站」。同一张证书、同一个 IP 上仍可能有 `www` 和 `map`，不能单靠证书替代 Host。

#### 同一 IP 上多个前端：Nginx 怎么做

对外 HTTPS 网站前面通常放一层 Nginx（或 Caddy、网关、Ingress），角色是 443 上的大门。前端进程往往听 `3000`、`8080`，**不直接占用 443**。同一 IP 上两个进程都 `listen 443` 会冲突，所以 443 只交给 Nginx。

重点三步：

1. 多个域名的 DNS 都指向同一台机器（如 `10.10.10.10`）。
2. Nginx 只在 443 上听一份，用 `server_name` 对请求里的 `Host`（HTTPS 时先用 SNI 选证书）。
3. 命中的 `server` 用 `root` 指静态目录，或用 `proxy_pass` 转到本机不同端口。

```nginx
server {
    listen 443 ssl;
    server_name tcamp.qq.com;
    ssl_certificate     /etc/nginx/certs/tcamp.qq.com.pem;
    ssl_certificate_key /etc/nginx/certs/tcamp.qq.com.key;
    root /var/www/tcamp;
    location / { try_files $uri $uri/ /index.html; }
}

server {
    listen 443 ssl;
    server_name admin.tcamp.qq.com;
    ssl_certificate     /etc/nginx/certs/admin.tcamp.qq.com.pem;
    ssl_certificate_key /etc/nginx/certs/admin.tcamp.qq.com.key;
    root /var/www/admin;
    location / { try_files $uri $uri/ /index.html; }
}
```

独立进程时把 `root` 换成 `proxy_pass http://127.0.0.1:3000;` / `3001;`，并带上 `proxy_set_header Host $host;`。只有一个域名时也可以按路径拆（`/admin/`），那时靠 URL 路径而不是 Host，前端的 `publicPath` / `base` 要配成对应前缀。HTTP 同理，端口换成 80，一样靠 Host 分流。

#### 证书配在 Nginx 上

浏览器连的是 Nginx 的 443，必须先拿出证书完成握手，再转到前端。证书是大门上的身份证，配在 Nginx 的 `ssl_certificate`（公钥证书）和 `ssl_certificate_key`（私钥）上。后面的前端一般不用管证书。

证书上写着它能为哪些名字作证。访问的域名必须出现在证书的 **SAN** 里：

- **每个域名一张：** 每个 `server` 填自己的 `.pem` / `.key`。SNI 是 `tcamp.qq.com` 就拿出第一张，是 `admin.tcamp.qq.com` 就拿出第二张。
- **一张通配符 `*.tcamp.qq.com`：** 能覆盖该域下一级子域名。两个 `server` 可填**同一对**文件。站点仍靠 `server_name` 分开，只是证书不用各办一张。`*.tcamp.qq.com` 通常**不包含**根域名 `tcamp.qq.com` 本身，除非证书上另外写了根域名。

#### HTTPS 校验细节（TLS 握手）

校验发生在 TLS 握手，页面内容还没传。任一步失败，浏览器红锁/拦截，不会进到前端。

```
1. TCP 连上 IP:443
2. 浏览器 → ClientHello（SNI: tcamp.qq.com）
3. Nginx  → 按 SNI 选出证书，把证书链发回来
4. 浏览器   校验证书
5. Nginx   用私钥证明「这张证是我的」
6. 双方算出会话密钥，之后内容加密
7. 才发 HTTP（Host: tcamp.qq.com）
```

Nginx 发来的通常是一条链，例如：站点证 ← 中间 CA 签发 ← 根 CA 签发。根证预装在系统/浏览器「信任的根证书」里。

浏览器大致检查：

1. **域名是否匹配：** SAN 里有没有当前访问的名字；通配符 `*.tcamp.qq.com` 能匹配 `admin.tcamp.qq.com`，一般不能匹配 `tcamp.qq.com` 本身。
2. **是否在有效期内：** 当前时间在 `Not Before` 与 `Not After` 之间。
3. **链是否连到信任根：** 用上一张证的公钥验下一张的数字签名，直到系统里已有的根。链断、中间证没带全、自签名，都会失败。
4. **是否被吊销：** OCSP / CRL（有的浏览器软失败，不阻塞）。
5. **用途：** 须能用于服务器身份验证。

以上只说明「这张证是 CA 签给这个域名的，且仍有效」。证书可以复制，**私钥不能给别人**。服务器必须用私钥做一次密码学运算；浏览器用证书里的公钥来验。验过才证明对面持有私钥。`.pem` 发给浏览器看，`.key` 绝不能发给浏览器。

失败时的典型现象：域名和 SAN 不一致 → 证书与网站名称不匹配；过期 → 证书过期；链到不了信任根 → 不受信任的颁发机构；私钥对不上 → 握手失败。

**证书 / SNI：证明正在和真的该域名加密通信。Host：加密建好之后，才决定进哪个前端。**

### 流量来源
1. **Web 应用：** 服务端语言（如 Python、Java）处理业务逻辑，客户端语言（如 JavaScript、HTML）负责展示。
2. **移动应用：** 通过 HTTP 和 JSON 与 Web 服务器通信，交换轻量数据。

---

## 第 2 节：数据库拆分
用户规模增长后，把数据库迁到独立服务器，让 Web 层和数据层可以各自扩展。

<div style="margin-left:3rem">
   <img src="./images/database.png" width="400" />
</div>

### 数据库选型

1. **关系型数据库（SQL）：** 结构化数据存放在表中。例如：MySQL、PostgreSQL。
2. **非关系型数据库（NoSQL）：** 适合非结构化数据或低延迟场景。常见类型包括：
   - 键值存储（Key-Value）
   - 图数据库（Graph）
   - 列式存储（Column Store）
   - 文档存储（Document Store）

- 在以下情况下，非关系型数据库可能更合适：
   - 应用需要极低延迟。
   - 数据非结构化，或没有关联关系。
   - 只需要对数据进行序列化/反序列化（JSON、XML、YAML 等）。
   - 需要存储海量数据。

这里说的是 **两种数据库（以及它们习惯存放的数据形态）**。核心差别：**有没有固定的表结构、行与行之间靠不靠「关系」来查。** 很多系统两个都用（订单走 MySQL，会话走 Redis，日志走文档或列式），并不是二选一。

#### 关系型（SQL）：先定表，数据必须对上格子

数据放在 **表** 里，每张表有固定列（schema）。一行就是一条记录，列就是字段。多张表用外键连起来，查询用 SQL 做 JOIN。

例如用户和下单：

```
users                          orders
id | name | email              id | user_id | amount
1  | 张三 | a@x.com            1  | 1       | 99
```

`orders.user_id` 指向 `users.id`，这就是「关系」。要查张三的所有订单，靠 JOIN，不必把用户信息在每张订单里再抄一遍。

特点：

- **结构化：** 类型、必填、唯一，数据库会帮你卡住。
- **关系清晰：** 用户、订单、商品分表，用 JOIN 拼回来。
- **事务强：** 转账、下单这种「几张表要么一起成功要么一起失败」很合适。
- 换列、海量写入、极低延迟时往往更吃力（要改表、垂直扩展或分片）。

适合：账号、订单、库存、支付——规则稳定、要一致。

#### 非关系型（NoSQL）：不先定死表，按访问方式存

没有统一的「多表 + JOIN」模型。上面四类都是 NoSQL，但彼此也不一样：

| 类型 | 怎么存 | 直观理解 |
|------|--------|----------|
| **键值（Redis、DynamoDB 一类）** | `key → value` | 像一本超大字典 |
| **文档（MongoDB）** | 一条就是一份 JSON | 用户和其订单可以塞在同一个文档里 |
| **列式（Cassandra、HBase）** | 按列族存，一行可以有很多动态列 | 适合超宽、超大表 |
| **图（Neo4j）** | 点和边 | 好友、推荐、依赖关系 |

文档库可以没有单独的 `orders` 表，直接：

```json
{
  "name": "张三",
  "email": "a@x.com",
  "orders": [{ "amount": 99 }, { "amount": 20 }]
}
```

没有强制 schema，字段可多可少；一般 **不靠 JOIN**，而是按 key / 文档把一次请求要的数据放一块。

书里说 NoSQL 更合适的情况，对应这些差异：要极低延迟（很多是内存或按主键点查，少 JOIN）；数据非结构化、没有关系（日志、JSON、各用户字段都不一样）；存进去再原样拿出来即可；体量极大时按 key 分片、水平扩展更顺。代价通常是：跨记录事务弱、临时换一种查询方式很难、数据可能重复（反规范化）。

#### 对照

| | 关系型 SQL | 非关系型 NoSQL |
|--|------------|----------------|
| 结构 | 先定表和列 | 灵活，或只有 key |
| 表之间 | 用外键、JOIN | 少 JOIN，能嵌就嵌、能按 key 查就按 key 查 |
| 一致性 | 事务强 | 往往更偏性能和扩展，事务弱一些 |
| 扩展 | 单机很强，横向分片麻烦 | 更容易加机器 |
| 典型 | 订单、账户 | 缓存、会话、日志、海量宽表、社交关系图 |

---

## 第 3 节：垂直扩展 vs 水平扩展
### 垂直扩展（Vertical Scaling）
- 给现有服务器增加资源（CPU、内存）。
- 受硬件上限约束，且缺少冗余。

### 水平扩展（Horizontal Scaling）
- 向服务器池中增加机器，更适合大规模系统。
- 用负载均衡器在多台服务器之间分发请求。
---

## 第 4 节：负载均衡器

<div style="margin-left:3rem">
   <img src="./images/load-balancer.png" width="400" />
</div>

**负载均衡器**把流量分发到多台服务器。好处包括：
1. **冗余：** 某台服务器宕机时，流量会被转到其他机器。
   - 例如服务器 1 下线后，全部流量会打到服务器 2。
2. **可扩展：** 流量突增时可以方便地加机器。
   - 网站流量快速增长时，可以继续加服务器来消化增量。

图里是：**外面一个负载均衡器，后面多台 Web Server。** 落到腾讯云 + K8s，要分清 **云上的 CLB** 和 **集群里的 Ingress**。很多人说的「在 K8s 上部署负载均衡」，其实常常把它们混在一起。

#### 腾讯云 TKE：CLB 在哪，Server 在哪

业务进程在 **K8s 的 Pod** 里，Pod 被调度到 **Worker 节点**（一般是 CVM，或 TKE 超级节点/原生节点）。这些节点上的 Pod 才是书里的「Server 1、Server 2」，在你的 VPC 里。

**CLB 不跑在你的业务 Pod 里。** 它是腾讯云托管的入口（一个 VIP），建在云厂商的网络里。你在控制台或用 `Service: type LoadBalancer` 申请的是这层。

```
用户
  → 腾讯云 CLB（负载均衡，通常不在你的 K8s 里）
  → 集群节点 或 Pod
  → 你的前端/后端容器
```

域名（如 `tcamp.qq.com`）解析到 **CLB 的 IP**，不是某个 Pod 的 IP。Pod IP 会变，CLB 后面的后端列表由 TKE 跟着 Endpoint 更新。

#### 「部署在 K8s 上」通常是哪种

**1. Service `type: LoadBalancer`（最常见）**

K8s 只声明「我要一个对外 LB」。TKE 会在集群**外面**建一台 **CLB**，并把后端绑到节点或 Pod：

| 谁 | 在哪 |
|----|------|
| CLB（书里的 Load Balancer） | 腾讯云托管，VPC 里的 VIP |
| 后端 Server | Worker 节点上的 Pod |

流量常见两条路：

- **CLB → 节点 IP:NodePort → kube-proxy → Pod**（传统）
- **CLB 直接绑 Pod 网卡**（VPC-CNI 等，少一跳）

无论哪条，**应用都在集群节点上的 Pod 里。**

**2. Ingress（Nginx / Traefik 等）**

集群**里面**还会跑一组 Ingress Pod（也在 Worker 上），按域名/`Host` 分流，很像第 1 节的 Nginx。公网入口往往仍是：

```
用户 → CLB → Ingress Pod → 各个业务 Service → 业务 Pod
```

- CLB：四层/七层入口，在云上
- Ingress：在 K8s 里，按 `tcamp.qq.com` 转到不同服务
- 业务：还是各命名空间里的 Pod

**3. 自己在集群里跑 Nginx，没有 CLB**

可以，但要自己管节点 IP、弹性公网 IP、证书。生产上 TKE 更常用 CLB 当大门。

#### 和书上那张图对齐

| 第 4 节图 | 腾讯云 TKE |
|-----------|------------|
| Load Balancer | 云 CLB（偶尔再加集群内 Ingress） |
| Web Server 1 / 2 | 不同节点上的 Pod 副本 |
| 加机器 | `replicas` 变多，或加 Worker 节点 |
| 一台 Server 挂了 | Pod 被调度走 / 节点被摘，CLB 不再打到它 |

**一句话：负载均衡（CLB）在腾讯云侧；真正干活的 Server 是 K8s Worker 上的 Pod。** 「在 K8s 上部署 LB」多半是创建了 LoadBalancer/Ingress，让云自动开 CLB，并不是把 CLB 进程装进集群当普通应用跑。

---

## 第 5 节：数据库复制

<div style="margin-left:3rem">
   <img src="./images/database-replication.png" width="400" />
</div>

### 主从模型
- **主库（Master）：** 处理写操作。
   - 所有修改数据的命令（insert、delete、update）都必须发到主库。
- **从库（Slave）：** 处理读操作，提升性能和可靠性。
   - 大多数应用读多写少，因此从库数量通常多于主库。

### 收益
1. 读操作可并行，性能更好。
2. 通过冗余获得高可用和数据可靠性。


### 故障处理
- 只有一台从库且它下线时，读请求会临时打到主库。
- 有多台从库时，读请求会转到其他健康从库，并用新服务器替换故障机。
- 主库下线时，会把某台从库提升为新主库。
- 生产环境中被选中的从库可能尚未完全追上主库，需要跑数据恢复脚本补齐数据（多主、环形复制等方法也有帮助）。

---

## 第 6 节：缓存
**缓存**把频繁访问的数据放在内存里，减轻数据库压力。缓存层是临时存储，比数据库快得多。

<div style="margin-left:3rem">
   <img src="./images/cache.png" width="500" />
</div>

### 缓存要点
1. **适用场景：** 数据读多写少时，适合用缓存。
2. **过期策略：** 缓存到期后会被删除。没有过期策略时，数据会一直占着内存。
3. **一致性：** 指数据存储与缓存保持同步。对两者的写操作通常不在同一个事务里，因此可能出现不一致。
4. **故障缓解：** 单台缓存服务器是单点故障（SPOF），建议在多个数据中心部署多台缓存。
5. **淘汰策略：** 缓存满了就要淘汰条目腾出内存。LRU 是最常用的淘汰策略。

---

## 第 7 节：内容分发网络（CDN）
**CDN** 把静态内容（图片、CSS、JavaScript）缓存在地理上分散的节点上，从而加快加载。

<div style="margin-left:3rem">
   <img src="./images/cdn.png" width="400" />
</div>

### 工作流程
1. 用户向最近的 CDN 节点请求内容。
2. 若未命中，则从源站拉取并缓存。

自定义域名接 CDN 时，通常把 `www.mysite.com` / `static.tcamp.qq.com` **CNAME** 到厂商给的 CDN 主机名，由对方的 DNS（GeoDNS 或 Anycast）返回离用户近的边缘节点 IP。对象存储（如腾讯云 COS）本身一般只解析到**桶所在地域**的入口；要就近访问，需要再开 CDN，让 CNAME 指向 CDN 而不是 `*.cos.地域.myqcloud.com`。详见第 1 节「DNS：A 记录与 CNAME」。


### CDN 要点
1. **成本：** CDN 由第三方运营，进出流量通常按量计费。
2. **缓存过期：** 过期时间既不能太长也不能太短。
3. **CDN 降级：** 出现临时故障时，客户端应能发现问题并改向源站拉资源。
4. **文件失效：** 文件更新后应让缓存失效，指向新文件。

---

## 第 8 节：无状态 Web 层
把会话数据放到共享存储后，Web 服务器就可以无状态。这样可以：
1. 更容易水平扩展。
2. 按流量自动扩缩容。

<div style="margin-left:3rem">
   <img src="./images/stateless.png" width="400" />
</div>

---

## 第 9 节：多数据中心
跨多个数据中心部署可以提高可用性、降低延迟。常见做法包括：

<div style="margin-left:3rem">
   <img src="./images/data-center.png" width="400" />
</div>

1. **GeoDNS 路由：** 把用户导向最近的数据中心。
2. **数据复制：** 在各中心之间同步数据，减少不一致。

### 关键考虑
- **流量调度：** 需要有效工具把流量打到正确的数据中心。
- **数据同步：** 常见策略是在多个数据中心之间复制数据。
- **测试与发布：** 自动化发布工具对保证各中心服务一致很重要。

---

## 第 10 节：消息队列
**消息队列**是持久化组件（可驻留内存），用于异步通信。它既是缓冲区，也负责分发异步请求。

<div style="margin-left:3rem">
   <img src="./images//message-queue.png" width="500" />
</div>

- 输入侧服务称为生产者/发布者，创建消息并发布到队列。
- 其他服务称为消费者/订阅者，连接到队列并执行消息所定义的操作。

---

## 第 11 节：日志、指标与自动化

<div style="margin-left:3rem">
   <img src="./images/logging.png" width="400" />
</div>

### 为什么重要
1. **日志：** 跟踪错误和系统健康状况。
2. **指标：** 观察性能和用户行为。
3. **自动化：** 简化测试、发布和扩缩容。

---

## 第 12 节：数据库扩展
### 垂直扩展
- 增加硬件资源，但有物理和成本上限。
- 若干缺点：
   - 单点故障风险更高。
   - 整体成本高。

### 水平扩展（分片 / Sharding）

<div style="margin-left:3rem">
   <img src="./images/horizontal-scaling.png" width="400" />
</div>

- 用键（例如 `user_id`）把数据拆到多个分片上。
   - 分片把大库拆成更小、更好管理的部分，称为 shard。
   - 各分片 schema 相同，但实际数据互不相同。
- 分片键非常关键。选择时要能把数据尽量均匀打散。

#### 挑战
1. **重新分片：** 在以下情况需要 reshard：
   - 单分片因增长过快装不下更多数据。
   - 数据分布不均，部分分片会更快耗尽。
   - 可用一致性哈希缓解这些问题。

2. **明星问题（Celebrity problem）：** 对某一分片的访问过多会导致过载。
   - 解决办法之一是给每个高热用户单独分配分片。

3. **Join 与反规范化：** 库被拆到多台机器后，跨分片 join 很困难。
   - 常见做法是反规范化，让查询能在单表内完成。

---

## 结语
### 要点
1. 保持 Web 层无状态。
2. 每一层都做冗余。
3. 用缓存和 CDN 优化性能。
4. 用分片扩展数据层。
5. 解耦组件以获得灵活性。

本章为构建能支撑百万用户的可扩展系统打下基础。
