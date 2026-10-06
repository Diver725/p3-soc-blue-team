# P3 Web 基础补课与复现手册

> 目的：补齐 P3 场景 2～4 所依赖的 Web、HTTP、数据库、PHP 和蓝队检测基础。
>
> 这份手册不替代 `P3理论补课与复现手册.md`。M1～M7 讲 SOC 和 Wazuh，本手册讲 Web 安全和实验中的具体技术。

## 一、适用背景

P3 前四个场景中：

```text
场景 1：SSH 暴力破解
场景 2：SQL 注入
场景 3：Web 目录扫描
场景 4：可疑 Web 文件 / FIM
```

场景 2～4 都依赖一些没有系统学过的 Web 知识：

- HTTP 请求和响应
- GET 和 POST
- HTML 表单参数
- Cookie 和 Session
- SQL 与数据库
- Web 路径和状态码
- PHP 和 Webshell
- 文件完整性监控

目标是能够做到：

```text
看到实验现象，知道它属于哪一层
看到日志，知道关键字段在说什么
看到 Wazuh 规则，知道规则为什么触发
看到事件，知道如何处置和验证
```

## 二、模块路线

| 模块 | 主题 | 对应实验 | 必须能回答 |
|---|---|---|---|
| W1 | Web 系统与 HTTP | 所有场景 | 浏览器、Apache、PHP、数据库如何连接 |
| W2 | HTML 表单与 GET/POST | 场景 2、3、4 | 输入框参数如何到达服务器 |
| W3 | Cookie、Session 与登录 | 场景 2 | 登录后服务器如何记住用户 |
| W4 | URL、路径和状态码 | 场景 3 | 200、301、302、403、404 各代表什么 |
| W5 | 数据库与 SQL 基础 | 场景 2 | SELECT、WHERE、UNION 的作用 |
| W6 | SQL 注入原理与复现 | 场景 2 | 输入如何改变 SQL 查询 |
| W7 | 目录扫描与 Gobuster | 场景 3 | 如何从日志识别批量路径探测 |
| W8 | PHP、Webshell 与 FIM | 场景 4 | 文件新增、修改、删除如何被发现 |
| W9 | 四个场景综合串联 | P3 全场景 | 攻击链、日志、规则、处置如何对应 |

## 三、固定复盘流程

每个模块都按同一方式学习：

```text
1. 先理解概念
2. 对照现有实验
3. 观察原始日志
4. 找到 Wazuh 规则和告警字段
5. 用自己的话复述
6. 半扶复现一次
7. 独立完成关键步骤
```

## 四、模块内容

### W1：Web 系统与 HTTP

核心结构：

```text
浏览器 / curl
→ HTTP 请求
→ Apache
→ PHP
→ MariaDB
→ HTTP 响应
```

要理解：

- 请求方法：GET、POST、HEAD
- 请求头：Host、User-Agent、Cookie
- 响应状态码：200、301、302、403、404
- 请求日志和告警日志的区别

实验观察：

```bash
curl -v http://192.168.30.20
curl -I http://192.168.30.20
```

在 p3-dvwa 观察：

```bash
sudo tail -n 10 /var/log/apache2/access.log
sudo tail -n 10 /var/log/apache2/error.log
```

验收问题：

1. 浏览器访问网址时，实际发生了什么？
2. Apache 和 PHP 各自负责什么？
3. Wazuh 从哪里看到 Web 访问？

### W2：HTML 表单与 GET/POST

核心链路：

```text
HTML input 的 name
→ HTTP 参数
→ PHP 的 $_GET 或 $_POST
→ 应用逻辑
```

GET 示例：

```text
/vulnerabilities/sqli/?id=1
```

POST 示例：

```text
POST /login.php
username=admin&password=password
```

要理解：

- GET 参数在 URL 中
- POST 参数在请求体中
- `name` 决定参数名
- `method` 决定提交方式

实验观察：

```bash
curl -s http://192.168.30.20/login.php | sed -n '/<form/,/<\/form>/p'
```

验收问题：

1. `id=1` 中的 `id` 是什么？
2. 用户名输入框为什么会变成 `username=admin`？
3. 为什么 SQL 注入在 access.log 中能看到 `id` 参数？

### W3：Cookie、Session 与登录

登录流程：

```text
GET /login.php
→ 返回登录页面和 CSRF token
→ POST 用户名和密码
→ 登录成功
→ Set-Cookie: PHPSESSID=...
→ 后续请求携带 Cookie
```

要理解：

- Cookie 保存在客户端
- Session 数据通常保存在服务器
- PHPSESSID 是会话标识
- user_token 用于 CSRF 防护

实验观察：

```bash
rm -f /tmp/dvwa-cookies.txt
curl -s -c /tmp/dvwa-cookies.txt -o /dev/null http://192.168.30.20/login.php
cat /tmp/dvwa-cookies.txt
```

验收问题：

1. 登录成功后服务器怎样记住用户？
2. `-b` 和 `-c` 在 curl 中分别做什么？
3. 为什么登录需要先获取 CSRF token？

### W4：URL、路径和状态码

URL 路径映射：

```text
http://192.168.30.20/login.php
→ /var/www/html/login.php
```

状态码：

```text
200：存在且允许访问
301：永久重定向，通常是目录
302：临时重定向，常见于登录跳转
403：存在但禁止访问
404：不存在
```

实验观察：

```bash
curl -I http://192.168.30.20/login.php
curl -I http://192.168.30.20/server-status
curl -I http://192.168.30.20/not-exist
```

验收问题：

1. `/config` 返回 301 说明什么？
2. `/server-status` 返回 403 说明什么？
3. 为什么目录扫描要看状态码组合？

### W5：数据库与 SQL 基础

数据库结构：

```text
数据库
└── 表
    ├── 列
    └── 行
```

基础 SQL：

```sql
SELECT first_name, last_name
FROM users
WHERE user_id = '1';
```

要理解：

- `SELECT`：查询
- `FROM`：数据来自哪个表
- `WHERE`：过滤条件
- `UNION`：合并两个查询结果
- `--`：注释

实验观察：

```bash
sudo mysql -e "USE dvwa; SHOW TABLES; DESCRIBE users; SELECT user_id, user FROM users LIMIT 5;"
```

验收问题：

1. `WHERE user_id = '1'` 的作用是什么？
2. 数据库表和 HTML 页面有什么关系？
3. 为什么 SQL 是 Web 应用和数据库之间的语言？

### W6：SQL 注入原理与复现

正常输入：

```text
id=1
```

异常输入：

```text
id=1' OR '1'='1
```

可能形成的 SQL：

```sql
SELECT first_name, last_name
FROM users
WHERE user_id = '1' OR '1'='1';
```

核心概念：

```text
单引号提前结束字符串
OR 改变条件逻辑
'1'='1' 恒为真
UNION 拼接额外查询结果
-- 注释后续 SQL
```

要观察：

- Apache access.log 中的 URL 参数
- Wazuh Rule 31106
- 自定义 Rule 100100
- 响应结果是否比正常查询多

验收问题：

1. 为什么输入可以改变 SQL 结构？
2. 为什么 `OR '1'='1'` 会返回大量数据？
3. 参数化查询为什么能防 SQL 注入？

### W7：目录扫描与 Gobuster

Gobuster 做的事：

```text
使用字典
→ 批量发送 HTTP GET
→ 观察状态码
→ 判断路径是否存在
```

关键日志特征：

```text
同一源 IP
短时间大量请求
固定 User-Agent
多个 404
夹杂 301、403、200
```

Wazuh 检测：

```text
31101：单条 4xx
100200：同一源 IP 120 秒内命中 8 次
```

验收问题：

1. 目录扫描属于攻击链哪个阶段？
2. 为什么单个 404 不是告警，多个 404 可能是扫描？
3. UFW 阻断后如何验证扫描源被限制？

### W8：PHP、Webshell 与 FIM

PHP 是服务器端脚本：

```text
浏览器请求 .php
→ Web 服务器交给 PHP
→ PHP 执行
→ 返回 HTML 或其他数据
```

Webshell：

```text
被攻击者故意放置的恶意脚本
用于执行命令、读取文件或维持访问
```

PHP 文件不一定是后门，普通 PHP 也可以是登录、上传和数据库代码。

FIM 检测范围：

```text
文件新增：Rule 554
文件修改：Rule 550
文件删除：Rule 553
自定义 Web 文件新增：Rule 100300
```

处置原则：

```text
先隔离文件
→ 保留哈希和权限
→ 验证 Web 路径不可访问
→ 再分析来源
```

验收问题：

1. PHP 文件和 Webshell 有什么区别？
2. FIM 能直接告诉你是谁上传的吗？
3. 为什么隔离比直接删除更适合应急响应？

### W9：四个场景综合串联

把每条链路写成同一种格式：

```text
攻击动作
→ 原始日志
→ Wazuh 解码
→ 规则命中
→ 告警字段
→ IOC
→ 处置
→ 验证
→ 报告
```

场景对应：

```text
场景 2：SQL 注入
输入参数 → Apache → Wazuh 31106/100100 → UFW

场景 3：目录扫描
Gobuster → Apache → Wazuh 31101/100200 → UFW

场景 4：可疑文件
PHP 写入 → FIM → 554/100300/550/553 → 隔离与验证
```

验收问题：

1. 这三个场景分别处于攻击链什么阶段？
2. 哪些证据来自原始日志，哪些来自 Wazuh 告警？
3. 哪些操作是阻断，哪些操作是证据保全？

## 五、建议学习顺序

```text
第一轮：W1、W2、W3、W4
第二轮：W5、W6
第三轮：W7
第四轮：W8
最后：W9
```

学习时不要只读文档。每个模块至少完成：

```text
概念复述
日志观察
告警字段解释
一次半扶复现
```

## 六、不需要死记的内容

暂时不需要：

- 背下所有 curl 参数
- 从零编写所有攻击命令
- 背 Wazuh 所有内置规则 ID
- 背完整 SQL 语法

需要掌握：

```text
知道每类命令解决什么问题
知道日志字段代表什么
知道规则为什么触发
知道如何处置和验证
```

## 七、下一步衔接

完成 W1～W9 后，再进入 `P3理论补课与复现手册.md` 的 M1～M7。

最终目标是能把这两个层次连接起来：

```text
Web 技术基础
→ 攻击方法和日志
→ Wazuh 检测规则
→ 蓝队研判和应急处置
```
