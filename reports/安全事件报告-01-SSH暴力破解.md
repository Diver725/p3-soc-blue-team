# 安全事件报告 01：SSH 暴力破解

## 一、事件概述

| 项目 | 内容 |
|---|---|
| 事件编号 | P3-IR-001 |
| 事件类型 | SSH 暴力破解 |
| 检测时间 | 2026-09-29 15:25:56 ～ 15:26:20 UTC |
| 攻击源 | p3-kali，192.168.30.30 |
| 受影响主机 | p3-dvwa，192.168.30.20 |
| 目标服务 | SSH 22/tcp |
| 目标用户 | lab |
| 攻击工具 | Hydra |
| 攻击结果 | 10 次失败尝试，0 次成功 |
| Wazuh Agent | 001，p3-dvwa |
| 当前状态 | 已阻断并恢复，事件关闭 |

## 二、事件时间线

| 时间（UTC） | 事件 |
|---|---|
| 15:25:56 | Kali 开始使用 Hydra 对 p3-dvwa 的 SSH 服务进行密码尝试 |
| 15:25:58 | p3-dvwa 首次记录认证失败 |
| 15:25:59 | 出现 `Failed password for lab` |
| 15:26:07 | 连接达到 SSH 最大认证尝试次数并被断开 |
| 15:26:09 | Wazuh 产生 Rule 2502 和 Rule 5763 告警 |
| 15:26:16 | Hydra 完成攻击，未发现有效密码 |
| 15:26:20 | 分析员开始查看原始日志和 Wazuh 告警 |
| 处置阶段 | 使用 UFW 临时封禁 192.168.30.30 到 p3-dvwa 的 22/tcp |
| 验证阶段 | Kali TCP 连接测试返回 `ssh-blocked` |
| 恢复阶段 | 删除临时 UFW 规则，恢复 fail2ban 服务 |

## 三、攻击路径

```text
p3-kali 192.168.30.30
→ SSH TCP 22
→ p3-dvwa 192.168.30.20
→ 使用错误密码字典尝试登录用户 lab
→ 多次认证失败
```

攻击命令摘要：

```text
hydra -l lab -P /tmp/p3-wrong-passwords.txt -t 2 ssh://192.168.30.20
```

使用的测试密码均为实验用错误密码，没有尝试公网目标。

## 四、原始日志证据

p3-dvwa 的 SSH 日志：

```text
Sep 29 15:25:59 p3-dvwa sshd[2531]: Failed password for lab from 192.168.30.30 port 41250 ssh2
Sep 29 15:25:59 p3-dvwa sshd[2532]: Failed password for lab from 192.168.30.30 port 41258 ssh2
Sep 29 15:26:07 p3-dvwa sshd[2532]: error: maximum authentication attempts exceeded for lab from 192.168.30.30 port 41258 ssh2 [preauth]
Sep 29 15:26:07 p3-dvwa sshd[2532]: Disconnecting authenticating user lab 192.168.30.30 port 41258: Too many authentication failures [preauth]
```

关键字段：

```text
源 IP：192.168.30.30
目标主机：p3-dvwa
目标用户：lab
目标端口：22
结果：认证失败
```

## 五、Wazuh 告警

### Rule 5760：单次 SSH 认证失败

```text
Rule ID：5760
Level：5
Description：sshd: authentication failed.
MITRE：T1110.001、T1021.004
```

含义：

```text
检测到一次 SSH 密码认证失败
```

### Rule 2502：重复密码失败

```text
Rule ID：2502
Level：10
Description：syslog: User missed the password more than one time
MITRE：T1110
```

含义：

```text
同一来源在短时间内出现多次密码错误
```

### Rule 5763：SSH 暴力破解

```text
Rule ID：5763
Level：10
Description：sshd: brute force trying to get access to the system. Authentication failed.
Frequency：8
MITRE：T1110
Tactic：Credential Access
Technique：Brute Force
```

含义：

```text
Wazuh 识别到来自同一来源的大量 SSH 失败尝试，
判定为 SSH 暴力破解事件。
```

## 六、IOC

| 类型 | 值 |
|---|---|
| 源 IP | 192.168.30.30 |
| 目标 IP | 192.168.30.20 |
| 目标端口 | 22/tcp |
| 目标用户 | lab |
| 时间窗口 | 2026-09-29 15:25:56 ～ 15:26:20 UTC |
| 告警规则 | 2502、5760、5763 |
| MITRE | T1110、T1110.001、T1021.004 |
| 攻击工具特征 | Hydra 用户代理/连接模式未单独提取，主要通过高频失败认证识别 |

## 七、影响评估

```text
登录成功次数：0
目标账号是否被接管：否
目标服务是否中断：未中断
数据是否泄露：未发现
```

结论：

```text
事件为受控实验中的模拟攻击，
未获得有效凭据，未造成实际业务影响。
```

## 八、研判结论

```text
True Positive
```

判断依据：

- 原始 SSH 日志明确记录多次失败登录
- 源 IP 固定为 192.168.30.30
- 目标用户固定为 lab
- 短时间内出现 10 次密码尝试
- Wazuh Rule 5763 明确命中暴力破解规则
- Hydra 输出显示 0 个有效密码

不存在误报证据。

## 九、应急处置

### 1. 临时阻断

在 p3-dvwa 上执行：

```bash
sudo ufw insert 1 deny from 192.168.30.30 to any port 22 proto tcp
```

验证规则：

```bash
sudo ufw status numbered
```

结果：

```text
[ 1] 22/tcp DENY IN 192.168.30.30
```

### 2. 阻断验证

在 p3-kali 上执行：

```bash
timeout 3 bash -c '</dev/tcp/192.168.30.20/22' && echo ssh-open || echo ssh-blocked
```

结果：

```text
ssh-blocked
```

说明处置动作生效。

### 3. 恢复

删除临时规则：

```bash
sudo ufw delete deny from 192.168.30.30 to any port 22 proto tcp
```

恢复 fail2ban：

```bash
sudo systemctl start fail2ban
systemctl is-active fail2ban
```

## 十、恢复验证

- p3-dvwa 的 Wazuh Agent 仍保持 Active。
- 临时 UFW 封禁已删除。
- fail2ban 已恢复运行。
- 未发现攻击者成功登录。
- 该事件无需重启业务系统。

## 十一、改进建议

1. 保留 fail2ban 并确认 SSH jail 生效。
2. 优先推进 SSH 密钥认证，减少密码爆破面。
3. 限制 SSH 来源到管理网段。
4. 对 Rule 5763 设置更明显的 Dashboard 告警视图。
5. 将攻击源 IP、目标用户和时间窗口纳入 IOC 库。
6. 后续测试 Wazuh Active Response 或自动化封禁，减少人工处置时间。
7. 为 SSH 暴力破解建立固定演练脚本和验证流程。

## 十二、证据文件

- p3-dvwa SSH journald 原始日志
- p3-wazuh `alerts.json` 告警记录
- p3-dvwa UFW 规则截图
- p3-kali Hydra 输出
- Kali 阻断验证输出
