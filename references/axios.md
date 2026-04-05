
# Axios npm供应链投毒事件安全分析报告
## 报告基础信息
| 字段 | 内容 |
|---|---|
| 报告名称 | Axios npm供应链投毒事件安全分析报告 |
| 分析时间 | 2026-04-05 |
| 分析人员 | 安全应急响应团队 |
| 风险等级 | 严重 |
| 标签 | #供应链安全 #投毒攻击 #npm生态 #RAT #UNC1069 |

---

## 一、事件概述
> 2026年3月31日，npm生态中最基础的HTTP客户端库Axios遭遇严重供应链投毒攻击。幕后黑手是朝鲜APT组织UNC1069，通过攻陷Axios核心维护者的npm账号，绕过GitHub的代码审核与CI/CD流程，向npm发布了两个恶意版本（1.14.1、0.30.4），同时覆盖1.x和0.x两大版本线。攻击者通过预埋恶意依赖`plain-crypto-js`，利用postinstall安装钩子，在用户安装包时自动部署跨平台远程访问木马（RAT），实现对开发者终端的完全控制，整个攻击窗口仅持续3小时，却波及了全球数百万开发者。

---

## 二、事件时间线
| 时间（UTC+8） | 事件节点 |
|---|---|
| 2026-03-30 13:57 | 攻击者使用一次性账号，发布干净的`plain-crypto-js@4.2.0`版本，构建可信发布历史，为后续投毒做准备 |
| 2026-03-30 23:59 | 攻击者发布恶意版本`plain-crypto-js@4.2.1`，注入带postinstall钩子的恶意投递器 |
| 2026-03-31 08:21 | 攻击者使用被盗的维护者账号，发布恶意版本`axios@1.14.1`，标记为latest最新版，瞄准现代版本用户 |
| 2026-03-31 09:00 | 发布针对旧版本用户的恶意版本`axios@0.30.4`，标记为legacy旧版，实现全版本线覆盖 |
| 2026-03-31 11:29 | npm官方紧急下架两个恶意版本，攻击窗口正式关闭，整个攻击仅持续3小时8分钟 |
| 2026-03-31 ~ 04-03 | Elastic、腾讯安全、谷歌威胁情报等厂商陆续发布深度分析报告，谷歌将攻击归因于UNC1069组织 |

---

## 三、攻击手法深度分析
### 3.1 初始入侵路径
本次攻击是典型的"账号劫持+依赖预埋"的精密供应链攻击：
1. 第一步：攻击者攻陷Axios核心维护者的npm账号，窃取了发布令牌，绕过了GitHub的代码审核、CI/CD流水线，直接向npm registry发布恶意包，GitHub源码仓库本身没有任何恶意代码，常规的源码审计完全无法发现。
2. 第二步：攻击者提前18小时，用一次性账号发布了干净的`plain-crypto-js@4.2.0`版本，在npm上建立可信的发布历史，避免新包触发安全警报。
3. 第三步：在投毒前30分钟，发布带恶意postinstall钩子的`plain-crypto-js@4.2.1`版本。
4. 第四步：用被盗的维护者账号，发布Axios的恶意版本，仅修改了package.json，新增了对`plain-crypto-js`的依赖，Axios本身的代码完全干净，极具迷惑性。

### 3.2 恶意代码行为分析
当用户执行`npm install axios`时，恶意依赖会自动触发postinstall脚本，执行完整的跨平台投毒流程：
1. **第一阶段：平台检测与载荷投递**
   恶意投递器会检测当前的操作系统，然后从攻击者的C2服务器`sfrclak.com:8000`下载对应平台的第二阶段RAT载荷：
   - **Windows平台**：下载PowerShell脚本，伪装成Windows Terminal的`wt.exe`，写入`%PROGRAMDATA%`目录，添加注册表启动项实现持久化
   - **macOS平台**：下载C++编译的二进制文件，伪装成苹果系统守护进程`com.apple.act.mond`，写入`/Library/Caches`目录
   - **Linux平台**：下载Python脚本，伪装成动态链接器`ld.py`，写入`/tmp`目录，用nohup后台运行

2. **第二阶段：跨平台RAT控制**
   三个平台的RAT共享完全相同的C2协议：
   - 每60秒向C2服务器发送心跳，使用Base64编码的JSON传输数据，伪装成IE8的User-Agent规避检测
   - 支持远程命令执行、文件系统遍历、内存二进制注入，攻击者可以完全控制受感染的主机
   - 支持绕过macOS的Gatekeeper校验，实现恶意代码的免杀执行

3. **第三阶段：反取证自清理**
   恶意代码执行完成后，会自动删除自身的痕迹：
   - 删除postinstall钩子，将package.json替换为干净的版本
   - 删除临时的恶意文件，事后检查node_modules完全无法发现任何痕迹
   - 整个过程无任何报错、无任何提示，用户完全无感知

### 3.3 攻击组织与工具
谷歌威胁情报团队将本次攻击归因于朝鲜APT组织UNC1069，该组织长期活跃于供应链攻击与加密货币窃取领域，本次使用的RAT是其标志性的WAVESHAPER.V2恶意软件，该组织此前就曾通过供应链攻击针对加密货币公司发起攻击。

---

## 四、影响范围评估
### 4.1 受影响版本清单
| 组件名称 | 受影响恶意版本 | 官方安全修复版本 |
|---|---|---|
| Axios npm包 | 1.14.1、0.30.4 | 1.x: <=1.14.0；0.x: <=0.30.3 |
| 恶意依赖 | plain-crypto-js@4.2.1 | 无，该包为攻击者伪造 |

### 4.2 用户影响范围
- 受影响用户：2026年3月31日08:21~11:29期间，安装了上述恶意版本的用户，包括直接安装，以及通过下游依赖间接安装的用户
- 影响规模：Axios是npm生态的基础组件，周下载量超1亿次，直接/间接依赖项目超17.4万个，哪怕只是CI/CD流水线的自动构建，都可能中招
- 潜在危害：攻击者可以完全控制受感染的主机，窃取SSH密钥、云凭证、API密钥，甚至通过横向移动渗透企业内网，实现长期控制。

---

## 五、用户自查与排查方案
### 5.1 依赖版本检查
首先检查本地的Axios版本，确认是否命中受影响版本：
```bash
# 检查Axios版本
npm list axios
# 检查是否存在恶意依赖plain-crypto-js
npm list plain-crypto-js
# 检查锁文件，确认是否有恶意版本的记录
grep -E 'axios@(1\.14\.1|0\.30\.4)|plain-crypto-js@4\.2\.1' package-lock.json yarn.lock pnpm-lock.yaml
```

### 5.2 恶意残留检查
检查系统中是否存在RAT的残留文件：
```bash
# macOS用户
find /Library/Caches -name "com.apple.act.mond"
# Windows用户
dir %PROGRAMDATA%\wt.exe
# Linux用户
find /tmp -name "ld.py"
```

### 5.3 网络行为排查
检查系统网络日志，确认是否存在向恶意C2域名的访问记录：
- 检查是否有访问`sfrclak.com`的请求
- 检查是否有使用IE8的User-Agent的异常POST请求

---

## 六、修复与缓解方案
### 6.1 紧急修复措施
1. **立即卸载恶意版本**：卸载受影响的Axios版本，安装安全版本，清理缓存：
   ```bash
   # 清理node_modules和锁文件
   rm -rf node_modules package-lock.json yarn.lock pnpm-lock.yaml
   # 安装安全版本
   npm install axios@1.14.0 --ignore-scripts
   # 清理npm缓存
   npm cache clean --force
   ```
2. **全量重置敏感凭证**：
   假设所有在风险窗口期内的环境的凭证都已泄露，立即重置：
   - 所有SSH私钥，同步更新服务器的authorized_keys
   - 所有云平台的AccessKey、K8s集群Token
   - 所有Git Token、npm令牌、数据库密码
   - 所有LLM API密钥、环境变量中的敏感凭证
3. **全盘安全扫描**：对终端、服务器进行全盘扫描，清理RAT残留，排查是否有持久化后门

### 6.2 长期缓解措施
1. 依赖版本锁定：在package.json中固定依赖版本，禁止使用`^`、`~`、`latest`等模糊版本号，避免自动升级到恶意版本
2. 禁用安装脚本：全局禁用npm的postinstall脚本，从源头阻止安装阶段的恶意代码执行：
   ```bash
   npm config set ignore-scripts true
   ```
3. CI/CD加固：
   - 使用lock文件，用`npm ci`代替`npm install`，保证构建一致性
   - 清理CI/CD的缓存，避免缓存命中恶意包
4. 供应链监控：部署供应链安全监控，实时监测npm包的异常发布、异常依赖变更

---

## 七、IOC 入侵检测指标
以下是本次攻击的入侵检测指标，可用于安全设备检测：
### 恶意组件版本
```
axios==1.14.1
axios==0.30.4
plain-crypto-js==4.2.1
```

### 恶意文件特征
```
macOS恶意文件：/Library/Caches/com.apple.act.mond
Windows恶意文件：%PROGRAMDATA%\wt.exe
Linux恶意文件：/tmp/ld.py
```

### 恶意C2服务器地址
```
域名：
sfrclak.com (IP: 142.11.206.73)
```

### 恶意网络特征
```
POST请求，User-Agent为mozilla/4.0 (compatible; msie 8.0; windows nt 5.1; trident/4.0)
Base64编码的JSON数据上传
```

---

## 八、总结与安全建议
### 事件总结
本次Axios投毒事件是npm生态有史以来最精密的供应链攻击之一，攻击者通过"预埋依赖+账号劫持+自清理反取证"的手法，实现了对全球数百万开发者的无感知投毒。事件暴露了当前开源生态的信任链风险：npm registry和GitHub是两个独立的系统，维护者的npm账号一旦沦陷，就可以绕过所有的代码审核流程，直接向开发者推送恶意代码，而开发者的自动更新习惯，也让攻击者可以快速扩散恶意包。

### 安全建议
#### 开发者侧
1. 固定依赖版本，禁止使用模糊版本号，避免自动升级
2. 禁用npm的postinstall脚本，从源头阻止安装阶段的恶意代码
3. 为所有npm、GitHub账号开启2FA，定期轮换发布令牌
4. 定期轮换敏感凭证，避免凭证长期有效

#### 企业侧
1. 部署私有npm源，对公共包进行审核与拦截
2. 加固CI/CD流水线，使用lock文件，清理构建缓存
3. 隔离开发环境，限制开发工具的系统权限
4. 定期对开发团队做供应链安全的培训，提升安全意识

---

## 九、参考来源
1. 谷歌威胁情报：《Axios npm供应链攻击归因于UNC1069》：http://news.qq.com/rain/a/20260403A01U5Y00
2. Elastic Security：《Axios 供应链攻击深度剖析》：https://www.51cto.com/article/839828.html
3. 腾讯安全：《Axios npm供应链攻击威胁分析》：https://m.sohu.com/a/1005615215_121124365/
4. StepSecurity：《三亿周下载量的基石塌陷:Axios 恶意投毒事件深度研判》：https://www.51cto.com/article/839562.html
5. 稀土掘金：《每周1亿次下载的axios被投毒了，但是源码里没有一行恶意代码!》：https://juejin.cn/post/7623256350250516518
6. npm官方：《Security Advisory: Malicious versions of axios》
