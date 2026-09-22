# ScanMan Web · 扫描报告前端

ScanMan 的网页端：把后端产出的 JSON 扫描报告渲染成一份可读的源码清单，
也可以直接上传源文件、调用本地 Flask 后端**真跑模型**。

技术栈：React 18 + TypeScript + Vite 6 + Ant Design 5（图表是手写 SVG，没引图表库）。

```text
报告 JSON  ──┐
             ├─→ FileRow[]（唯一一次 join） ─→ 概览 / 四张图 / 函数清单 / 函数详情
上传 + 后端 ──┘
```

------

## 快速开始

```bash
cd web
npm install
npm run dev          # 默认 http://localhost:5173
```

**只看看报告**时不需要 Python、GPU 或模型权重：打开页面点「示例报告」，
或者直接带参数访问 <http://localhost:5173/?r=sample_report.json>。

要在网页上真跑模型，另开一个终端从**仓库根目录**启动后端：

```bash
python scripts/serve.py          # 默认监听 127.0.0.1:8000
```

| 命令                | 作用                                              |
| ------------------- | ------------------------------------------------- |
| `npm run dev`       | 开发服务器（含 `/api` → `127.0.0.1:8000` 的代理） |
| `npm run build`     | `tsc -b && vite build`，产物在 `dist/`            |
| `npm run preview`   | 预览 `dist/` 的构建产物                           |
| `npm run typecheck` | 只做类型检查（`tsc --noEmit`）                    |

> 改了后端端口（`scripts/serve.py --port`）就要同步改
> `vite.config.ts` 里 proxy 的 `target`，否则页面上只会看到「连不上后端」。

------

## 两种数据来源

### 1. 静态示例报告（`public/`）

`public/` 下的 JSON 会被原样发布到站点根路径。`?r=文件名.json` 指定读哪一份，
缺省读 `sample_report.json`；`?sample=1` 等价于「加载默认示例」。
文件名只允许字母、数字、点、下划线、连字符（其它输入会被忽略并回落到默认报告）。

| 地址                                   | 内容                         |
| -------------------------------------- | ---------------------------- |
| `?r=sample_report.json`                | 检测 + CWE 分类（完整链路）  |
| `?r=sample_report_detection_only.json` | 仅检测，没有 CWE             |
| `?r=sample_report_extract_only.json`   | 仅抽取函数，**没有任何判定** |

后两份不是凑数的：它们专门用来验证「没跑过检测」不会被显示成「判定为安全」。
自己生成的报告丢进 `web/public/` 就能用同样的方式打开。

### 2. 真实扫描（上传 → 后端）

页面把文件以 base64 原始字节 POST 到 `/api/scan`，开发环境下由 Vite 代理到
`http://127.0.0.1:8000`（`scripts/serve.py`）。前端一律写相对路径 `/api/*`，
所以开发和「后端托管 `dist/`」两种跑法用同一份代码。

页眉右侧的状态灯一直显示模型的四种状态：正在连接 / 加载中 / 就绪 / 加载失败。
「开始检测」只有在 `ready` 且已选文件时才可点，不能点的原因直接写在按钮下面。

------

## 后端接口

| 接口              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| `GET /api/health` | 模型状态 `loading` / `ready` / `error`，以及设备、检查点、阈值、类别数 |
| `POST /api/scan`  | 请求体 `{ files: [{ client_id, name, content_b64 }] }`，返回 `{ report, uploads }` |

`uploads` 是逐文件的回执（`client_id` / `name` / `path` / `status` / `reason?`），
前端只认它，不猜文件名或下标。

错误码按「谁的问题」分开，界面上给的处理建议也不同：

| 状态码 | 含义                                            | 页面表现                         |
| ------ | ----------------------------------------------- | -------------------------------- |
| `400`  | 上传参数不合法（文件名、扩展名、空文件、超限…） | 直接把后端的原话显示出来         |
| `409`  | 已有扫描在进行                                  | 「等它跑完再试」                 |
| `413`  | 请求体超过 64 MiB                               | 提示少传几个文件                 |
| `503`  | 模型还在加载 / 加载失败                         | 「模型还没准备好：…」            |
| `500`  | 扫描过程异常                                    | 原样显示，同时后端终端有完整堆栈 |

上传限制（前端和后端各挡一道，前端在**选文件时**就挡，不让用户白等一次请求）：

| 项     | 限制                                                         |
| ------ | ------------------------------------------------------------ |
| 文件数 | 一次最多 32 个                                               |
| 单文件 | 不超过 2,000,000 字节（`MAX_FILE_BYTES`）                    |
| 文件名 | 只允许字母、数字、点、下划线、连字符                         |
| 扩展名 | 与 `src/extract.py` 的 `LANGUAGE_BY_EXT` 对齐（C / C++ / Python / JS / TS / PHP） |

------

## 页面结构

页面是「左列选文件 + 右侧看结论」的两栏布局，下钻链是一条直线：

```text
侧栏文件 ─→ 概览 + 四张图 ─→ 选中文件 ─→ 函数清单 ─→ 某个函数 ─→ 源码与判定
```

| 文件                                 | 作用                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| `src/App.tsx`                        | 页面状态机：文件、报告、健康状态、选中项；`rows` 的**唯一一次 join** 也在这里 |
| `src/api.ts`                         | 数据层：`loadHealth` / `runScan` / `loadSampleReport`，统一抛带上下文的 `ApiError` |
| `src/rows.ts`                        | 把「用户选的文件」和「报告里的文件」拼成 `FileRow[]`，含六种文件状态与排序 |
| `src/types.ts`                       | 报告的类型定义，以及字段出现条件的约定（见下节）             |
| `src/components/Header.tsx`          | 标题 + 模型状态灯                                            |
| `src/components/Sidebar.tsx`         | 文件清单、添加 / 移除 / 清空、「开始检测」                   |
| `src/components/Overview.tsx`        | 一个 Hero 数字（疑似有漏洞的函数数）+ 一行 KPI               |
| `src/components/charts/`             | 概率分布直方图、文件风险排行、CWE 分布、文件 × 概率分档热力图 |
| `src/components/FunctionList.tsx`    | 某个文件的函数清单（默认按漏洞概率降序，可切回行号）         |
| `src/components/FunctionDetail.tsx`  | 左侧源码（真实行号 + Prism 高亮）、右侧判定 / Top-K 候选     |
| `src/components/charts/chartkit.tsx` | 图表配色令牌与公共零件（卡片、图例、悬浮读数）               |
| `src/format.ts`                      | 展示层的格式化：百分比、固定小数、千分位、路径缩写           |
| `src/cwe.ts`                         | `CWE-119 → 缓冲区溢出` 的中文简称表（28 类），查不到就只显示编号 |
| `src/timing.ts`                      | `withMinDuration`：扫描动画的最短显示时长                    |
| `src/theme.ts`、`src/index.css`      | Ant Design 主题令牌 + 全局设计令牌                           |

------

## 几条必须遵守的约定

这些不是风格偏好，改错了页面会给出**错误结论**：

1. **可选字段是「键不存在」，不是 `null`。**
   `build_report()` 刻意不写 `null`；判断要写 `'cwe' in fn` / `fn.cwe !== undefined`，
   写 `fn.cwe !== null` 会把「没跑分类」读成「没有类别」。唯一真正会是 `null` 的是顶层函数的
   `parent_name`。
2. **「没跑过」≠「判定为安全」。**
   文件有六种状态（`pending` / `failed` / `empty` / `undetected` / `clean` / `hit`），
   不能简化成 `suspicious_count > 0 ? 红 : 绿`。图表里对应的「没有结论」色是
   `CHART.nodata`，缺了它，一份没跑检测的报告会显示成「全部安全」。
3. **比较风险用 `prob_vulnerable`，不要用 `confidence`。**
   `confidence` 的含义随判定结果变化：判为 `vulnerable` 时等于 `p(漏洞)`，
   判为 `safe` 时却是 `p(安全)`。
4. **上传必须传原始字节（base64），不能先 `file.text()`。**
   后端的抽取器以字节为真相源（这样才处理得了 GBK 等非 UTF-8 文件），
   前端先按 UTF-8 解一次会让字节偏移改变，**`start_line` 全部错位**。
5. **行身份用 `client_id`，不用文件名，也不用下标。**
   用户完全可以选两个都叫 `test.c` 的文件；被跳过的文件不会进 `report.files`，
   按下标对齐会整体错位。
6. **阈值要画在图上。** 直方图里是虚线，热力图里标出所在档，函数详情里画判定线 ——
   只写一个「86.4%」看不出它离判定边界有多远。

------

## 报告与示例报告的再生成

报告的完整字段契约见 [`../docs/08_全流程与接口规范.md`](../docs/08_全流程与接口规范.md)，
顶层字段包括 `schema_version`、`tool`、`generated_at`、`root`、`checkpoint`、
`threshold`、`classifier_checkpoint`、`summary`、`skipped[]`、`files[]`。

重新生成 `public/` 里的示例报告（在**仓库根目录**执行，需要模型权重）：

```bash
# 检测 + CWE 分类
python scripts/scan_project.py demo_verified/ \
    --checkpoint outputs/merged_detection_codebert/best \
    --classifier outputs/merged_top27_codebert_e20/best \
    --output web/public/sample_report.json
```

把 `demo_verified/` 换成目标目录或单个文件即可生成自己的报告，
加不加 `--checkpoint` / `--classifier` 就得到「仅抽取」「仅检测」两种形态。
全部参数见 [`../docs/09_命令参考.md`](../docs/09_命令参考.md) 第 6 节。

------

## 界面上的几条设计取舍

- 报告当**源码清单**渲染，不当卡片仪表盘：等宽标识符、真实行号、细分割线，
  唯一强调色只出现在可交互处。
- 图表配色是算出来的，不是挑出来的：命中 `#b02a1f` ↔ 安全 `#0e8f5a`
  在色盲可辨性、对比度、色度上逐项校验通过。**页面文字的 `--ok` 不能用在图表里**
  （色度偏低，色块里读起来发灰）。
- 全站只有一处连续动画（扫描导轨），且刻意不表示进度：后端一次性返回结果、
  不报进度，走格或回头的动画等于编一个后端没给的数字。
- 「开始检测」有最短显示时长（600ms 下限，不是加法），以免几毫秒返回时
  界面看起来像「点了没反应」；失败也照样等满。

## 相关文档

| 文档                                                         | 内容                                  |
| ------------------------------------------------------------ | ------------------------------------- |
| [`../README.md`](../README.md)                               | 项目总览、模型与实验指标              |
| [`../docs/08_全流程与接口规范.md`](../docs/08_全流程与接口规范.md) | 报告字段契约与字段出现条件表          |
| [`../docs/09_命令参考.md`](../docs/09_命令参考.md)           | 全部脚本的命令与参数（含 Web 前后端） |
| [`../docs/10_全链路贯通手册.md`](../docs/10_全链路贯通手册.md) | 从原始数据到网页点击的完整数据流      |
