# 长江雨课堂自动解题器（cjsolver）

用 Python 通过 CDP 接管本机已装的 Chrome，抓取长江雨课堂的随堂习题，交给
DeepSeek / 千问 / 豆包等大模型作答。支持**只提示**、**自动点击**、**直接调接口**
三种作答模式，自带本地网页控制台与三级模拟检测。

不下载 Playwright 自带浏览器、不需要 `playwright install`——直接接管你的 Chrome，
登录状态可复用。

> ⚠️ 仅供个人学习辅助与技术研究。请遵守学校规定与雨课堂服务条款，
> 不要用它替代自己的诚信作答。

---

## 目录

- [它是怎么工作的](#它是怎么工作的)
- [环境要求](#环境要求)
- [安装](#安装)
- [快速开始](#快速开始)
- [目录结构](#目录结构)
- [作答模式](#作答模式)
- [模型服务](#模型服务)
- [模拟检测](#模拟检测能不能收到题目)
- [网页控制台](#网页控制台)
- [命令行](#命令行)
- [配置](#配置)
- [测试](#测试)
- [常见问题](#常见问题)
- [已知限制](#已知限制)

---

## 它是怎么工作的

```
Chrome(--remote-debugging-port) ──CDP──> Playwright
                                           │
              ┌────────────────────────────┴────────────────────────────┐
              │ ① 网络层（优先）                                        │
              │   • GET /api/v3/lesson/presentation/fetch 响应          │
              │   • wss://<host>/wsapp/ WebSocket 帧                    │
              │   → 拿到 problemType / options / blanks 等结构化字段    │
              └────────────────────────────┬────────────────────────────┘
                                           │ 抓不到时
              ┌────────────────────────────┴────────────────────────────┐
              │ ② DOM 兜底                                              │
              │   扫描「A. / B、/ C）」开头的选项行，聚类出题干与选项    │
              └────────────────────────────┬────────────────────────────┘
                                           ▼
                        DeepSeek / 千问 / 豆包 …（OpenAI 兼容端点）
                                           │
              manual 只提示 ｜ dom 点击作答 ｜ api 调接口
                                           ▼
                     records/session-<时间戳>.jsonl 逐题落盘
```

题目**优先从接口拿**，因为那里有 `problemType`、`options`、`blanks` 等结构化字段，
比从页面上"读"更可靠；DOM 扫描只作兜底。

| 用途 | 接口 |
| --- | --- |
| 进行中的课堂 | `GET /api/v3/classroom/on-lesson` |
| 课件（含随堂题） | `GET /api/v3/lesson/presentation/fetch?presentation_id=…` |
| 提交 / 重答 | `POST /api/v3/lesson/problem/answer`、`/api/v3/lesson/problem/retry` |
| 课堂事件推送 | `wss://<host>/wsapp/` |

支持的站点：

| key | 站点 | 域名 |
| --- | --- | --- |
| `changjiang`（默认） | 长江雨课堂 | `changjiang.yuketang.cn` |
| `standard` | 雨课堂标准版 | `www.yuketang.cn` |
| `pro` | 雨课堂专业版 | `pro.yuketang.cn` |

**题型编码**（接口里的 `problemType`）：

| 编码 | 题型 | 答案提交格式 |
| --- | --- | --- |
| `1` | 单选 | `["A"]` |
| `2` | 多选 | `["A","C"]` |
| `3` | 投票 | `["A"]` |
| `4` | 填空 | `["空1","空2"]` |
| `5` | 主观 | `{"content":"…","pics":[]}` |

---

## 环境要求

- **Python 3.10+**
- **本机已安装 Chrome**（Edge 也在自动探测范围内）

依赖（见 `requirements.txt`）：

```
playwright>=1.44
httpx>=0.27
aiohttp>=3.9        # 仅网页控制台需要
PyYAML>=6.0
rich>=13.7
python-dotenv>=1.0
```

> 注意：`pyproject.toml` 的 `dependencies` 里**没有列 `aiohttp`**（它只用
> `pip install -r requirements.txt` 或 `一键启动.bat` 时才会装上）。
> 用 `pip install .` 装包后跑 `--web` 会缺 aiohttp，手动补一下即可。

---

## 安装

**最简单：双击 `一键启动.bat`。** 它会自动检查 Python、缺依赖就
`pip install -r requirements.txt`，然后进入交互式菜单。

手动安装：

```powershell
python -m pip install -r requirements.txt
```

不需要 `playwright install`——本项目只用 CDP 接管现有 Chrome。

也可以装成命令行工具（此时注意上面提到的 `aiohttp`）：

```powershell
python -m pip install -e .
cjsolver --help
```

---

## 快速开始

```powershell
Copy-Item .env.example .env      # 填入你要用的那家 API Key
python run.py --check            # 自检：配置 / 模型连通性 / 调试端口
python run.py                    # 启动监听（默认 manual 模式）
python run.py --web              # 或：打开网页控制台 http://127.0.0.1:8765/
```

首次运行会自动以调试模式拉起 Chrome，**在这个窗口里扫码登录一次**即可，
登录状态存在专用配置目录 `.chrome-profile/`，不影响你日常的 Chrome。

不想真在上课时试验？用模拟检测：

```powershell
python run.py --simulate offline
```

---

## 目录结构

```
一键启动.bat / 打开控制台.bat     # 双击即用（纯 ASCII，中文 UI 由 Python 输出）
run.py                            # 免安装启动脚本（等价于 cjsolver 命令）
config.yaml                       # 实际配置（已被 .gitignore 忽略）
config.example.yaml               # 带完整注释的模板
.env.example                      # 各服务商 Key 的模板
requirements.txt / pyproject.toml
src/cjsolver/
├── cli.py                        # 命令行入口与子命令
├── launcher.py                   # 交互式菜单 + 初始配置向导
├── solver.py                     # 编排：抓题 → 问模型 → 作答 → 记录
├── submit.py                     # 三种作答方式的落地实现
├── simulate.py                   # 三级模拟检测
├── providers.py                  # 8 家服务商预设与 Key 解析
├── config.py                     # 配置加载、优先级、Chrome 探测
├── models.py                     # Problem / Option / Suggestion 数据模型
├── adhoc.py                      # 手输题目文本的解析
├── envfile.py                    # .env 读写与密钥打码
├── console.py / logo.py          # 终端输出 / 立体 YKT 标题
├── ai/
│   ├── deepseek.py               # OpenAI 兼容客户端（重试、降级）
│   └── prompts.py                # 提示词与答案归一化解析
├── browser/
│   ├── launcher.py               # 以调试模式拉起 Chrome、等待端口就绪
│   └── driver.py                 # CDP 接管、选页、跳课堂、on-lesson 查询
├── web/
│   ├── server.py                 # aiohttp 路由（SSE 事件流）
│   ├── runtime.py                # 控制台运行时（配置、监听、模拟）
│   ├── bus.py                    # 事件总线
│   ├── settings.py               # console-settings.json 持久化
│   └── static/index.html         # 单页前端
└── yuketang/
    ├── capture.py                # 响应 + WebSocket 帧监听、离线投喂
    ├── parsing.py                # 纯函数解析（多字段名兼容）
    ├── dom.py                    # DOM 兜底扫描与选项点击
    └── constants.py              # 站点、API 路径、题型映射
tests/                            # 199 个离线单元测试 + fixtures
tools/                            # 三个冒烟测试
records/                          # 问答记录（JSONL）与页面 dump
```

约 6300 行 Python 源码。

---

## 作答模式

在 `config.yaml`、控制台设置里改，或用 `--mode` 临时覆盖：

| 模式 | 行为 |
| --- | --- |
| `manual`（默认） | 只打印答案、解析、置信度，**完全不碰页面** |
| `dom` | 在页面上按选项文字定位并点击，再点「提交」 |
| `api` | 页面内 `fetch` 直接 POST 作答接口（**实验性**） |

想先看效果又不提交，加 `--dry-run`（任何模式下都不会真正提交）。
`dom` 模式找不到选项元素会**跳过并告警**，不会乱点。
`manual` 以外的模式启动时会有二次确认，`--yes` 可跳过。

---

## 模型服务

全部走 OpenAI 兼容端点，换一家只动 `provider` / `base_url` / `model` 三项。
在控制台「设置 → 模型服务」里可以直接切。

| 服务商 | Base URL | 环境变量 |
| --- | --- | --- |
| **DeepSeek（默认）** | `https://api.deepseek.com` | `DEEPSEEK_API_KEY` |
| 通义千问（阿里云百炼） | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `DASHSCOPE_API_KEY` |
| 豆包（火山方舟） | `https://ark.cn-beijing.volces.com/api/v3` | `ARK_API_KEY` |
| 腾讯混元（元宝同源） | `https://api.hunyuan.cloud.tencent.com/v1` | `HUNYUAN_API_KEY` |
| 智谱 GLM | `https://open.bigmodel.cn/api/paas/v4` | `ZHIPU_API_KEY` |
| Kimi（月之暗面） | `https://api.moonshot.cn/v1` | `MOONSHOT_API_KEY` |
| 硅基流动 | `https://api.siliconflow.cn/v1` | `SILICONFLOW_API_KEY` |
| 自定义 | 自己填 | `AI_API_KEY` |

**每家一个 Key 变量。** 切到千问却还把 DeepSeek 的 Key 发过去只会换来一个 401，
所以程序按当前服务商去取对应变量，缺哪个会明确告诉你要设哪个环境变量。
只想用一个变量就用 `AI_API_KEY`（所有服务商都认，优先级低于专属变量）。

Key 的解析优先级：`config.yaml` / 控制台里**显式填的 Key** > 该服务商的专属变量 > `AI_API_KEY`。

Base URL 一律可手动改——这些地址是外部事实，会变，预设只是省得手输。

> 元宝 App 没有公开 API，表里是同底座的**腾讯混元**接口。

DeepSeek 预设模型为 `deepseek-flash`（快、便宜、支持看图）与 `deepseek-v4-pro`；
旧的 `deepseek-chat` / `deepseek-reasoner` 已停用。

**视觉能力**：题干画在课件图片里时（接口只给 `image_url`），打开
`deepseek.vision_enabled` 即可把图片一并交给模型。`vision_model` 留空就用当前模型
（`deepseek-flash` 本身就支持看图），换别家时填对应视觉模型（如 `qwen-vl-max`）。

---

## 模拟检测：能不能收到题目

**不用真在上课，也不用等老师发题**，就能验证整条链路：

```powershell
python run.py --simulate offline    # 不碰浏览器
python run.py --simulate browser    # 需要 Chrome
```

控制台左栏有三个按钮，验证深度递增：

| | 验证什么 |
| --- | --- |
| ① 离线自检 | 把仿真载荷喂给**真实的解析链路**：题型映射、字段提取、去重、提交格式，并真实调一次模型 |
| ② 浏览器自检 | 往当前页面注入模拟答题卡，验证 DOM 抓题、选项定位与点击、提交按钮识别 |
| ③ 端到端投喂 | 把仿真题目喂给**正在运行的监听会话**，走完抓题 → 问模型 → 作答 → 落盘 |

第 ③ 项只能从控制台发起（`--simulate live` 在命令行下会提示改用控制台）。

仿真载荷**刻意混用了雨课堂历史出现过的一堆字段写法**：选项既有 `{content:…}`
也有纯字符串，题干既有 `prompt` 也有 `content`，id 既有 `problemId` 也有
`problem_id`，WS 帧里用全小写 `lessonid`。上游一改字段，自检会**先于真实课堂**
报出来。

---

## 网页控制台

```powershell
python run.py --web          # 或双击 打开控制台.bat
```

打开 `http://127.0.0.1:8765/`。单页应用，后端 aiohttp，与 Playwright
**共用同一个 asyncio 事件循环**（不跨线程，避免 Playwright 的线程亲和性问题）。

- **实时事件流**：每题一张卡片，答案 / 解析 / 置信度 / 耗时紧随其后
- **带边框日志**：左侧色条按级别着色，带时间戳
- **自动滚动**：默认贴底；滚轮上滑会暂停并浮出「↓ 回到底部」。
  只看用户手势，**不用 `scroll` 事件推断**——程序自己滚动也会触发它，必然误判
- **心跳日志**：监听期间定时输出状态（时长、题目计数、接口/WS 流量、当前页面），
  终端与网页同时可见；间隔看 `solver.heartbeat_interval`（默认 30 秒，`0` 关闭）
- **设置面板**：模式、置信度、DOM 轮询、自动进课堂、窗口尺寸、心跳间隔、
  模型服务、自定义背景

主要接口：`/api/state`、`/api/events`（SSE）、`/api/watch/start|stop`、
`/api/browser/attach|detach`、`/api/simulate`、`/api/ask`、`/api/settings`、
`/api/providers`、`/api/apikey`、`/api/dump`、`/api/probe`。

只监听 `127.0.0.1`，不对外开放。

> 控制台里存的设置会落在 `console-settings.json`，**优先级高于 `config.yaml`**。
> 改了 yaml 没生效时先看这里。密钥不会写进这个文件（它会进仓库），而是写 `.env`。

---

## 命令行

```powershell
python run.py                    # 启动监听（默认 manual）
python run.py --menu             # 交互式一键启动菜单
python run.py --web              # 网页控制台
python run.py --check            # 自检：配置 + 模型连通性 + 调试端口
python run.py --list-sites       # 列出支持的站点
python run.py --simulate offline # 离线抓题自检
python run.py --dump             # 导出页面结构，用于校准选择器
python run.py --ask "单选 1+1=?
A. 1
B. 2"                            # 不开浏览器，直接问题目（- 表示读 stdin）
python run.py --mode dom --yes --duration 1800   # 自动作答，跑 30 分钟
```

`--ask` 支持从纯文本里解析出题干与选项（识别 `A.` / `B、` / `C）` 等写法），
也可以管道传入：

```powershell
Get-Content q.txt | python run.py --ask -
```

**参数一览**

| 参数 | 说明 |
| --- | --- |
| `--config 路径` | 指定配置文件，默认查找 `./config.yaml` |
| `--site` | `changjiang`（默认）/ `standard` / `pro` |
| `--mode` | `manual` / `dom` / `api` |
| `--dry-run` | 只推理，不下发任何页面动作 |
| `--duration 秒` | 到点自动退出，默认一直运行 |
| `--yes` | 自动作答时跳过二次确认 |
| `--web` | 启动网页控制台 |
| `--menu` | 交互式一键启动菜单 |
| `--check` | 自检配置、模型连通性与 Chrome |
| `--ask [文本]` | 直接对一段题目文本作答（`-` = 读标准输入） |
| `--dump` | 导出当前页面 HTML 与扫描结果 |
| `--simulate {offline,browser,live}` | 跑一次模拟检测 |
| `--list-sites` | 列出支持的站点 |
| `--host` / `--port` | 控制台监听地址与端口，默认 `127.0.0.1:8765` |
| `--no-open` | 启动控制台时不自动打开浏览器 |
| `-v / --verbose` | 输出 DEBUG 日志 |

`--web` / `--menu` / `--check` / `--ask` / `--dump` / `--simulate` / `--list-sites`
是互斥的，一次只能用一个。

**交互式菜单**（`--menu` 或双击 `一键启动.bat`）：

| 选项 | 功能 |
| --- | --- |
| 1 | 打开网页控制台（推荐） |
| 2 | 仅提示答案 |
| 3 | 自动点击作答 |
| 4 | 自动作答预演 |
| 5 | 自检 |
| 6 | 手动输入一道题 |
| 7 | 初始配置（写 API Key、选站点） |
| 8 | 打开配置文件（记事本编辑 `config.yaml`） |
| 0 | 退出 |

---

## 配置

`config.yaml` 的完整注释见 `config.example.yaml`。

**优先级：控制台设置 > 命令行 > 环境变量(.env) > `config.yaml` > 内置默认值。**

> 注意：控制台里存过的值写在 `console-settings.json`，会覆盖 `config.yaml`。

常改的几项：

```yaml
site: changjiang                  # changjiang / standard / pro

deepseek:
  provider: deepseek              # 换服务商只动这三项
  model: deepseek-flash
  base_url: https://api.deepseek.com
  temperature: 0.0
  timeout: 90
  max_tokens: 1200
  json_mode: true                 # 用 response_format=json_object
  extra_prompt: ""               # 追加到每次提问的额外要求
  vision_enabled: false           # 题干在图片里时开启
  vision_model: ""

answer:
  mode: manual                    # manual / dom / api
  dry_run: false                  # true 时任何模式都不真正提交
  min_confidence: 0.0             # 低于该置信度只提示，不自动作答
  auto_submit: true               # dom 模式下是否自动点「提交」
  click_delay: 0.8                # 点击选项后的等待秒数
  submit_button_texts: [提交, 确定, 交卷, 提交答案, 确认提交]
  option_selectors: []            # 留空用内置启发式；可填 CSS 选择器精确定位选项
  question_selectors: []          # 同上，用于 DOM 兜底抓题

solver:
  poll_interval: 2.0              # DOM 兜底轮询间隔（秒）
  dom_fallback: true              # 网络抓不到题时启用 DOM 兜底
  auto_enter_lesson: false        # 检测到正在上课时自动跳转课堂页
  enter_lesson_interval: 20.0
  heartbeat_interval: 30.0        # 心跳日志间隔；0 关闭
  record_dir: records             # 问答记录（JSONL）输出目录
  log_level: INFO

browser:
  cdp_url: http://127.0.0.1:9222  # Chrome 远程调试地址
  chrome_path: ""                 # 留空则自动探测（含 Edge）
  user_data_dir: .chrome-profile  # 专用配置目录（首次需在此窗口登录一次）
  auto_launch: true               # 端口未监听时自动以调试模式拉起 Chrome
  launch_timeout: 40              # 等调试端口就绪的秒数
  window_width: 800               # 启动监听时缩窗口，0 = 不调整
  window_height: 600
```

除 Key 之外的少量项也可以用环境变量覆盖：`DEEPSEEK_BASE_URL`、`DEEPSEEK_MODEL`、
`CJ_PROVIDER`、`CJ_CDP_URL`、`CJ_SITE`。
（`DEEPSEEK_API_KEY` **故意不做映射**——它只属于 DeepSeek 一家，交给服务商解析逻辑处理。）

**记录落盘**：每次运行在 `records/session-<时间戳>.jsonl` 逐题记录题目、答案、解析、
置信度、耗时、提交结果。

**选择器失效时**（雨课堂改版）：`python run.py --dump` 导出页面结构（`page.html`）
和扫描结果（`scan.json`），把选项 / 题干容器的 class 填进
`answer.option_selectors` / `answer.question_selectors` 即可精确定位。

---

## 测试

```powershell
python -m pytest -q                # 199 个离线单元测试
python tools/smoke_browser.py      # 浏览器层：注入仿真答题卡，验证扫题与点击
python tools/smoke_console.py      # 控制台：真浏览器里量自动滚动/暂停/回到底部
python tools/smoke_live.py         # 端到端：Chrome + 真实模型，一路验到事件流
```

单元测试全部离线：不访问雨课堂、不需要 Chrome、不需要 API Key
（网络与浏览器交互由 fixtures 和桩替代）。`tests/fixtures/` 下有
`presentation_sample.json` 与 `websocket_sample.json` 两份真实载荷样本。

三个冒烟测试都开真 Chrome、都不访问雨课堂、不修改任何线上数据；
`smoke_live.py` 需要有效的 API Key。

> 如果 `python -m pytest` 报 `No module named pytest`，先装开发依赖：
> `python -m pip install -e ".[dev]"` 或 `python -m pip install pytest pytest-asyncio`。

---

## 常见问题

**调试端口起不来 / 等待超时**

该 `user_data_dir` 已被另一个 Chrome 占用。关掉对应窗口，或换一个目录。
也可以手动启动 Chrome 后把 `browser.auto_launch` 设为 `false`：

```powershell
& "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  --remote-debugging-port=9222 --user-data-dir="D:\chrome-debug"
```

**401 / 鉴权失败**

`.env` 里的 Key 无效或已失效，或用了**别的服务商**的 Key。注意要填**当前服务商**
对应的变量（切到千问就得填 `DASHSCOPE_API_KEY`，或改用通用的 `AI_API_KEY`）。

**抓到题了但答案总是空**

模型没按 JSON 输出。程序已做降级解析（兼容 `答案：X` 这类老格式）；仍失败就换个
模型，或在 `deepseek.extra_prompt` 里强调「只输出 JSON」。

如果用的是不支持 `response_format` 的端点，客户端会自动去掉该参数重试一次。

**不想让它自动点提交**

`answer.auto_submit: false`，只点选项，提交交给你。

**答案提交后没反应**

`api` 模式依赖的接口字段可能已变，属于实验性。改用 `dom`——那条路径走的是页面
自己的代码，更稳。

**改了 `config.yaml` 没生效**

控制台里存过的设置写在 `console-settings.json`，优先级更高。在控制台里改回来，
或删掉那个文件。

**控制台报 `No module named aiohttp`**

用 `pip install .` 装的包不含 aiohttp（`pyproject.toml` 里没列）。补装：
`python -m pip install aiohttp`。

---

## 已知限制

- `api` 模式的作答接口字段结构可能有版本差异，属于实验性。
- DOM 扫描依赖「选项以 `A.` / `B、` / `C）` 开头」这个视觉约定，改版后需用
  `option_selectors` 校准。
- 主观题和纯填空题的页面落点无法靠文字可靠定位，`dom` 模式下**不自动填写**，
  只打印答案。
- 使用 `manual` 以外的模式前，请确认当前页面确实是你本人的课堂。
- 图片题需要显式打开 `vision_enabled`，且所用模型得支持读图。
