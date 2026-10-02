# 场景 1 半扶复现记录：SSH 暴力破解

## 复现日期

2026-10-02

## 复现目标

在已经理解理论的基础上，重新完成：

```text
Hydra SSH 暴力破解
→ p3-dvwa 原始日志
→ Wazuh Rule 5763
→ Dashboard 告警
→ UFW 阻断
→ 验证
→ 回滚
```

## 复现过程

### 1. 环境确认

- p3-wazuh 的 Manager、Indexer、Dashboard 均为 active
- p3-dvwa 的 Wazuh Agent 为 active
- p3-kali 能访问 192.168.30.20:22
- p3-dvwa UFW 没有残留的 Kali 拒绝规则
- fail2ban 没有残留封禁

### 2. 攻击准备

- 暂时停止 p3-dvwa 的 fail2ban
- 在 p3-kali 准备错误密码字典
- 使用 Hydra 发起 10 次 SSH 密码尝试

### 3. 原始日志验证

```text
Failed password：14 条
源 IP：192.168.30.30
目标用户：lab
Accepted login：无
当前异常会话：无
```

攻击结果为：

```text
True Positive
攻击尝试：真实
登录成功：否
```

### 4. Wazuh 告警验证

```text
Rule ID：5763
Level：10
Frequency：8
Description：sshd: brute force trying to get access to the system
Agent：p3-dvwa
Source IP：192.168.30.30
Target user：lab
MITRE：T1110 / Brute Force
```

`previous_output` 中包含多条之前的失败日志，证明 Rule 5763 是基于时间窗口内的事件关联触发，而不是单条日志误报。

### 5. 处置和回滚

- 在 p3-dvwa 使用 UFW 在最前面插入针对 Kali 的 22/tcp DENY 规则
- 从 p3-kali 验证 TCP 22 连接被阻断
- 确认阻断生效后删除临时 UFW 规则
- 恢复 fail2ban 服务
- 确认没有残留封禁

## 复现结论

```text
攻击模拟：成功
日志采集：成功
Wazuh 检测：成功
告警研判：成功
处置和回滚：成功
```

场景 1 的端到端流程已经完成半扶复现。
