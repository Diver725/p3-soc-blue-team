# P3 · SOC 安全运营与护网蓝队应急响应演练

定位：SIEM 监控 + 告警研判 + IOC 提取 + 应急响应，按护网蓝队值守方式组织（就业导向，含金量最高）。

## 技术栈
Wazuh（SIEM，单节点）+ Linux 靶机 + Web 靶机（DVWA）+ Kali（攻击机，仅内网）；Kibana 仪表盘做态势感知视图

## 周期
约 4-6 周

## 目录
- architecture/   网络拓扑图
- docs/           部署文档、规则调优记录、操作步骤
- scripts/        日志解析、事件模拟（无害）脚本
- configs/        Wazuh 自定义检测规则、Agent 配置
- screenshots/    告警、研判、处置过程截图
- reports/        多份标准化安全事件报告、蓝队值守记录

## 环境规划（Host-only #4 · 192.168.30.0/24）
| 主机 | IP | 角色 |
|---|---|---|
| p3-wazuh | 192.168.30.10 | Wazuh Manager / Indexer / Dashboard |
| p3-dvwa | 192.168.30.20 | DVWA Web 靶机 + Wazuh Agent |
| p3-kali | 192.168.30.30 | 内网攻击机 |
| Windows 主机 | 192.168.30.1 | 管理访问入口 |

## 当前状态
- [ ] 第 1 周：SOC 环境搭建 + 全量日志采集
  - [x] 创建 Host-only #4：192.168.30.0/24
  - [x] 克隆 p3-wazuh 到 D:\VirtualBoxVMs
  - [x] p3-wazuh：8GB 内存、4 vCPU、Host-only #4、NAT
  - [x] p3-wazuh 虚拟磁盘扩到 64GB
  - [x] p3-wazuh 文件系统扩容到约 62GB
  - [x] p3-wazuh 主机名修改为 p3-wazuh
  - [x] p3-wazuh 静态 IP 修改为 192.168.30.10/24
  - [x] 部署 Wazuh 4.14 单节点（Manager/Indexer/Dashboard 均 active）
  - [x] Dashboard 登录页可通过 https://192.168.30.10 访问
  - [x] 创建 p3-wazuh-base 和 p3-wazuh-installed 快照
  - [x] 创建 p3-dvwa：192.168.30.20，Host-only #4 + NAT
  - [x] 部署 Apache、PHP、MariaDB 和 DVWA
  - [x] 安装 Wazuh Agent 4.14，Manager 端显示 Agent 001 Active
  - [x] 创建 p3-dvwa-base 快照
  - [x] 创建 p3-dvwa-dvwa-installed 快照
  - [x] 创建 p3-dvwa-agent-installed 快照
  - [x] 验证 Dashboard Endpoints 和日志采集
  - [ ] 冻结/暂停 Wazuh 包自动升级并整理部署记录
  - [x] 创建 p3-kali 攻击机（D 盘，2GB/2 vCPU，Host-only #4 + NAT）
  - [x] 配置 p3-kali 静态 IP 为 192.168.30.30/24，并创建 p3-kali-base 快照
- [ ] 第 2-3 周：检测规则调优 + 4 类事件模拟
  - [x] 场景 1：SSH 暴力破解（Hydra → Wazuh 5763 → UFW 阻断 → 恢复）
  - [ ] 场景 2：Web SQL 注入
  - [ ] 场景 3：Web 目录扫描
  - [ ] 场景 4：可疑文件/权限或提权异常
- [ ] 第 4-5 周：分析师工作流（告警→研判→IOC→隔离→复盘）+ 值守记录
- [ ] 第 6 周：态势感知仪表盘 + 收尾

详细设计见：`docs/P3环境设计与第1周计划.md`

## 技能点清单（项目完成后补，对齐岗位 JD 关键词）

## 简历描述（STAR，项目完成后补）

## 面试高频题（项目完成后补）









