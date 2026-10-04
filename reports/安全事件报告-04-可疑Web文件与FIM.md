# 安全事件报告 04：可疑 Web 文件与文件完整性异常

## 一、事件概述

| 项目 | 内容 |
|---|---|
| 事件编号 | P3-IR-004 |
| 事件类型 | 可疑 Web 文件创建 / 文件完整性异常 |
| 检测时间 | 2026-10-04 09:47:46 ～ 10:13:05 UTC |
| 模拟来源 | p3-dvwa 本地用户 `lab` 使用 `sudo -u www-data` 模拟 Web 服务写文件 |
| 受影响主机 | p3-dvwa，192.168.30.20 |
| 目标目录 | `/var/www/html/hackable/uploads` |
| 目标服务 | Apache / DVWA |
| 检测模块 | Wazuh FIM / Syscheck realtime |
| 检测规则 | Rule 554、100300、550、553 |
| 当前状态 | 测试文件已隔离，事件关闭 |

说明：本场景没有使用真实恶意代码，测试文件只输出无害字符串，目的是验证 Wazuh 对 Web 可写目录中文件新增、修改和删除的检测与响应流程。

## 二、事件时间线

| 时间（UTC） | 事件 |
|---|---|
| 09:45:12 | p3-dvwa 加载 FIM realtime 配置，开始监控 `/var/www/html/hackable/uploads` |
| 09:47:46 | 创建无害测试文件 `p3-fim-test.php` |
| 09:47:46 | Rule 554 检测到文件新增 |
| 10:02:46 | 创建第二个测试文件 `p3-fim-test-100300.php` |
| 10:02:46 | 自定义 Rule 100300 检测到 uploads 目录中的 PHP 文件新增 |
| 10:06:12 | 两个测试文件被移出 Web 目录并隔离，Rule 553 检测到文件删除 |
| 10:08:52 | Kali 访问原测试路径返回 HTTP 404 |
| 10:11:14 | 创建第三个测试文件 `p3-fim-modify-test.php`，Rule 100300 再次命中 |
| 10:11:34 | 修改第三个测试文件，Rule 550 检测到大小和哈希变化 |
| 10:13:05 | 第三个测试文件被隔离，Rule 553 再次检测到文件删除 |

## 三、验证目标

本场景验证以下能力：

```text
FIM / Syscheck 实时监控
Web 可写目录中的文件新增检测
自定义高等级 Web 文件规则
文件大小、权限、属主、哈希和时间采集
文件内容变化检测与 diff
文件删除检测
可疑文件隔离和哈希保全
Web 路径不可访问验证
```

## 四、FIM 配置

在 p3-dvwa 的 `/var/ossec/etc/ossec.conf` 中增加：

```xml
<directories realtime="yes" report_changes="yes" check_all="yes">/var/www/html/hackable/uploads</directories>
```

参数含义：

- `realtime="yes"`：文件变化后实时检测
- `report_changes="yes"`：记录文件内容变化和 diff
- `check_all="yes"`：检查大小、权限、属主、时间、inode 和哈希

Syscheck 启动日志：

```text
Monitoring path: '/var/www/html/hackable/uploads', with options 'size | permissions | owner | group | mtime | inode | hash_md5 | hash_sha1 | hash_sha256 | report_changes | realtime'.
Directory set for real time monitoring: '/var/www/html/hackable/uploads'.
```

## 五、Wazuh 告警

### 1. 文件新增：Rule 554

```text
Rule ID：554
Level：5
Description：File added to the system.
Path：/var/www/html/hackable/uploads/p3-fim-test.php
Event：added
Mode：realtime
Size：125
Permission：rw-rw----
Owner：www-data:www-data
SHA256：3089842268be1cac228e42e23c3de6e5e2d286826e51ef1c93dd92ff3895779c
```

### 2. 自定义 Web 文件新增规则：Rule 100300

```xml
<group name="syscheck,web_upload,">
  <rule id="100300" level="10">
    <if_sid>554</if_sid>
    <field name="file">/var/www/html/hackable/uploads/</field>
    <description>Custom suspicious web file added to DVWA upload directory.</description>
    <mitre>
      <id>T1505.003</id>
    </mitre>
    <group>attack,web_shell,</group>
  </rule>
</group>
```

命中结果：

```text
Rule ID：100300
Level：10
Description：Custom suspicious web file added to DVWA upload directory.
MITRE：T1505.003 / Web Shell
Path：/var/www/html/hackable/uploads/p3-fim-test-100300.php
Event：added
SHA256：e44c3cd662221b0b07148c7d8c39294a9ae94ea3202c386c6d851e68e852a490
```

### 3. 文件内容修改：Rule 550

```text
Rule ID：550
Level：7
Description：Integrity checksum changed.
Path：/var/www/html/hackable/uploads/p3-fim-modify-test.php
Event：modified
Size：55 → 82
Changed attributes：size,mtime,md5,sha1,sha256
旧 SHA256：55dbfefa3e34a140b08a01c9c6387e94f5658e39dbc5b213d3065d9ed684f0b0
新 SHA256：37c24f49d1fc40b38f87c555f5070a27592faca9176679f53047ffe30db66f10
```

告警中的 diff：

```diff
4a5
> // P3 modification test v2
```

### 4. 文件删除：Rule 553

```text
Rule ID：553
Level：7
Description：File deleted.
Event：deleted
Mode：realtime
```

被删除的文件路径：

```text
/var/www/html/hackable/uploads/p3-fim-test.php
/var/www/html/hackable/uploads/p3-fim-test-100300.php
/var/www/html/hackable/uploads/p3-fim-modify-test.php
```

## 六、IOC 与证据

| 项目 | 值 |
|---|---|
| 目标主机 | p3-dvwa，192.168.30.20 |
| 目标目录 | /var/www/html/hackable/uploads |
| 文件一 | p3-fim-test.php |
| 文件一 SHA256 | 3089842268be1cac228e42e23c3de6e5e2d286826e51ef1c93dd92ff3895779c |
| 文件二 | p3-fim-test-100300.php |
| 文件二 SHA256 | e44c3cd662221b0b07148c7d8c39294a9ae94ea3202c386c6d851e68e852a490 |
| 文件三 | p3-fim-modify-test.php |
| 文件三修改后 SHA256 | 37c24f49d1fc40b38f87c555f5070a27592faca9176679f53047ffe30db66f10 |
| 属主/属组 | www-data:www-data |
| 原始权限 | rw-rw---- |
| 隔离目录 | /root/p3-quarantine |
| 隔离后权限 | 600 |

## 七、影响评估

- FIM 能在 realtime 模式下检测文件新增、修改和删除。
- 自定义 Rule 100300 能将 Web uploads 目录中的文件新增提升到 Level 10。
- Rule 550 能发现文件大小和哈希变化，并记录具体 diff。
- Rule 553 能确认可疑文件已经被移出 Web 目录。
- 当前文件没有真实恶意逻辑，没有执行系统命令，也没有造成数据泄露。
- 真实环境中，合法代码发布也可能触发同类告警，需要结合变更管理、sudo、SSH 和 Web 日志进行研判。

## 八、应急处置

### 隔离文件

```bash
sudo mkdir -p /root/p3-quarantine
sudo chmod 700 /root/p3-quarantine
sudo mv /var/www/html/hackable/uploads/p3-fim-test.php /root/p3-quarantine/
sudo mv /var/www/html/hackable/uploads/p3-fim-test-100300.php /root/p3-quarantine/
sudo mv /var/www/html/hackable/uploads/p3-fim-modify-test.php /root/p3-quarantine/
```

### 收紧隔离文件权限

```bash
sudo chmod 600 /root/p3-quarantine/p3-fim-test.php
sudo chmod 600 /root/p3-quarantine/p3-fim-test-100300.php
sudo chmod 600 /root/p3-quarantine/p3-fim-modify-test.php
```

### 隔离后哈希验证

```text
3089842268be1cac228e42e23c3de6e5e2d286826e51ef1c93dd92ff3895779c  p3-fim-test.php
e44c3cd662221b0b07148c7d8c39294a9ae94ea3202c386c6d851e68e852a490  p3-fim-test-100300.php
37c24f49d1fc40b38f87c555f5070a27592faca9176679f53047ffe30db66f10  p3-fim-modify-test.php
```

### Web 路径验证

```bash
curl -I --max-time 3 http://192.168.30.20/hackable/uploads/p3-fim-test-100300.php
```

结果：

```text
HTTP/1.1 404 Not Found
```

uploads 目录当前只剩 DVWA 自带文件：

```text
dvwa_email.png
```

## 九、研判结论

```text
True Positive（受控模拟事件）
```

依据：

- 文件确实出现在被监控的 Web 上传目录。
- FIM 获取了文件路径、大小、权限、属主、inode 和 SHA256。
- 自定义 Rule 100300 准确匹配 uploads 目录中的 PHP 文件新增。
- 文件内容修改后，Rule 550 检测到哈希变化并记录 diff。
- 文件移出目录后，Rule 553 检测到删除。
- 隔离后哈希保持不变，证明处置过程没有破坏证据。

## 十、改进建议

1. 在真实环境启用 `whodata` / auditd，补充文件变化的进程和用户归因。
2. 禁止 Web 上传目录直接执行 PHP，降低 Webshell 落地后的执行风险。
3. 对上传文件实施扩展名、MIME 类型和内容校验。
4. 将上传目录移出 Web 根目录，使用受控下载接口。
5. 为 Web 目录中的 PHP 文件新增、修改和权限变化设计分级规则。
6. 将 FIM 告警与 sudo、SSH、Apache 访问日志关联分析。
7. 在生产环境部署前评估合法发布流程，避免高等级规则产生误报。
8. 将自定义规则和 Agent 配置纳入 Git 版本管理。

## 十一、复现注意事项

- 实验必须在 Host-only 内网进行，不能扫描或攻击公网。
- 测试文件必须保持无害，不能包含真实 Webshell、反弹 Shell 或下载执行逻辑。
- 文件移动后哈希应保持不变，否则需要重新检查证据链。
- `dvwa_email.png` 是 DVWA 自带示例文件，不属于本次事件。
- 执行命令前确认主机提示符，避免把 p3-dvwa 命令误执行到 p3-wazuh。
