# Antigravity 国内环境完整配置指南

> Google Antigravity IDE + Antigravity CLI（`agy`）从安装、登录、订阅到命令行使用的全流程记录。
> 基于 2026 年 11 月的实际配置过程整理，**所有账号、支付、网络信息均已脱敏**。

📄 完整 PDF：[`Antigravity-CN-Setup-Guide.pdf`](./Antigravity-CN-Setup-Guide.pdf)

---

## 这份文档解决什么问题

官方文档和大多数教程都默认你在海外网络环境下操作。但在国内实际配置时，会连续撞上下面这些坑，而它们几乎都没有被写下来：

- Antigravity 的后端是 Go 程序，**不读 Windows 系统代理**，界面能开但模型列表一直加载不出来
- 某些地区的机房 IP 会被 Google 判定为"不支持的地区"，即使能连上也不给用
- Google One 的**方案页按出口 IP 显示货币**，但**实际扣款按付款资料所在国家**计价
- 预付卡/虚拟卡余额只够一个月，不取消续订会连环扣款失败
- 系统盘空间紧张时，用户数据和扩展目录怎么搬走
- 如何让 `agy` 和 Claude Code 共用同一套 skills

---

## 一、先看结论：装哪个、用什么

| 产品 | 图标 | 作用 |
|---|---|---|
| Antigravity App | 白色底 | Agent 编排工作台，**不能编辑代码** |
| **Antigravity IDE**（推荐） | **黑色网格底** | 基于 VS Code 的完整编辑器 + Agent 面板，能写代码、能改文件 |

> 只想写代码 → 装**黑色图标**的 Antigravity IDE。

另外：**个人版 Gemini CLI 已于 2026 年被 Google 停止服务**。账号登录成功后也会立刻报错：

```
Failed to sign in. Message: This client is no longer supported for
Gemini Code Assist for individuals. To continue using Gemini, please
migrate to the Antigravity suite of products: https://antigravity.google
```

官方指定的替代品是 **Antigravity CLI（命令名 `agy`）**，用法与 Claude Code 高度一致。

---

## 二、前置条件

| 条件 | 说明 |
|---|---|
| 网络 | 需要能稳定访问 Google 的网络环境（建议日本、美国等地区节点）。服务端会校验访问来源地区，无法绕过。 |
| Google 账号 | 账号"关联国家/地区"必须在支持列表内（美国、日本、韩国、新加坡、台湾等；**中国大陆、香港、澳门不在列表内**）。可访问 `policies.google.com/country-association-form` 查看。 |
| 支付方式 | 支持美元结算的国际信用卡或虚拟卡。免费额度可先用，但额度较低。 |
| 系统 | Windows 10 64 位及以上，预留 5 GB 以上空间。 |

> ⚠️ Antigravity 的地区判定是 **出口 IP 地区 + 账号关联地区** 双门槛。仅靠节点不能救一个地区不符的账号；反之账号地区正确但出口 IP 被判高风险（如机房 IP），也会被拦。

---

## 三、安装 Antigravity IDE

### 3.1 下载

`https://antigravity.google/download` → **Antigravity IDE (Standalone)** → Windows x64，约 190 MB。

### 3.2 装到非系统盘

Inno Setup 打包，支持静默安装并指定目录：

```bat
"Antigravity IDE.exe" /VERYSILENT /SUPPRESSMSGBOXES /NORESTART ^
  /DIR="E:\Antigravity" /MERGETASKS="!runcode"
```

`/MERGETASKS="!runcode"` 表示装完不自动启动。

### 3.3 把用户数据搬出 C 盘

程序默认把用户数据写在 `%APPDATA%\Antigravity IDE`，扩展写在 `%USERPROFILE%\.antigravity-ide`。

该版本**禁用了便携模式**（产品配置含 `target` 字段，`data` 文件夹不生效），所以用**目录联接**：

```powershell
# 先把原目录内容移走，再建联接
New-Item -ItemType Directory -Force -Path "E:\Antigravity\data\user-data" | Out-Null
Move-Item "C:\Users\<你>\AppData\Roaming\Antigravity IDE" "E:\Antigravity\data\user-data"
New-Item -ItemType Junction -Path "C:\Users\<你>\AppData\Roaming\Antigravity IDE" `
         -Target "E:\Antigravity\data\user-data"

New-Item -ItemType Junction -Path "C:\Users\<你>\.antigravity-ide" `
         -Target "E:\Antigravity\data\home\.antigravity-ide"
```

> 一定要**先把原目录内容移动过去再建联接**，否则数据丢失。

### 3.4 中文化

需要**两个**语言包叠加：

| 语言包 | 作用 |
|---|---|
| `MS-CEINTL.vscode-language-pack-zh-hans`（取与内核匹配的版本，如 1.106.x） | VS Code 内核界面 |
| `editorss.antigravity-language-pack-zh-hans` | Antigravity 自身扩展字符串 |

```bat
bin\antigravity-ide.cmd --install-extension MS-CEINTL.vscode-language-pack-zh-hans@1.106.0
bin\antigravity-ide.cmd --install-extension editorss.antigravity-language-pack-zh-hans
```

> **关键坑**：该版本**不认** `argv.json` 里的 `"locale": "zh-cn"`，必须用命令行参数 `--locale=zh-cn` 启动才生效。

### 3.5 启动脚本（同时解决代理与语言）

Antigravity 后端是 Go 程序，**不读取 Windows 系统代理**，必须显式设置环境变量：

```bat
@echo off
set "HTTP_PROXY=http://127.0.0.1:端口"
set "HTTPS_PROXY=http://127.0.0.1:端口"
set "NO_PROXY=localhost,127.0.0.1"
start "" "E:\Antigravity\Antigravity IDE.exe" --locale=zh-cn
```

把开始菜单快捷方式指向这个 `.cmd`（图标仍可用原 exe 的），点开就是"中文 + 带代理"。

---

## 四、登录 Google 账号

### 4.1 正确姿势：把登录链接复制到已登录的浏览器

1. 点 **Get Started** → **Continue with Google**
2. 出现 **Copy sign-in link**，点击复制
3. 粘贴到**已登录 Google 账号的浏览器**打开
4. 选择账号 → 同意授权 → 浏览器跳回 `http://localhost:端口/auth/callback`
5. 回到 IDE，等待显示 **Welcome back, 你的名字**

### 4.2 常见报错

| 报错 | 原因与处理 |
|---|---|
| `context deadline exceeded` | 登录链接放置过久失效，或回调时网络不通。重新生成并尽快完成。 |
| `Post "https://oauth2.googleapis.com/token": dial tcp ...` | IDE 后端没走代理，用 3.5 的脚本启动。 |
| `invalid_client` | 授权链接被截断，重新完整复制。 |
| 要求短信验证 | 账号侧安全验证，必须本人完成。 |

---

## 五、订阅 Google AI Pro（可选）

### 5.1 免费额度

Tab 补全与命令不限次数；Agent 额度按"干了多少活"计算、**每周刷新**（官方不公布具体数字）；2026 年 11 月起免费档仅保留 Gemini 系模型。重度使用需要订阅。

### 5.2 关键规则：页面标价看 IP，实际扣款看付款资料

> Google One 的**方案页**按当前出口 IP 显示货币（日本 IP 显示日元、美国 IP 显示美元），但**结算时按付款资料所在国家计价**。页面显示日元、付款资料是美国时，最终扣款仍是 **US$19.99/月**。

### 5.3 建立付款资料

1. `pay.google.com` → **新增付款方式**
2. 填写卡号、有效期、CVC；持卡人姓名按卡片填写
3. 国家选 **United States**，账单地址与卡匹配（邮编必须真实存在）
4. 出现 **All set with Google Pay!** 即成功

> 页面是中文时，国家下拉列表按中文排序很难找。可在 URL 后加 `?hl=en` 切英文界面，列表按字母排序，用 `End` 键跳到末尾再向上找 United States 更快。

### 5.4 下单

1. `one.google.com/about/plans` → **Google AI** 标签
2. 选 **Google AI Pro**（**不要选 AI Plus**，对 Antigravity 额度无加成）
3. 结算弹窗确认**币种 USD、金额 $19.99**
4. 确认付款

### 5.5 买完立刻取消自动续订

`play.google.com/store/account/subscriptions` → 找到订阅 → **Manage** → **取消订阅**。服务保留到本周期结束，下周期不再扣款。

> 💳 预付卡/虚拟卡余额往往只够一个月。不取消续订，下月扣款失败会被反复重试，容易导致**卡被封、账号被标记**。

---

## 六、安装 Antigravity CLI（agy）

### 6.1 安装

```powershell
irm https://antigravity.google/cli/install.ps1 | iex
```

装到其他盘：

```powershell
curl.exe -sL https://antigravity.google/cli/install.ps1 -o install.ps1
powershell -NoProfile -File .\install.ps1 -d "E:\tools\agy"
```

约 245 MB，脚本会做 SHA-512 校验。装完 `agy --version` 验证。

### 6.2 包装脚本（负责 PATH + 代理）

```bat
@echo off
rem agy 是 Go 二进制，必须显式设置代理环境变量
set "AGY_EXE=E:\tools\agy\agy.exe"
set "PROXY=http://127.0.0.1:端口"

set "HTTP_PROXY=%PROXY%"
set "HTTPS_PROXY=%PROXY%"
set "ALL_PROXY=%PROXY%"
set "NO_PROXY=localhost,127.0.0.1"

"%AGY_EXE%" %*
endlocal & exit /b %ERRORLEVEL%
```

把该 `.cmd` 放进任何已在 PATH 中的目录（如 npm 全局目录），即可在任意位置直接调用 `agy`。

### 6.3 为什么 agy 必须挂代理

- Go 程序**不读系统代理**，只认 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`
- 直连 `daily-cloudcode-pa.googleapis.com` 等 Google 服务，国内无法访问
- 即使能连上，**Google 还会校验出口 IP 地区**，部分机房 IP 会被拒：`User location is not supported for the API use.`

> 验证代理是否生效：把 `HTTP_PROXY` 指向一个不存在的端口，如果报的是网络错误而不是"地区不支持"，说明环境变量确实生效了。

### 6.4 首次运行

1. **配色方案**选择（方向键 + 回车）
2. **服务条款与数据使用**：中间勾选框是"同意收集交互数据用于改进产品"，**可选**，可取消勾选
3. ↑/↓ 移到 **[Done]** 回车（别停在 [Previous] 上回车，会退回上一步）

> agy 的登录态与 Antigravity IDE 共用，一般不需要重复登录。

### 6.5 常用命令

| 命令 | 作用 |
|---|---|
| `agy` | 交互式界面（类似 Claude Code） |
| `agy -p "问题"` | 单次执行后退出 |
| `agy -c` | 继续上次对话 |
| `agy --mode plan` | 只读规划模式 |
| `agy --add-dir D:\目录` | 追加可访问目录 |
| `agy --model 模型名` | 指定模型 |
| `agy -y` | 自动批准所有工具调用（危险） |
| `agy mcp add/list` | 管理 MCP 服务器 |

---

## 七、让 CLI 认识你的项目

### 7.1 上下文文件

项目根目录放 `GEMINI.md`（agy 自动读取），写清楚项目用途、回答风格、常用命令、注意事项。相当于 Claude Code 的 `CLAUDE.md`。

### 7.2 与 Claude Code 共用技能库

Claude Code 读 `.claude/skills/<技能>/SKILL.md`，agy 读 `.agents/skills/`。用目录联接指向同一份：

```powershell
New-Item -ItemType Junction -Path "D:\myproject\.agents\skills" `
         -Target "D:\myproject\.claude\skills"
```

> Claude 专有字段（`allowed-tools`、hooks、`$ARGUMENTS`）agy 不支持会被忽略；`name` 与 `description` 通用，务必写清楚 `description`。

---

## 八、故障速查表

| 现象 / 报错 | 原因与处理 |
|---|---|
| IDE 一直加载、模型列表加载失败 | IDE 未走代理，用带 `HTTP_PROXY` 的脚本启动 |
| `dial tcp 127.0.0.1:端口: connectex ... refused` | 代理程序没启动或端口变了 |
| 登录后 `context deadline exceeded` | 登录链接过期，重新生成并尽快完成 |
| Gemini 网页提示"所在国家/地区目前不支援 Gemini" | 出口 IP 被判不支持地区，换节点 |
| Google One 提示"你的账号目前无法使用 Google One" | 同上，多为出口 IP 问题 |
| `agy` 报 `Eligibility check failed ... EOF` | 节点不通或被拒；换节点或检查代理环境变量 |
| `agy` 报 `User location is not supported for the API use` | 出口 IP 地区不被接受，换地区节点 |
| `agy` 报 `plan model not specified` | 启动时没拉到模型列表（多为节点瞬时抖动），重试即可 |
| Gemini CLI 报 `This client is no longer supported ...` | 个人版 Gemini CLI 已被关停，与订阅无关，改用 `agy` |
| AI Studio 跳到"可用地区"文档页 | 该账号或 IP 不具备 AI Studio 资格，无法创建免费 API key |
| 注册新账号最后一步"我们无法创建您的 Google 账号" | 出口 IP 被风控（机房 IP 极易触发） |

---

## 九、风险与合规

1. **一人一号**。Google 服务条款不允许为规避额度限制批量注册账号；同设备同手机号的多个账号会被判定关联。
2. **地区一致性**。账号关联地区、付款资料国家、出口 IP 尽量一致。用某地区卡却在另一地区消费，是常见封号触发点。
3. **不要用第三方中转接入**。通过第三方工具转发 Antigravity 登录凭证违反其附加条款，Google 曾据此批量停用账号。
4. **订阅记得取消续订**，预付卡余额不足导致的连环扣款失败会同时伤害卡和账号。
5. **技能/上下文文件里不要放密码、密钥**，这些文件会被模型读取。

---

## 一句话总结

装黑色图标那个 Antigravity IDE → 用带代理环境变量的脚本启动 → 登录时把授权链接复制到已登录的浏览器 → 需要更高额度就订阅 AI Pro（页面币种看 IP、扣款币种看付款资料）→ 命令行用 `agy`（必须给代理环境变量）→ 用目录联接把 Claude Code 的 skills 共享给它。

---

## License

本文档采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.zh) 许可，欢迎转载与改进。
