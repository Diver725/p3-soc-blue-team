# 安全事件报告 02：Web SQL 注入

## 一、事件概述

| 项目 | 内容 |
|---|---|
| 事件编号 | P3-IR-002 |
| 事件类型 | Web SQL 注入 |
| 检测时间 | 2026-10-03 07:31:58 ～ 08:47:11 UTC |
| 攻击源 | p3-kali，192.168.30.30 |
| 受影响主机 | p3-dvwa，192.168.30.20 |
| 目标应用 | DVWA SQL Injection |
| 目标端口 | 80/tcp |
| 攻击工具 | curl 构造 HTTP 请求 |
| 攻击结果 | 首次 OR 注入返回 HTTP 200 并返回多条用户记录 |
| 自定义检测规则 | Wazuh Rule 100100，Level 10 |
| 当前状态 | 已阻断并回滚，事件关闭 |

## 二、事件时间线

| 时间（UTC） | 事件 |
|---|---|
| 07:31:58 | Kali 发送 `OR '1'='1` SQL 注入请求，HTTP 200 |
| 07:31:58 | DVWA 返回 admin、Gordon、Hack、Pablo、Bob 等多条记录 |
| 08:06:07 | Kali 发送大写 `UNION SELECT` 请求，HTTP 200 |
| 08:06:07 | Wazuh 命中内置 Rule 31106 |
| 08:11:23 | Kali 发送小写 `union select` 请求，Wazuh Rule 31106 再次命中 |
| 08:42:29 | 第二条 31106 告警进入 alerts.json |
| 08:47:11 | 添加自定义规则后再次发送 OR 注入，命中 Rule 100100，HTTP 302 |
| 处置阶段 | UFW 临时阻断 192.168.30.30 到 80/tcp |
| 验证阶段 | Kali curl 返回连接超时，输出 http-blocked |
| 恢复阶段 | 删除临时 UFW 规则 |

## 三、攻击路径

```text
p3-kali 192.168.30.30
→ HTTP 80/tcp
→ p3-dvwa 192.168.30.20
→ DVWA SQL Injection 页面
→ 修改 id 参数
→ SQL 查询逻辑被篡改
→ 返回多条用户记录
```

主要请求：

```text
GET /vulnerabilities/sqli/?id=1%27+OR+%271%27%3d%271&Submit=Submit
GET /vulnerabilities/sqli/?id=1%27+union+select+1%2c2--+-&Submit=Submit
```

## 四、原始日志证据

Apache access.log：

```text
192.168.30.30 - - [03/Oct/2026:07:31:58 +0000] "GET /vulnerabilities/sqli/?id=1%27+OR+%271%27%3d%271&Submit=Submit HTTP/1.1" 200 5294 "-" "curl/8.20.0"
192.168.30.30 - - [03/Oct/2026:08:06:07 +0000] "GET /vulnerabilities/sqli/?id=1%27+UNION+SELECT+1%2c2--+-&Submit=Submit HTTP/1.1" 200 5018 "-" "curl/8.20.0"
192.168.30.30 - - [03/Oct/2026:08:11:23 +0000] "GET /vulnerabilities/sqli/?id=1%27+union+select+1%2c2--+-&Submit=Submit HTTP/1.1" 200 5018 "-" "curl/8.20.0"
```

Decoded URL：

```text
/vulnerabilities/sqli/?id=1%27+OR+%271%27%3d%271&Submit=Submit
```

## 五、Wazuh 检测与规则调优

### 内置规则命中

```text
Rule 31106
Level 6
A web attack returned code 200 (success).
MITRE T1190
```

说明 Wazuh 能够识别请求属于 Web 攻击并且返回成功，但内置规则没有精确识别 `OR '1'='1` 布尔逻辑注入。

### 自定义规则

文件：

```text
configs/local_rules.xml
```

规则：

```text
Rule ID：100100
Level：10
Description：Custom SQL injection detection: OR tautology attempt.
MITRE：T1190
```

规则逻辑：

```text
基于 web access log 解析结果
匹配 %27+or+%27、%27+OR+%27、%27%20or%20%27、%27%20OR%20%27
```

复测结果：

```text
Rule 100100 命中
Dashboard 可见告警
alerts.json 可见完整字段
```

## 六、IOC

| 类型 | 值 |
|---|---|
| 源 IP | 192.168.30.30 |
| 目标 IP | 192.168.30.20 |
| 目标端口 | 80/tcp |
| 目标路径 | /vulnerabilities/sqli/ |
| 参数 | id |
| 注入特征 | `OR '1'='1`、`UNION SELECT` |
| User-Agent | curl/8.20.0 |
| 告警规则 | 31106、100100 |
| MITRE | T1190 |
| 时间窗口 | 2026-10-03 07:31:58 ～ 08:47:11 UTC |

## 七、影响评估

- 首次 OR SQL 注入返回 HTTP 200，并返回多条 DVWA 用户记录。
- 后续 UNION SELECT 请求也返回 HTTP 200。
- 最后一次自定义规则验证请求返回 HTTP 302，属于攻击尝试，不能单独证明注入成功。
- 目标为故意存在漏洞的 DVWA 靶机，没有真实业务数据。
- 未发现操作系统层面的入侵或持久化。

## 八、研判结论

```text
True Positive
```

依据：

- 原始 Apache 日志记录了恶意参数
- DVWA 返回了多条不应由普通 ID 查询返回的数据
- Wazuh 内置规则和自定义规则均产生了关联告警
- 攻击请求、源 IP、URL、User-Agent 和时间窗口一致

## 九、应急处置

### 临时阻断

```bash
sudo ufw insert 1 deny from 192.168.30.30 to any port 80 proto tcp
```

规则顺序：

```text
[ 1] 80/tcp DENY IN 192.168.30.30
[ 2] 22/tcp ALLOW IN 192.168.10.0/24
[ 3] 22/tcp ALLOW IN 192.168.30.0/24
[ 4] 80/tcp ALLOW IN 192.168.30.0/24
```

### 验证

在 p3-kali 上：

```bash
curl -I --max-time 3 http://192.168.30.20 || echo http-blocked
```

结果：

```text
curl: (28) Connection timed out
http-blocked
```

### 回滚

删除临时 UFW 规则，恢复原有 Web 服务访问。

## 十、恢复验证

- Apache 服务恢复正常访问。
- Wazuh Agent 持续在线。
- Rule 100100 可被真实请求触发。
- 临时 UFW 规则已删除。
- 没有发现真实数据泄露或系统持久化。

## 十一、改进建议

1. DVWA 仅保留在 host-only 网络，不暴露公网。
2. 在生产应用中使用参数化查询和预编译语句。
3. 对输入参数进行类型校验和规范化。
4. 结合 WAF 检测 SQL 注入特征。
5. 保留 Wazuh 自定义 SQL 注入规则，并增加针对 UNION、注释符和编码变体的检测。
6. 关注 HTTP 200 与 302 的差异：200 更可能代表攻击成功，302 更可能是未认证或重定向。
7. 对同一来源的多次 SQL 注入告警增加频率规则和自动处置。
