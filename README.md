# 智慧社区物业系统 UI 自动化测试项目

基于 **Python + Selenium + Pytest + Allure** 的端到端（E2E）UI 自动化测试项目，针对一套纯前端的「智慧社区物业系统」（前台业主端 + 物业后台端）进行测试。

项目采用经典的**五层自动化测试架构**（基础层 → 页面对象层 → 业务层 → 数据层 → 用例层），全程使用**显式等待（WebDriverWait）**，无任何 `time.sleep()` 硬等待，并实现了 **Allure 报告 + 失败自动截图**、**JSON 数据驱动**。

> 项目亮点：在测试执行过程中发现并修复了 **2 个产品级真实缺陷**（详见 [缺陷挖掘与修复记录](#-缺陷挖掘与修复记录)），完整覆盖了「正向 → 逆向 → 异常校验 → 状态流转」的测试思维。

---

## 一、被测系统简介

被测对象是一套纯前端实现的智慧社区物业系统（无真实后端，数据存储在浏览器 LocalStorage）：

| 端 | 文件 | 角色 |
|---|---|---|
| 前台 | `项目信息/社区服务前台.html` | 业主登录、在线报修 |
| 后台 | `项目信息/智慧社区服务平台2.0.html` | 物业管理员登录、工单分配 / 完成 / 驳回 |

**测试难点**：前后台是两个独立的本地 HTML 文件，数据通过 LocalStorage 传递——因此前台提交和后台读取必须在**同一个浏览器会话**内完成，且 Chrome 启动时必须携带 `--allow-file-access-from-files` 参数打通本地文件的存储隔离。

## 二、五层架构设计

```
Smartcommunity_Auto/
├── config.json                  # ① 配置层：前后台 URL、登录账号
├── test_data.json               # ① 数据层：测试数据（含 automation_test_cases 节点）
├── pages/                       # ②③ 页面对象层 + 基础层
│   ├── base_page.py             #    基础层：WebDriver 初始化、显式等待、点击/输入/取文本等通用封装
│   ├── front_page.py            #    页面对象：前台登录、在线报修（含 Select 下拉框处理）
│   └── backend_page.py          #    页面对象：后台登录、工单分配/完成/驳回（含原生 Alert 处理）
├── testcases/                   # ④ 驱动层 + ⑤ 用例层
│   ├── conftest.py              #    驱动层：Pytest fixture 管理浏览器启停 + 失败自动截图钩子
│   └── test_e2e_flow.py         #    用例层：6 条 E2E 用例，从 JSON 读取数据实现数据驱动
├── api_test/                    # 接口测试脚本（登录接口）
├── 项目信息/                     # 被测系统源码 + 需求文档
└── report/                      # Allure 测试结果（gitignore，本地生成）
```

**分层职责**：

1. **基础层（base_page.py）**：封装 webdriver-manager 驱动管理、显式等待、元素点击、输入（带键盘全选删除兜底）、文本获取等通用操作；
2. **页面对象层（front_page / backend_page）**：元素定位以 `id` 优先、`xpath`/`css` 兜底，封装为元组，并提供 `login()`、`submit_repair()`、`assign_order()` 等业务方法；
3. **配置/数据层（config.json / test_data.json）**：URL、账号与测试数据全部外置，修改数据无需改代码；
4. **驱动层（conftest.py）**：`pytest fixture` 管理浏览器生命周期；实现 `pytest_runtest_makereport` 钩子，**用例失败时自动截图并附加到 Allure 报告**；
5. **用例层（test_e2e_flow.py）**：只写业务流程与断言，不含任何定位细节。

## 三、核心测试场景（6 条 P0 用例）

| # | 类型 | 用例 | 步骤 | 断言 |
|---|------|------|------|------|
| 1 | 正向 | 前台在线报修 | 前台登录 → 在线报修提交（水管/紧急描述） | 右上角显示业主姓名；Toast「报修提交成功」；LocalStorage 中新增工单且状态=待分配 |
| 2 | 正向·核心链路 | 报修全流程流转 | 前台提交报修 → **同一会话**跳转后台 → 后台登录 → 物业服务 → 分配王师傅 → 完成工单 | 后台表格出现该工单；分配后状态=处理中；确认后状态=已完成 |
| 3 | 逆向 | 后台错误密码登录 | admin + 错误密码 | Toast「用户名或密码错误」，仍停留在登录页 |
| 4 | 逆向 | 前台错误密码登录 | zhangyezhu + 错误密码 | 错误提示出现，未跳转（登录按钮仍可见） |
| 5 | 异常校验 | 报修空描述提交 | 登录后打开报修弹窗，清空描述直接提交 | 拦截提示「请填写问题描述」；弹窗未关闭；LocalStorage 无脏数据 |
| 6 | 状态流转 | 工单驳回 | 后台找到「处理中」工单 → 驳回 → 确认原生 confirm | 状态标签回退为「待分配」 |

## 四、🔧 缺陷挖掘与修复记录

自动化测试不只是「跑脚本」，在编写与执行过程中发现并修复了以下 **2 个产品级真实缺陷**：

### 缺陷 1：前台「报修提交」功能完全失效（严重）

- **现象**：填写完报修弹窗点击「提交」，页面无任何反应，LocalStorage 无数据写入。
- **定位过程**：通过 Selenium 执行 JS 点击 + 捕获浏览器控制台日志，抓到 `Uncaught TypeError: Cannot read properties of undefined`。
- **根因**：页面存在**重复 ID**——外层弹窗容器 `<div id="modalContent">` 与内部描述输入框 `<textarea id="modalContent">` 同名。页面提交函数中 `document.getElementById('modalContent').value` 取到的是外层 div（其 `.value` 为 `undefined`），`.trim()` 直接抛异常导致整个提交逻辑中断。**手工功能测试同样会复现此 Bug。**
- **修复**：将外层容器改名为 `modalBox`（同步修改全部引用），`textarea` 保留原 id。
- **测试价值**：`By.ID` 定位在重复 ID 场景下会静默取第一个元素——自动化脚本最初也踩了这个坑（报 `invalid element state`），随后改用 `textarea#modalContent` CSS 选择器精确定位，与缺陷根因一一呼应。

### 缺陷 2：前后台 LocalStorage 数据不互通，跨页面流程无法流转（严重）

- **现象**：前台提交报修成功后，后台工单列表完全看不到新工单。
- **根因**：后台 HTML 中**没有任何读取 LocalStorage 的代码**，工单列表是写死的内存数组——与需求文档「前台产生数据、后台消费数据」的设计意图不符。
- **修复**：在后台页面加载逻辑中加入数据同步补丁，启动时读取 `localStorage.repairOrders` 合并进工单列表。

### 缺陷 3：报修提交缺少空内容校验（一般）

- **现象**：描述留空直接提交，前端自动用「无详细描述」兜底并提交成功，属于异常场景缺乏拦截。
- **修复**：在前台提交函数中加入空内容校验：描述为空时 Toast 提示「请填写问题描述」并阻止提交（对应用例 5）。

## 五、如何运行

### 1. 安装依赖

```bash
pip install selenium webdriver-manager pytest allure-pytest
```

### 2. 运行测试

在项目根目录执行：

```bash
pytest testcases/ -v --alluredir=./report/allure-results
```

### 3. 生成并查看 Allure 报告

需先安装 Allure 命令行（[allure2 下载](https://github.com/allure-framework/allure2/releases)，Windows 可用 `scoop install allure`）：

```bash
# 方式一：生成报告后自动打开
allure serve ./report/allure-results

# 方式二：生成静态报告到指定目录
allure generate ./report/allure-results -o ./report/allure-report --clean
allure open ./report/allure-report
```

报告包含每条用例的步骤、附件，**失败用例会自动附带浏览器现场截图**。

### 4. 环境说明与已知坑

- 本机无 Chrome 时，`conftest.py` 会自动回退到 Edge（Chromium 内核，启动参数通用）；
- Chrome 启动参数必须带 `--allow-file-access-from-files`，否则两个本地 HTML 的 LocalStorage 相互隔离，跨页面用例必失败；
- **前后台操作必须在同一个浏览器实例内完成**（纯前端项目，关闭浏览器 LocalStorage 数据即丢失），用例 2 通过 `driver.get()` 在会话内跳转实现；
- 若系统配置了 `HTTP_PROXY` 等代理变量，会劫持 Python 与本机 WebDriver 的通信（报 `unhandled request`），`conftest.py` 已在启动时自动清理；
- 下拉框统一使用 Selenium `Select` 类处理；原生 `confirm()` 弹窗通过 `switch_to.alert.accept()` 处理。

## 六、当前测试结果

```
testcases/test_e2e_flow.py::TestE2EFlow::test_01_front_submit_repair PASSED        [ 16%]
testcases/test_e2e_flow.py::TestE2EFlow::test_02_e2e_repair_assign_complete PASSED [ 33%]
testcases/test_e2e_flow.py::TestE2EFlow::test_03_backend_login_wrong_password PASSED [ 50%]
testcases/test_e2e_flow.py::TestE2EFlow::test_04_front_login_wrong_password PASSED [ 66%]
testcases/test_e2e_flow.py::TestE2EFlow::test_05_front_repair_empty_description PASSED [ 83%]
testcases/test_e2e_flow.py::TestE2EFlow::test_06_backend_reject_order PASSED       [100%]

======== 6 passed in 151.90s ========
```
