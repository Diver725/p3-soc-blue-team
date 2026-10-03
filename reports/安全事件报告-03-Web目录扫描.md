# 安全事件报告 03：Web 目录扫描

## 一、事件概述

| 项目 | 内容 |
|---|---|
| 事件编号 | P3-IR-003 |
| 事件类型 | Web 目录扫描 / Wordlist Scanning |
| 检测时间 | 2026-10-03 09:22:59 ～ 09:33:13 UTC |
| 攻击源 | p3-kali，192.168.30.30 |
| 受影响主机 | p3-dvwa，192.168.30.20 |
| 目标服务 | Apache HTTP 80/tcp |
| 扫描工具 | Gobuster 3.8.2 |
| 扫描规模 | 13 个测试路径 |
| 发现路径 | /config、/.git、/robots.txt、/login.php、/setup.php、/server-status |
| 自定义检测规则 | Wazuh Rule 100200，Level 10 |
| 当前状态 | 已阻断并回滚，事件关闭 |

## 二、事件时间线

| 时间（UTC） | 事件 |
|---|---|
| 09:22:59 | Gobuster 开始对 p3-dvwa 执行目录扫描 |
| 09:22:59 | Apache 记录 13 条 Web 请求 |
| 09:23:01 | Wazuh 开始产生 Rule 31101 Web 4xx 告警 |
| 09:23:01 | `/server-status` 触发 Rule 31516 Suspicious URL access |
| 09:33:12 | 添加自定义频繁扫描规则后再次执行 Gobuster |
| 09:33:13 | Wazuh Rule 100200 命中 |
| 处置阶段 | UFW 临时阻断 192.168.30.30 到 80/tcp |
| 验证阶段 | Kali 访问超时，输出 http-blocked |
| 恢复阶段 | 删除临时 UFW 规则，HTTP 恢复正常 |

## 三、攻击路径

```text
p3-kali 192.168.30.30
→ HTTP 80/tcp
→ p3-dvwa 192.168.30.20
→ Apache 逐个响应目录和文件请求
→ 根据 200、301、403、404 判断路径是否存在
```

扫描命令：

```bash
gobuster dir -u http://192.168.30.20 -w /tmp/p3-dirscan.txt -t 5 -q
```

该事件属于：

```text
Reconnaissance / Active Scanning / Wordlist Scanning
```

不是直接漏洞利用，而是攻击前的路径发现。

## 四、Gobuster 结果

```text
config        301
robots.txt    200
login.php     200
.git          301
server-status 403
setup.php     200
```

其他路径返回：

```text
404
```

状态码含义：

- `200`：资源存在并可访问
- `301`：目录或资源发生永久重定向
- `403`：路径存在但禁止访问
- `404`：路径不存在

重点观察：

- `/config` 和 `/.git` 返回 301，说明路径可能存在
- `/server-status` 返回 403，说明路径存在但受限
- `/login.php` 和 `/setup.php` 返回 200，属于公开 Web 入口

## 五、Apache 原始日志证据

代表日志：

```text
192.168.30.30 - - [03/Oct/2026:09:22:59 +0000] "GET /admin HTTP/1.1" 404 436 "-" "gobuster/3.8.2"
192.168.30.30 - - [03/Oct/2026:09:22:59 +0000] "GET /config HTTP/1.1" 301 524 "-" "gobuster/3.8.2"
192.168.30.30 - - [03/Oct/2026:09:22:59 +0000] "GET /server-status HTTP/1.1" 403 439 "-" "gobuster/3.8.2"
192.168.30.30 - - [03/Oct/2026:09:22:59 +0000] "GET /setup.php HTTP/1.1" 200 2565 "-" "gobuster/3.8.2"
```

关键字段：

```text
srcip：192.168.30.30
url：/admin、/config、/server-status 等
id：HTTP 状态码
User-Agent：gobuster/3.8.2
location：/var/log/apache2/access.log
```

## 六、Wazuh 检测

### 内置 Rule 31101

```text
Rule ID：31101
Level：5
Description：Web server 400 error code.
```

用于检测单个 404/4xx 请求。

### 内置 Rule 31516

```text
Rule ID：31516
Level：6
Description：Suspicious URL access.
URL：/server-status
```

### 自定义 Rule 100200

```text
Rule ID：100200
Level：10
Frequency：8
Timeframe：120
Description：Custom web directory scanning detection: multiple 4xx responses from same source.
MITRE：T1595.003
Tactic：Reconnaissance
Technique：Wordlist Scanning
```

触发逻辑：

```text
同一源 IP
120 秒内
Rule 31101 匹配 8 次
→ 触发 100200
```

`previous_output` 中包含之前的多个 404 请求，证明这是时间窗口内的关联告警。

## 七、IOC

| 类型 | 值 |
|---|---|
| 源 IP | 192.168.30.30 |
| 目标 IP | 192.168.30.20 |
| 目标端口 | 80/tcp |
| User-Agent | gobuster/3.8.2 |
| 扫描路径 | /admin、/backup、/config、/.git、/server-status 等 |
| 告警规则 | 31101、31516、100200 |
| MITRE | T1595.003 |
| 时间窗口 | 2026-10-03 09:22:59 ～ 09:33:13 UTC |

## 八、影响评估

- 攻击者发现了部分可访问、受限或重定向路径。
- `/.git`、`/config` 和 `/server-status` 属于较敏感的路径特征，在真实应用中可能增加后续攻击面。
- 当前目标为 DVWA 实验环境，没有真实业务数据。
- 未发现路径被进一步利用、文件被读取或服务器被入侵。

## 九、研判结论

```text
True Positive
```

依据：

- Apache 日志存在大量同源 HTTP 请求
- User-Agent 固定为 gobuster/3.8.2
- 请求路径来自目录字典
- 短时间出现多个 404/301/403/200
- Wazuh 产生了单条 4xx 告警并最终关联出 100200 扫描行为

## 十、应急处置

### 临时阻断

```bash
sudo ufw insert 1 deny from 192.168.30.30 to any port 80 proto tcp
```

### 验证

在 p3-kali：

```bash
curl -I --max-time 3 http://192.168.30.20 || echo http-blocked
```

结果：

```text
curl: (28) Connection timed out
http-blocked
```

### 回滚

```bash
sudo ufw delete deny from 192.168.30.30 to any port 80 proto tcp
```

恢复验证：

```text
HTTP/1.1 302 Found
Location: login.php
```

说明 Web 服务恢复。

## 十一、改进建议

1. 隐藏或限制 `/.git`、`/config`、`/server-status` 等敏感路径。
2. 对高频 404/403 请求设置阈值和自动封禁。
3. 结合 WAF 限制目录扫描和异常 User-Agent。
4. 保留 Rule 100200，并根据扫描字典规模调整 frequency。
5. 将扫描源 IP 和 User-Agent 纳入 IOC 库。
6. 在 Dashboard 中增加“Web 扫描趋势”和“Top 404 来源 IP”视图。
7. 将目录扫描和 SQL 注入等 Web 攻击进行关联分析。
