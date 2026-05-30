---
name: complex-web-tasks
description: 处理复杂的网页任务 — 多步骤网页交互、表单填写、数据爬取、登录流程、分页抓取、动态内容提取等。结合浏览器自动化、网页提取、Python脚本等多种工具，应对需要多步推理的网页场景。
---

# Complex Web Tasks

适用于需要多步骤交互、跨页面操作、或复杂数据提取的网页任务。

**核心原则：**
1. **单实例** — 同一域名只开一个页面，不重复多开
2. **防人机验证** — 操作节奏、交互方式要模拟真人
3. **实时保存** — 每步操作后立即保存数据，避免丢失
4. **高效填表** — 先扫描控件类型，从最快的方案开始尝试（URL参数 > DOM设值 > 键盘事件 > 模拟点击 > 可视化操作）
5. **先扫描后执行** — 先打开页面分析结构，再制定详细计划

---

## 什么时候用

- 需要登录网站后才能抓取数据
- 需要填写表单、提交、等待结果
- 分页数据抓取（翻页、无限滚动、加载更多）
- 网站数据需要拼接多个页面才能获得完整信息
- 反爬措施需要处理（Cookie、User-Agent、延迟）
- 搜索结果多页批量收集
- 网站有验证码/CAPTCHA（需用户介入）
- 需要比较多个网站的数据
- 需要监控网页变化并通知

---

## 工具选择策略

| 场景 | 工具 |
|------|------|
| 静态页面、内容提取 | `web_extract` |
| 搜索信息 | `web_search` |
| 需要浏览器渲染、JS、交互 | Browser skill（用 open-gstack-browser 加载 Browser） |
| 用户已在 Chrome 打开页面，需远程操作 | 通过 CDP 连 Windows Chrome（见 references/connect-windows-chrome.md） |
| 多步抓取 + 数据处理 | `execute_code` （Python + requests/httpx + BeautifulSoup） |
| 定时监控 | `cronjob` |

---

## 通用工作流

### 铁律：进入新页面 → 先问需求 → 再思考 → 最后执行

**在任何网页操作任务中，必须严格遵守以下顺序，一步不能乱：**

```
进入新的网页界面
    ↓
第1步：问清楚用户要什么（需求确认）
    ↓
第2步：根据需求思考最优方案（效率规划）
    ↓
第3步：扫描页面获取结构信息（信息采集）
    ↓
第4步：输出执行计划并让用户确认（计划确认）
    ↓
第5步：严格执行计划（执行）
    ↓
第6步：验证结果（验收）
```

#### 为什么这个顺序不能乱？

因为跳过任何一步都会导致：
- **跳过第1步** → 方向错了，做了用户不想要的事
- **跳过第2步** → 用了低效的方法，白白浪费时间
- **跳过第3步** → 不知道页面有什么字段，不知道怎么填
- **跳过第4步** → 用户不知道你要做什么，中间发现不对时已经晚了
- **跳过第5步** → 顺序搞错，先填后选的会导致表单验证失败
- **跳过第6步** → 提交了但不知道成功没有

---

### 第1步：进入新页面先问需求

**当跳转到一个新的网页界面时，不要直接操作，先停下来问用户：**

```
"进入这个页面了，你需要我做什么？
1. 填写表单？
2. 提取页面上的信息？
3. 点击某个按钮/链接？
4. 监控/等待某个变化？
5. 还是有其他需求？"
```

**具体案例：**

```
场景：用户让打开一个商品编辑页

❌ 错误做法：直接开始填写所有字段
✅ 正确做法：
   "商品编辑页已打开，你要我：
    A）修改商品标题和价格
    B）提取商品信息保存下来
    C）看看页面有什么问题
    D）其他需求"
```

**如果是提取信息类任务：**
```
"这个页面你想提取哪些信息？
例如：
- 表格中的数据（哪些列？）
- 商品的标题/价格/描述
- 图片链接
- 页面上的特定文字/数据
- 还是提取所有可见内容？"
```

---

### 第2步：根据需求思考最优方案

在动手之前，先想清楚用什么方法效率最高：

**思考框架：**

```
用户需求是什么？
    ↓
这个目标有几种实现方式？
    ↓
哪种最快？哪种最稳？
    ↓
选择最优方案
```

**不同任务的最优方案参考：**

| 任务类型 | 最优方案 | 备选方案 | 
|---------|---------|---------|
| **提取列表中所有数据** | 先看页面总数，用URL翻页 | 逐个点击查看 |
| **提取单条详情** | 直接提取当前页面内容 | — |
| **填写表单多个字段** | 批量DOM操作一次性填完 | 逐个字段填入 |
| **多步骤操作（先A后B）** | 按步骤串行执行，每步验证 | — |
| **提取图片/文件** | 获取下载链接批量下载 | 逐个保存 |
| **搜索+提取** | URL参数直接跳转到结果页 | 在页面内搜索 |

**信息提取的思考方式：**
```
要提取的信息是什么类型的？
  → 表格数据 → 获取表格所有行，结构化保存
  → 列表数据 → 翻页提取所有条目
  → 单条详情 → 直接获取当前页关键字段
  → 图片/文件 → 获取URL列表批量下载
  
提取后怎么处理？
  → JSON文件保存
  → CSV表格保存
  → 直接展示给用户
  → 作为下一步操作的输入
```

---

### 第3步：扫描页面获取结构信息

确认需求和方案后，开始扫描页面，获取操作所需的所有信息：

**扫描的重点因任务类型而异：**

**如果是填表任务：** 扫描所有表单字段、类型、必填状态、当前值
**如果是提取任务：** 扫描数据所在的位置、格式、数量、翻页方式
**如果是点击/导航任务：** 扫描目标按钮/链接的位置和状态

详见下方「第1步：扫描页面 → 制定计划」章节。

---

### 第4步：输出执行计划

扫描完成后，输出清晰的执行计划，让用户确认后再执行：

**填表任务的计划：**
```
═══════════════════════════════════════════════
页面：[页面名称]
需求：[用户说要做什么]
═══════════════════════════════════════════════

[步骤] 操作内容
───────────────────────────────────────────
 1    填写「商铺名称」→ 集创数据开发工作室
 2    选择「所在地区」→ 河南省/新乡市/红旗区
 3    填写「详细地址」→ 骆驼湾...
 4    填写「商铺详情」→ 已有内容，保留
 5    点击「确认保存」按钮
───────────────────────────────────────────
总计 5 步，预计 30 秒
```

**信息提取任务的计划：**
```
═══════════════════════════════════════════════
页面：[页面名称]
需求：提取[某类信息]
═══════════════════════════════════════════════

[阶段1] 获取列表
  1.1 提取当前页所有条目
  1.2 翻到下一页继续提取
  1.3 直到所有页完成

[阶段2] 保存结果
  2.1 整理为JSON/CSV格式
  2.2 保存到 /tmp/xxx.json

预计数据量：约 N 条
预计耗时：约 N 分钟
保存路径：/tmp/xxx.json
```

**让用户确认后再执行。**

---

### 第5步：严格执行计划

按照计划逐步执行，**不要跳步、不要改变顺序**。

**执行纪律：**
- 一次只做一步
- 每一步完成后验证是否成功
- 如果某步失败，暂停并分析原因，调整后继续
- 绝不跳过失败的步骤

**填表类任务的操作顺序：**
```
1. 先填文本字段（标题、描述、价格等）
2. 再选下拉选项（分类、地区等）  
3. 最后处理上传操作（图片、文件等）
4. 检查所有必填字段
5. 点击提交按钮
```
这个顺序不能乱，因为某些表单在提交前需要所有字段完整。

**信息提取类任务的操作顺序：**
```
1. 先获取总页数/总条目数
2. 提取第一页数据 → 实时保存
3. 翻到下一页 → 提取 → 保存
4. 重复直到全部完成
5. 汇总所有数据 → 输出最终文件
```

---

### 第6步：验证结果

操作完成后，必须验证结果：

**填表类验证：**
- 提交后页面是否有成功提示？
- 返回查看已提交的内容是否完整？
- 字段是否和填写的一致？

**信息提取类验证：**
- 提取的条目数量是否和页面一致？
- 关键字段是否有缺失？
- 数据格式是否正确（编码、换行等）？
- 输出文件是否可以正常打开？

---

### 流程示意图

```
用户说"帮我做XX"
    ↓
[第1步] 问清需求
    ↓ 用户说"填写表单"或"提取信息"
[第2步] 想最优方案
    ↓
[第3步] 扫描页面结构
    ↓
[第4步] 输出计划并让用户确认
    ↓ 用户说"开始"
[第5步] 按计划执行
    ↓
[第6步] 验证结果
    ↓ 全部完成
告诉用户结果
```

**绝对禁止：**
- ❌ 没问需求就直接填
- ❌ 没想方案就直接操作
- ❌ 没扫描就填写未知字段
- ❌ 没计划就盲目执行
- ❌ 打乱操作顺序
- ❌ 不验证就声称完成

---

### 第1步：扫描页面 → 制定计划

这是最关键的一步。**在任何实际操作前，必须先打开目标网站扫描页面结构，然后制定详细的执行计划。**

#### 深度扫描：逐块解析页面结构

不仅要看页面有什么，还要**明确每个区域、每个元素的意义和作用**。复杂页面上可能有几十个元素，需要区分哪些是数据、哪些是操作、哪些是干扰。

**扫描时应回答的核心问题：**
```
对于页面上的每一个区域/控件/按钮，搞清楚：
  "这是干什么的？"
  "我需要操作它吗？"
  "如果不操作会怎样？"
```

#### 扫描方法论：区块划分 + 意义识别

```
开页后，先把页面划分成功能区块：

┌─────────────────────────────────────────┐
│  头部导航区（主导航、用户信息、设置入口）  │
│  → 找：登录入口、个人中心、退出按钮       │
├─────────────────────────────────────────┤
│  筛选/搜索区（条件筛选、关键词搜索）       │
│  → 找：搜索框、筛选下拉、分类标签          │
├─────────────────────────────────────────┤
│  列表/内容区（数据展示、条目列表）         │
│  → 找：条目数量、翻页方式、每个条目包含的字段 │
├─────────────────────────────────────────┤
│  操作区（新增、编辑、删除、导出）          │
│  → 找：按钮、链接、批量操作入口            │
├─────────────────────────────────────────┤
│  侧边栏/辅助区（推荐、广告、相关信息）      │
│  → 判断：是功能相关还是干扰信息？          │
├─────────────────────────────────────────┤
│  表单区（如果是填写任务）                 │
│  → 逐字段分析：每个字段要填什么、不填会怎样  │
├─────────────────────────────────────────┤
│  弹窗/浮层/通知区（确认框、提示、验证码）   │
│  → 预判：操作后可能出现什么弹窗？           │
└─────────────────────────────────────────┘
```

#### 执行步骤

**第1步：区块划分扫描**

用 Browser 一次性扫描页面，用语义标签和类名识别区域：

```javascript
// 快速识别页面区块
const regions = {
  header: document.querySelector('header, .header, .navbar, .top-nav, [role="banner"]'),
  sidebar: document.querySelector('aside, .sidebar, .side-nav, [role="complementary"]'),
  main: document.querySelector('main, .main, .content, .main-content, [role="main"]'),
  footer: document.querySelector('footer, .footer, [role="contentinfo"]'),
  search: document.querySelector('.search, .search-box, .search-bar, input[type="search"]'),
  filter: document.querySelector('.filter, .filters, .filter-bar, .condition'),
  form: document.querySelector('form, .form, [role="form"]'),
  pagination: document.querySelector('.pagination, .page-nav, .pager, [aria-label*="pagination"]'),
  table: document.querySelector('table, .table, .data-table, .grid, [role="grid"]'),
  modal: document.querySelector('.modal, .dialog, .overlay, [role="dialog"]'),
  toast: document.querySelector('.toast, .message, .alert, .notification'),
};

Object.entries(regions).forEach(([name, el]) => {
  if (el) console.log(`✅ ${name}:`, el.tagName, el.className?.slice(0, 60));
});
```

**第2步：逐元素标注"这是什么、干什么的"**

对表单元素和交互控件，逐一标注其意义：

```javascript
// 标注页面上所有交互元素的用途
document.querySelectorAll('input, select, textarea, button, a.btn, [role="button"], [role="combobox"], [role="radio"]')
  .forEach(el => {
    const label = document.querySelector(`label[for="${el.id}"]`);
    const nearby = el.closest('.form-item, .field, .input-group');
    const nearbyText = nearby ? nearby.textContent.trim().slice(0, 20) : '';
    
    console.log({
      tag: el.tagName,
      type: el.type || el.getAttribute('role'),
      name: el.name || el.id,
      label_text: label?.textContent?.trim() || '',
      nearby_text: nearbyText,
      placeholder: el.placeholder || '',
      value: el.value?.slice(0, 30) || '',
      required: el.required || el.getAttribute('aria-required'),
      // 判断是什么控件
      control_type: classifyControl(el),
      // 意义判断
      purpose: inferPurpose(el, label, nearby),
      // 是否必操作
      must_operate: el.required || label?.textContent?.includes('*') || false,
    });
  });

function classifyControl(el) {
  const tag = el.tagName.toLowerCase();
  const type = (el.type || '').toLowerCase();
  const role = el.getAttribute('role') || '';
  if (tag === 'select') return '下拉菜单';
  if (tag === 'textarea') return '多行文本';
  if (type === 'text' || type === 'email' || type === 'tel') return '文本输入';
  if (type === 'password') return '密码输入';
  if (type === 'number') return '数字输入';
  if (type === 'date' || type === 'datetime-local') return '日期选择';
  if (type === 'checkbox') return '复选框';
  if (type === 'radio') return '单选框';
  if (type === 'file') return '文件上传';
  if (type === 'submit') return '提交按钮';
  if (type === 'hidden') return '隐藏字段';
  if (role === 'combobox' || role === 'listbox') return '自定义下拉';
  if (role === 'radio') return '自定义单选';
  if (role === 'tab') return 'Tab标签';
  if (el.closest('.ql-editor, .ProseMirror, [contenteditable]')) return '富文本编辑器';
  if (el.closest('.ant-select, .el-select')) return '框架下拉';
  if (el.closest('.ant-cascader, .el-cascader')) return '级联选择';
  if (el.closest('.ant-picker, .el-date-editor')) return '日期选择器';
  return '未知控件';
}

function inferPurpose(el, label, nearby) {
  // 通过标签文本、nearby文本、placeholder推断用途
  const text = (label?.textContent || nearby?.textContent || el.placeholder || '').toLowerCase();
  
  if (text.includes('标题') || text.includes('title') || text.includes('name')) return '商品/内容标题';
  if (text.includes('价格') || text.includes('price') || text.includes('金额')) return '价格/金额';
  if (text.includes('库存') || text.includes('stock') || text.includes('数量')) return '库存数量';
  if (text.includes('分类') || text.includes('类别') || text.includes('category')) return '商品分类';
  if (text.includes('描述') || text.includes('描述') || text.includes('description') || text.includes('详情')) return '商品/内容描述';
  if (text.includes('标签') || text.includes('tag') || text.includes('关键词') || text.includes('keyword')) return '标签/关键词';
  if (text.includes('图片') || text.includes('图片') || text.includes('image') || text.includes('upload')) return '图片上传';
  if (text.includes('颜色') || text.includes('color') || text.includes('规格')) return '商品规格/属性';
  if (text.includes('运费') || text.includes('shipping') || text.includes('物流')) return '运费设置';
  if (text.includes('时间') || text.includes('date') || text.includes('上架')) return '时间日期';
  if (text.includes('搜索') || text.includes('search')) return '搜索框';
  if (text.includes('邮箱') || text.includes('email')) return '邮箱输入';
  if (text.includes('手机') || text.includes('phone') || text.includes('电话')) return '手机号';
  if (text.includes('密码') || text.includes('password')) return '密码';
  if (text.includes('用户名') || text.includes('username')) return '用户名';
  if (text.includes('验证码') || text.includes('captcha') || text.includes('code')) return '验证码';
  if (text.includes('提交') || text.includes('保存') || text.includes('submit') || text.includes('save')) return '提交/保存按钮';
  if (text.includes('删除') || text.includes('delete') || text.includes('移除')) return '删除操作';
  if (text.includes('编辑') || text.includes('edit') || text.includes('修改')) return '编辑操作';
  if (text.includes('新增') || text.includes('新建') || text.includes('创建') || text.includes('add') || text.includes('create')) return '新增操作';
  if (text.includes('下一页') || text.includes('next')) return '翻页-下一页';
  if (text.includes('上一页') || text.includes('prev')) return '翻页-上一页';
  
  return `未知用途，需人工判断（附近文本: ${(label?.textContent || nearby?.textContent || el.placeholder || '无').slice(0, 30)}）`;
}
```

#### 第3步：输出页面结构清单

扫描完成后，输出结构化的页面清单：

```
═══════════════════════════════════════════════
页面结构分析：[URL]
═══════════════════════════════════════════════

[区块1] 头部导航区 (header.navbar)
  ├── 🔘 登录入口 (a.login-btn)：点击跳转登录页
  └── 🔍 搜索框 (input.search)：搜索商品

[区块2] 筛选区 (div.filter-bar)
  ├── 📋 分类下拉 (select#category)：选商品分类 → 影响列表内容
  ├── 📋 排序下拉 (select#sort)：按价格/销量排序
  └── 🔘 筛选按钮 (button#filter)：确认筛选条件

[区块3] 列表区 (div.product-list)
  ├── 共 50 条商品，每页 10 条
  ├── 翻页方式：URL参数 ?page=N
  ├── 📝 商品1：标题 / 价格 / 图片 / 编辑按钮
  ├── 📝 商品2：标题 / 价格 / 图片 / 编辑按钮
  └── ...

[区块4] 分页区 (div.pagination)
  ├── ⬅ 上一页 (a.prev)
  └── ➡ 下一页 (a.next)

═══════════════════════════════════════════════
表单字段明细（如果是填表任务）：
───────────────────────────────────────────
字段名     类型       必需  用途                  用户要求
───────────────────────────────────────────
title      text       ✅   商品标题               用户说"标题写新款跑鞋"
price      number     ✅   商品价格               用户说"价格299"
category   select     ✅   商品分类               ❓用户没说，自动推导或询问
stock      number     ❌   库存数量                用户没说，模式B自动补全
desc       textarea   ❌   商品描述                用户没说，模式B自动生成
images     file       ❌   商品图片                用户说"上传生成的图片"
───────────────────────────────────────────
```

#### 第4步：意义不明确的元素如何处理

扫描时遇到意义不明的元素，用以下方法推断：

**方法A：看周边文本**
```javascript
// 找元素附近最近的文本标签
const el = document.querySelector('select[name="s"]');
// 向上找最近的 label 或 form-item
const label = el.closest('.form-item, .field')?.querySelector('label, .label, .field-label');
```

**方法B：看placeholder**
```javascript
// placeholder 本身说明了这个字段的用途
el.placeholder // 如 "请输入商品标题"、"8-16位密码"
```

**方法C：看价值和预设值**
```javascript
// value和option可以推断这个字段是什么
// select的options → 选项列表
Array.from(el.options).map(o => o.text) // ["男装","女装","童装"] → 分类字段
```

**方法D：看页面标题和上下文**
```javascript
// 页面标题可以推断当前是什么页面
document.title // "编辑商品 - XXX平台" → 当前是编辑商品页
// URL路径也可以推断
window.location.pathname // /product/add → 新增商品 /product/123/edit → 编辑商品
```

**方法E：实在不确定 → 标记为"需确认"**
```javascript
// 无法推断用途的元素，标记出来供用户确认
const uncertainFields = [];
// ... 收集无法推断的字段
console.log('⚠️ 以下字段用途不明，需要确认：', uncertainFields);
```

---

#### 输出执行计划

扫描完成后，输出一个清晰的执行计划，格式如下：

扫描完成后，输出一个清晰的执行计划，格式如下：

```
═══════════════════════════════════════════════
网站：[域名]
目标：[用户要求做的事情]
═══════════════════════════════════════════════

[阶段1] 登录/准备
  1.1 打开登录页面
  1.2 输入凭据 → 提交
  1.3 等待跳转至目标页面

[阶段2] 数据采集
  2.1 定位数据列表
  2.2 提取当前页面数据 → 实时保存
  2.3 翻到下一页 → 重复提取
  2.4 直到所有页面完成

[阶段3] 数据处理
  3.1 去重/清洗
  3.2 转换为目标格式（JSON/CSV）
  3.3 保存最终文件

[阶段4] 验证
  4.1 核对总数
  4.2 抽检数据质量

预计数据量：约 N 条
预计耗时：约 N 分钟
风险点：[验证码/频率限制/登录过期等]
人机验证策略：模拟真人节奏 + 遇到验证码暂停引导用户
```

**输出计划后，让用户确认（或直接开干，根据用户偏好）。**

#### 针对不同页面的扫描重点关注

- **列表页：** 重点看数据量、翻页方式、每页条目数、是否有筛选条件
- **详情页：** 重点看内容结构、是否有多页内容、图片附件
- **表单页：** 重点看字段数量、必填字段、选项类型、校验规则
- **搜索结果页：** 重点看结果总数、分页、排序方式
- **登录页：** 重点看登录方式（账号密码/OAuth/SMS）、是否有验证码

#### 扫描中发现的障碍处理

| 发现 | 处理方式 |
|------|----------|
| 页面需要登录才能访问 | 先引导用户提供凭据 |
| 有验证码 | 停止自动操作，引导用户手动完成验证 |
| Cloudflare 保护 | 用完整 Browser（非无头） |
| 动态加载（JS渲染） | 必须用 Browser，不能用 web_extract |
| 页面结构复杂 | 用区块划分法逐块分析 |
| 某元素用途不明 | 用5种推断方法（周边文本/placeholder/预设值/页面标题/标记确认） |
| 必填字段无内容 | 精确模式→问用户；智能模式→自动补全 |
| 数据量大（1000+条） | 评估是否要用分页+定时任务的方案 |
| 页面有iframe内嵌内容 | 先切换到iframe再分析内部结构 |

---

### 第2步：分析任务

先分析需要做的事情，确定：
- 需要哪些页面/URL
- 同域名任务 → 复用已有页面，不新开
- 是否需要登录
- 是否需要交互（点击、滚动、表单）
- 数据量大小（10条 vs 10000条）
- 是否需要定时重复执行
- 数据保存策略（每步保存还是最后统一保存）

### 第3步：根据复杂度选择工具

#### 扫描中发现的障碍处理

**简单任务（1-2页，纯内容）**：
直接用 `web_extract` 提取后整理结果。无JS、无交互。

**中等任务（多页，需要交互）**：
用 Browser 打开网站，执行交互操作。**单页面原则：**
- 一个域名只打开一个标签页
- 在同一页面内导航（点击链接/按钮），不重复打开新的浏览器
- 如果用户指令涉及同一网站的多个子页面，用导航而非新开

**复杂任务（大量数据，多步骤流水线）**：
用 `execute_code` 编写 Python 脚本：
```python
from hermes_tools import web_extract, web_search, terminal, read_file, write_file
import json, re, time

# 步骤1：收集URL
search_result = web_search(query="site:example.com products")
# 从搜索结果提取URL
urls = [r["url"] for r in search_result.get("data", {}).get("web", [])]

# 步骤2：逐个提取 + 实时保存
all_data = []
for i, url in enumerate(urls[:10]):
    page = web_extract(urls=[url])
    item = parse_page(page)
    all_data.append(item)
    # 实时保存：每步都写文件
    write_file(path="/tmp/progress.json", content=json.dumps(all_data, ensure_ascii=False, indent=2))
    print(f"[{i+1}/{len(urls[:10])}] Saved {item.get('title', url)}")
    time.sleep(1.5)  # 礼貌延迟，防封
```

### 第4步：数据处理

提取原始数据后，通常需要：
- 清洗（去重、格式化）
- 结构化（JSON/CSV）
- 分析（统计、排序）
- 保存最终文件

### 第5步：验证

- 检查数据完整性（数量、关键字段）
- 检查数据一致性（格式、编码）
- 输出样本数据供用户确认

---

## 防人机验证策略

这是最关键的部分。人机验证通常由以下行为触发：

### 触发原因及对策

| 触发原因 | 对策 |
|----------|------|
| 请求频率过高（<1秒内连续请求） | 每次操作后等待 1-3 秒 |
| 鼠标轨迹不自然（瞬间到达） | 使用 Browser 模拟真实移动 |
| 无头浏览器特征 | 使用完整 Browser |
| 登录后立即大量操作 | 登录后等待 2-3 秒再操作 |
| 短时间内大量翻页 | 每翻 5 页暂停 5 秒 |
| 填表速度过快 | 字段间隔 0.5-1 秒 |
| 多次错误输入 | 检查表单结构后再输入 |

### 反检测清单

- [ ] 每次页面操作后等待 1-2 秒
- [ ] 表单字段逐字段填写，不批量粘贴
- [ ] 翻页间隔至少 2 秒
- [ ] 连续操作 20 步后暂停 5-10 秒
- [ ] 不使用固定间隔（在 1-3 秒之间随机）
- [ ] 避免在深夜/非正常时段大量操作
- [ ] 遇到验证码立即停止，引导用户手动处理

---

## 复杂控件处理指南

这是核心章节。网页表单的复杂控件（下拉菜单、级联选择、富文本编辑器、自定义组件等）是最容易卡住自动化流程的地方。以下是针对每一种控件的具体处理方案。

---

### 控件优先级策略

遇到填写任务时，按以下优先级尝试：

```
1级（最快）：URL参数预填 → 查看表单action URL是否有GET参数可用
2级（直接）：原生<input>/<select> DOM操作 → selectElement.value = 'x'
3级（可靠）：发出键盘事件 → 模拟用户逐项输入
4级（兜底）：模拟点击 → 点击元素、等待弹出、选择选项
5级（最慢）：可视化操作 → 通过屏幕坐标点击
```

**始终从1级开始**，4级和5级只在前面都失败时使用。

---

### 1. 标准 `<select>` 下拉菜单

这是最简单的下拉控件，直接通过DOM操作。

**单选框（单选）：**
```javascript
// 方法A：直接设value（最快）
document.querySelector('select[name="category"]').value = '3';
document.querySelector('select[name="category"]').dispatchEvent(new Event('change', { bubbles: true }));

// 方法B：按显示文本选择
const select = document.querySelector('select[name="category"]');
for (const opt of select.options) {
  if (opt.text.includes('电子产品')) {
    select.value = opt.value;
    break;
  }
}
select.dispatchEvent(new Event('change', { bubbles: true }));
```

**多选框（multiple）：**
```javascript
const select = document.querySelector('select[name="tags"]');
// 清空所有选中
for (const opt of select.options) opt.selected = false;
// 选中指定选项
const values = ['tech', 'ai', 'python'];
for (const opt of select.options) {
  if (values.includes(opt.value)) opt.selected = true;
}
select.dispatchEvent(new Event('change', { bubbles: true }));
```

---

### 2. 自定义下拉菜单（非标准 `<select>`）

很多现代UI框架（Ant Design、Element UI、Tailwind等）使用 `div`/`ul`/自定义组件来模拟下拉菜单。特征：没有 `<select>` 标签。

**识别方法：** 查看目标元素标签，如果是 `<div>`、`<span>`、`<button>` 且有下拉触发器（点击显示列表），则是自定义下拉。

**操作方法：**

```
第1步：点击触发器让选项列表展开
第2步：等待选项列表出现（0.5-1秒）
第3步：找到目标选项并点击
第4步：等待列表收起
```

```javascript
// 点击触发器展开下拉
document.querySelector('.ant-select-selector').click();
await new Promise(r => setTimeout(r, 500));

// 找到选项列表中的目标
const allOptions = document.querySelectorAll('.ant-select-item-option');
for (const opt of allOptions) {
  if (opt.textContent.includes('电子产品')) {
    opt.click();
    break;
  }
}
await new Promise(r => setTimeout(r, 500));
```

**常用UI框架的selector参考：**

| 框架 | 触发器 | 选项面板 | 选项项 |
|------|--------|----------|--------|
| Ant Design | `.ant-select-selector` | `.ant-select-dropdown` | `.ant-select-item-option` |
| Element UI | `.el-select .el-input` | `.el-select-dropdown` | `.el-select-dropdown__item` |
| Element Plus | `.el-select-v2` | `.el-select-dropdown` | `.el-select-dropdown__item` |
| Ant Design Vue | `.ant-select-selector` | `.ant-select-dropdown` | `.ant-select-item-option` |
| Naive UI | `.n-base-selection` | `.n-select-menu` | `.n-select-option` |
| React Select | `.css-...-control` | `.css-...-menu` | `.css-...-option` |
| Vuetify | `.v-select` | `.v-menu__content` | `.v-list-item` |
| Chakra UI | `.chakra-select__wrapper` | `.chakra-select__menu` | `[role="option"]` |
| MUI (Material) | `.MuiSelect-select` | `.MuiPaper-root` | `.MuiMenuItem-root` |
| Headless UI | `[role="combobox"]` | `[role="listbox"]` | `[role="option"]` |
| 原生 Select | `select[name="x"]` | — | `option[value="x"]` |

---

### 3. 级联选择器（Cascader）

多级联动下拉，如：省→市→区、分类→子分类→子子分类。

**操作流程：**

```
第1步：点击级联触发器打开第一级
第2步：点击第一级选项 → 自动展开第二级
第3步：点击第二级选项 → 自动展开第三级
第4步：点击最后一级选项 → 完成选择
每级之间等待 0.5-1 秒
```

**Ant Design Cascader：**
```javascript
// 打开级联
document.querySelector('.ant-cascader-picker').click();
await new Promise(r => setTimeout(r, 600));

// 选省
const level1 = document.querySelectorAll('.ant-cascader-menu-item');
for (const item of level1) {
  if (item.textContent.includes('广东省')) { item.click(); break; }
}
await new Promise(r => setTimeout(r, 600));

// 选市
const level2 = document.querySelectorAll('.ant-cascader-menu-item');
for (const item of level2) {
  if (item.textContent.includes('深圳市')) { item.click(); break; }
}
await new Promise(r => setTimeout(r, 600));

// 选区（最后一级）
const level3 = document.querySelectorAll('.ant-cascader-menu-item');
for (const item of level3) {
  if (item.textContent.includes('南山区')) { item.click(); break; }
}
```

**Element UI Cascader：**
```javascript
document.querySelector('.el-cascader .el-input').click();
// 选项在 .el-cascader-menu__item 内
```

**级联选完后的验证：** 检查级联显示区域文本是否包含了所有选择的层级。

---

### 4. 自动补全/搜索下拉（AutoComplete, Select with Search）

这种控件需要输入关键词后等搜索结果出现再选择。

```
第1步：点击输入框（focus）
第2步：输入搜索关键词（逐字输入，间隔100-200ms）
第3步：等待搜索结果显示（1-2秒）
第4步：从结果列表中选择目标项
第5步：等待下拉关闭
```

```javascript
// 聚焦并输入
const input = document.querySelector('.ant-select-selection-search-input');
input.focus();
input.value = '电子';
input.dispatchEvent(new Event('input', { bubbles: true }));
await new Promise(r => setTimeout(r, 1500));

// 等待搜索结果出现，选择
const options = document.querySelectorAll('.ant-select-item-option');
for (const opt of options) {
  if (opt.textContent.includes('电子产品')) {
    opt.click();
    break;
  }
}
```

---

### 5. 日期选择器

有三种常见形式，按优先级操作：

| 类型 | 特征 | 最佳操作 |
|------|------|----------|
| 文本输入型 | `<input type="date">` 或 `<input>` 可直接输入 | 直接设value |
| 弹窗点选型 | 点击后弹出日历面板 | 直接输文本或用DOM点击 |
| 双日历范围型 | 选择开始+结束日期 | 先点开始日，再点结束日 |

**文本输入型（最简单）：**
```javascript
// 直接设值
const dateInput = document.querySelector('input[type="date"]');
dateInput.value = '2025-06-15';
dateInput.dispatchEvent(new Event('input', { bubbles: true }));
dateInput.dispatchEvent(new Event('change', { bubbles: true }));

// 或者用 flatpickr / laydate 的直接输
document.querySelector('#date-input').value = '2025-06-15';
```

**弹窗点选型（通用技巧）：**
```javascript
// 方法A：先用键盘输入（很多日期选择器支持）
const input = document.querySelector('.ant-picker input');
input.focus();
input.value = '2025-06-15';
input.dispatchEvent(new Event('input', { bubbles: true }));
// 确认输入
const enterEvent = new KeyboardEvent('keydown', { key: 'Enter', bubbles: true });
input.dispatchEvent(enterEvent);

// 方法B：方法A失败后再用日历点选
// 1. 点击打开日历
document.querySelector('.ant-picker').click();
await new Promise(r => setTimeout(r, 400));
// 2. 定位到目标年月，循环翻到目标年月
while (!document.querySelector('.ant-picker-header-view').textContent.includes('2025年6月')) {
  document.querySelector('.ant-picker-next-icon')?.click(); // 或 prev-icon
  // 或 document.querySelector('.ant-picker-header-super-next-btn')?.click(); // 年
  await new Promise(r => setTimeout(r, 300));
}
// 3. 点击目标日期
const cells = document.querySelectorAll('.ant-picker-cell');
for (const cell of cells) {
  if (cell.textContent.trim() === '15' && !cell.classList.contains('ant-picker-cell-disabled')) {
    cell.click(); break;
  }
}
```

**日期范围选择器：**
```javascript
// 先点开始日期
document.querySelectorAll('.ant-picker')[0].click();
// 选择开始日期...
// 再点结束日期（通常弹出第二个月面板）
// 选择结束日期...
```

---

### 6. 复选框组 / 多选组（Checkbox Group）

```javascript
// 批量选中多个选项
const values = ['option1', 'option2', 'option3'];
const checkboxes = document.querySelectorAll('input[type="checkbox"]');
checkboxes.forEach(cb => {
  if (values.includes(cb.value)) {
    cb.checked = true;
    cb.dispatchEvent(new Event('change', { bubbles: true }));
  }
});
```

**自定义复选框（非原生input）：**
```javascript
// 常见于UI框架：点击 div/span 来切换选中状态
const items = document.querySelectorAll('.el-checkbox, .ant-checkbox-wrapper');
items.forEach(item => {
  if (item.textContent.includes('选项A')) {
    // 检查是否已选中
    const isChecked = item.querySelector('.ant-checkbox-checked, .is-checked');
    if (!isChecked) item.click();
  }
});
```

---

### 7. 单选按钮组（Radio Group）

```javascript
// 原生：直接设checked
document.querySelector('input[name="gender"][value="male"]').checked = true;
document.querySelector('input[name="gender"][value="male"]')
  .dispatchEvent(new Event('change', { bubbles: true }));

// 自定义（UI框架）：找到对应项并点击
document.querySelectorAll('.ant-radio-wrapper').forEach(r => {
  if (r.textContent.includes('男')) r.click();
});
```

---

### 8. 树形选择器（TreeSelect / Tree）

用于选择分类、组织架构等层级数据。

```
第1步：点击展开树选择器
第2步：依次展开父节点（展开到目标所在的层级）
第3步：选中目标节点
```

**Ant Design TreeSelect：**
```javascript
// 点击展开
document.querySelector('.ant-select-tree').click();
await new Promise(r => setTimeout(r, 500));

// 展开节点（点击展开箭头）
const expandBtns = document.querySelectorAll('.ant-select-tree-switcher');
expandBtns.forEach(btn => {
  if (btn.classList.contains('ant-select-tree-switcher_close')) btn.click();
});
await new Promise(r => setTimeout(r, 500));

// 选择目标
const nodes = document.querySelectorAll('.ant-select-tree-title');
for (const node of nodes) {
  if (node.textContent.includes('目标分类')) {
    node.closest('.ant-select-tree-treenode')?.click();
    break;
  }
}
```

---

### 9. 标签输入框（Tags / Tag Input）

用于输入多个标签、关键词。

```javascript
// 输入标签后按 Enter 或 逗号
const input = document.querySelector('.ant-tag-input, .el-tag__input input');
const tags = ['标签1', '标签2', '标签3'];

for (const tag of tags) {
  input.focus();
  input.value = tag;
  input.dispatchEvent(new Event('input', { bubbles: true }));
  await new Promise(r => setTimeout(r, 200));
  // 按 Enter 确认
  const enter = new KeyboardEvent('keydown', { key: 'Enter', keyCode: 13, bubbles: true });
  input.dispatchEvent(enter);
  await new Promise(r => setTimeout(r, 300));
}
```

---

### 10. 富文本编辑器（Rich Text / WYSIWYG）

如 TinyMCE、CKEditor、Quill、ProseMirror、Tiptap 等。

**通用方案：** 找到编辑器内容区，直接设 innerHTML：
```javascript
// Quill / Tiptap / ProseMirror
document.querySelector('.ql-editor, .ProseMirror, [contenteditable="true"]').innerHTML = '<p>要输入的内容</p>';

// TinyMCE
tinymce.get('editor_id').setContent('<p>要输入的内容</p>');

// CKEditor
CKEDITOR.instances.editor1.setData('<p>要输入的内容</p>');
```

---

### 11. 滑块 (Slider)

```javascript
// 原生 range
const slider = document.querySelector('input[type="range"]');
slider.value = '75';  // 0-100
slider.dispatchEvent(new Event('input', { bubbles: true }));
slider.dispatchEvent(new Event('change', { bubbles: true }));

// Ant Design Slider：直接输入或拖动
// 优先看是否有数字输入框伴生
const input = document.querySelector('.ant-slider input, .ant-input-number-input');
if (input) {
  input.value = '75';
  input.dispatchEvent(new Event('input', { bubbles: true }));
}
```

---

### 12. 拖拽排序 / 拖拽上传

```javascript
// 拖拽上传：找到 drop zone，直接设 files
const dropzone = document.querySelector('.ant-upload-drag, .el-upload-dragger');
const input = dropzone.querySelector('input[type="file"]');
if (input) {
  const fileList = new DataTransfer();
  fileList.items.add(new File([''], 'file.pdf'));
  input.files = fileList.files;
  input.dispatchEvent(new Event('change', { bubbles: true }));
}
```

---

### 13. 分页标签 / Tab 切换

```javascript
// 切换到特定Tab
const tabs = document.querySelectorAll('.ant-tabs-tab, .el-tabs__item, [role="tab"]');
for (const tab of tabs) {
  if (tab.textContent.includes('目标Tab名称')) {
    tab.click();
    break;
  }
}
await new Promise(r => setTimeout(r, 1000)); // 等待内容加载
```

---

### 14. 弹窗 / 模态框 / 确认对话框

```javascript
// 关闭弹窗
document.querySelector('.ant-modal-close, .el-dialog__close, .close-button')?.click();

// 点击确认按钮
document.querySelector('.ant-modal-confirm-btns .ant-btn-primary, .el-message-box__btns .el-button--primary')?.click();

// 点击取消
document.querySelector('.ant-modal-confirm-btns .ant-btn:not(.ant-btn-primary)')?.click();
```

---

### 15. 文件上传

```javascript
// 方法A：直接找到 file input 设文件
const fileInput = document.querySelector('input[type="file"]');
const file = new File([''], '/path/to/document.pdf', { type: 'application/pdf' });
const dt = new DataTransfer();
dt.items.add(file);
fileInput.files = dt.files;
fileInput.dispatchEvent(new Event('change', { bubbles: true }));

// 方法B：多文件上传
const dt = new DataTransfer();
['file1.pdf', 'file2.pdf', 'file3.pdf'].forEach(name => {
  dt.items.add(new File([''], name));
});
fileInput.files = dt.files;
fileInput.dispatchEvent(new Event('change', { bubbles: true }));
```

---

### 16. 验证码 / 人机验证弹窗

```javascript
// 停止自动化，通知用户
// 不要尝试自动破解 reCAPTCHA/hCaptcha
// 引导用户手动完成验证
```

---

### 诊断工具：分析页面上的控件

当不确定页面用了什么UI框架时，在 Browser 中执行：

```javascript
// 检测常见的UI框架
const frameworks = {
  antd: !!document.querySelector('.ant-select, .ant-btn, .ant-table'),
  element: !!document.querySelector('.el-select, .el-button, .el-table'),
  elementPlus: !!document.querySelector('.el-select-v2, .el-button'),
  vuetify: !!document.querySelector('.v-select, .v-btn'),
  mui: !!document.querySelector('.MuiSelect-root, .MuiButton-root'),
  chakra: !!document.querySelector('.chakra-select, .chakra-button'),
  naive: !!document.querySelector('.n-select, .n-button'),
  headless: !!document.querySelector('[role="combobox"], [role="listbox"]'),
  bootstrap: !!document.querySelector('.form-select, .btn, .dropdown-menu'),
};
console.log('Detected UI frameworks:', Object.entries(frameworks).filter(([k,v]) => v).map(([k]) => k));

// 列出所有表单元素
document.querySelectorAll('input, select, textarea, [role="combobox"], [role="textbox"], [contenteditable="true"]')
  .forEach(el => {
    console.log({
      tag: el.tagName,
      type: el.type || el.getAttribute('role'),
      name: el.name || el.id,
      required: el.required || el.getAttribute('aria-required'),
      placeholder: el.placeholder,
    });
  });
```

---

## 填表总流程

```
1. 分析表单结构 → 用诊断工具检测UI框架，列出所有字段
2. 从1级方案开始尝试 → URL参数预填
3. 逐个填写字段 → 按控件类型选择最优处理方式
4. 提交前检查 → 验证所有必填字段已正确填写
5. 提交 → 点击提交按钮
6. 等待结果 → 等待跳转/成功提示出现
7. 结果检查 → 验证提交成功，保存提交结果/凭证
```

---

## 实时保存策略

**为什么需要实时保存：** 网页抓取可能因为网络中断、验证码、IP封禁、页面结构变化等原因中断。每步保存确保已获取数据不丢失。

### 保存策略

```
每获取N条数据 → 写入临时文件 → 继续下一批
                ↓
              中断后从临时文件恢复，跳过已获取的
```

### 实现模式

```python
import json, os, time
from hermes_tools import write_file

PROGRESS_FILE = "/tmp/scrape_progress.json"
RESULT_FILE = "/tmp/scrape_final.json"

def save_progress(data):
    """实时保存进度，覆盖写入"""
    write_file(path=PROGRESS_FILE, content=json.dumps(data, ensure_ascii=False, indent=2))

def load_progress():
    """恢复已保存的进度"""
    if os.path.exists(PROGRESS_FILE):
        with open(PROGRESS_FILE) as f:
            return json.load(f)
    return []

def process_items(urls):
    all_data = load_progress()  # 恢复
    seen_urls = {item.get("url") for item in all_data if item.get("url")}
    
    for url in urls:
        if url in seen_urls:
            continue  # 跳过已处理的
        # ... 处理逻辑 ...
        all_data.append({"url": url, "title": title, "content": content})
        save_progress(all_data)  # 每步保存
        print(f"✓ Saved: {title}")
        time.sleep(1.5)
    
    # 最终保存
    write_file(path=RESULT_FILE, content=json.dumps(all_data, ensure_ascii=False, indent=2))
    return all_data
```

### 保存时机

| 时机 | 动作 |
|------|------|
| 每获取1条数据 | 追加到内存列表 + 写入临时文件 |
| 每完成10页 | 写一次中间结果快照 |
| 发生错误时（except块） | 立即保存当前所有数据 |
| 完成时 | 写出最终结果文件 |
| 用户中断时 | 引导用户恢复进度 |

---

## 单实例页面管理

**核心规则：同一域名只开一个页面，不重复多开。**

### 页面生命周期

```
首次访问某个域名：
  打开浏览器 → 访问目标URL → 标记该域名"已打开"

后续操作同一域名：
  检查该域名是否已有一个页面 → 有则直接在当前页面导航 → 无则新开

退出时：
  不要主动关闭浏览器，让用户决定
```

### 具体实现

当使用 Browser 时：

1. **首次打开：**
   - 记录打开的域名和URL
   - 说清楚目前打开了哪个页面

2. **后续操作：**
   - 检查是否已打开同一域名的页面
   - 如果是，在当前页面导航到新URL（不是新开标签页）
   - 如果域名不同，开新标签页
   - 如果需要跨域名操作，先完成当前任务再开新页面

3. **导航代替多开：**
   - 需要查看列表页 → 详情页 → 返回列表 → 另一个详情页
   - 正确做法：列表页点击 → 详情页查看 → 返回 → 再点下一个
   - 错误做法：每次打开新标签页

4. **多个任务冲突处理：**
   - 如果用户在中途要求做完全不同域名的事
   - 先保存当前页面进度
   - 切换到新任务
   - 完成后恢复之前的页面（如必要）

---

## 媒体内容生成与自动上传

用户可能要求 Agent 先生成图片/视频等媒体内容，再自动上传到目标网站。这个流程需要：生成 → 下载到本地 → 通过网页表单上传。

---

### 整体流程

```
用户提出需求（如"生成一张产品图上传到XX网站"）
       ↓
1. 分析上传目标页面 → 扫描上传表单结构
       ↓
2. 选择媒体生成工具 → 生成内容
       ↓
3. 下载/保存生成的文件到本地
       ↓
4. 打开上传目标网站 → 填写表单 → 上传文件
       ↓
5. 验证上传结果 → 确认成功
```

---

### 第1步：分析上传目标

用 Browser 打开目标网站，扫描上传界面：

```
1. 上传方式：
   - 文件选择器：<input type="file"> （最常见）
   - 拖拽区域：有 drag & drop 区域
   - 粘贴上传：支持 Ctrl+V / 剪贴板粘贴
   - API上传：有上传接口可直传

2. 上传限制：
   - 文件格式限制（accept="image/png, .jpg"）
   - 文件大小限制
   - 尺寸/分辨率要求
   - 是否要求压缩/裁剪

3. 附带字段（如标题、标签、描述等）：
   - 需要一并填写的文本字段
```

---

### 第2步：选择媒体生成工具

根据用户需求选择生成方式：

| 需求 | 工具/技能 |
|------|----------|
| 生成图片 | ComfyUI, fal-ai, MiniMax (mmx-cli) |
| 生成视频 | ComfyUI (视频模型), fal-ai video, MiniMax video |
| 本地上已有文件 | 直接用现有文件 |
| 从网页下载素材 | Browser / web_extract 获取下载链接 |

**图片识别（获取图片信息）：**

当用户提供图片链接或本地图片时，需要先识别图片内容、提取文字、分析场景。使用 **豆包视觉理解模型（KPI — 图片理解）** 进行图片识别：

```python
from hermes_tools import terminal, vision_analyze, write_file

# 方式A：用 vision_analyze 工具（内置支持）直接识别
# 传入图片URL或本地路径，返回图片描述
result = vision_analyze(
    image_url="https://example.com/product_image.jpg",
    question="请详细描述这张图片的内容、主体、颜色、文字信息"
)
print(result)

# 方式B：通过豆包API调用（需配置豆包API key）
# 配置方法：在 hermes config 中设置豆包 visual provider
# hermes config set visual_provider doubao
# 之后 vision_analyze 会自动使用豆包视觉模型

# 方式C：用终端调用豆包视觉API
result = terminal(command="""
curl -X POST https://ark.cn-beijing.volces.com/api/v3/chat/completions \\
  -H "Authorization: Bearer $DOUBAO_API_KEY" \\
  -H "Content-Type: application/json" \\
  -d '{
    "model": "doubao-vision-pro-32k",
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "text", "text": "详细描述这张图片的内容"},
          {"type": "image_url", "image_url": {"url": "https://example.com/photo.jpg"}}
        ]
      }
    ]
  }'
""")
```

**图片识别后可以做什么：**
- 提取图片中的文字（如海报文字、商品标签、截图内容）
- 识别图片主体和场景（判断是什么商品、什么场景）
- 分析图片质量（清晰度、构图、是否符合上传要求）
- 根据识别结果自动填充相关表单字段（如"这是一张红色跑鞋照片"→ 自动帮填颜色=红色、分类=运动鞋）

**生成并保存文件：**
```python
from hermes_tools import terminal, write_file
import json

# 示例：用 ComfyUI 生成图片（根据实际情况调整）
result = terminal(
    command="comfy run --workflow product-photo.json --prompt '红色跑鞋侧面图'",
    timeout=120
)
# 输出中获取生成的文件路径
generated_path = "/tmp/comfyui_output/product_shoe.png"

# 或者用 fal-ai
# terminal("fal-ai comfy --workflow-url https://fal.ai/models/... --prompt '...'")

# 检查文件是否存在
check = terminal(command=f"ls -lh {generated_path}")
```

**fal-ai 方式（已安装 fal-ai 时）：**
```python
from hermes_tools import terminal
# 直接调用 fal-ai 生成
result = terminal(
    command="python3 -c \"
import fal_client
result = fal_client.run('fal-ai/flux/dev', arguments={'prompt': '产品图描述'})
print(result['images'][0]['url'])
\"",
    timeout=120
)
# 下载到本地
terminal(command=f"curl -o /tmp/output.png '{result}'")
```

---

### 第3步：将文件上传到目标网站

#### 方式A：通过 Browser 模拟文件上传（最通用）

```javascript
// 1. 直接设 file input 的 files 属性
const fileInput = document.querySelector('input[type="file"]');
if (!fileInput) {
  // 可能是拖拽区域，找隐藏的 file input
  fileInput = document.querySelector('.upload-area input[type="file"], [class*="upload"] input[type="file"]');
}

// 创建 File 对象
const response = await fetch('file:///tmp/generated_image.png');
const blob = await response.blob();
const file = new File([blob], 'product_photo.png', { type: 'image/png' });

// 设值
const dt = new DataTransfer();
dt.items.add(file);
fileInput.files = dt.files;
fileInput.dispatchEvent(new Event('change', { bubbles: true }));

// 等待上传完成
await new Promise(r => setTimeout(r, 3000));
```

**对于拖拽上传区域：**
```javascript
// 找到拖拽区域，触发 drag 事件
const dropzone = document.querySelector('.ant-upload-drag, .el-upload-dragger, [class*="dropzone"]');
const fileInput = dropzone.querySelector('input[type="file"]');
// 大部分拖拽上传底层也是 file input
// 直接设 file input 的 files 即可触发上传
```

#### 方式B：通过拼多多/电商平台上传API

如果目标网站有公开的上传API，可以直传：

```python
from hermes_tools import terminal, web_extract
import json

# 先获取上传token/签名
upload_config = web_extract(urls=["https://目标网站.com/api/upload/token"])

# 用 curl 上传
result = terminal(
    command=f'curl -X POST -F "file=@/tmp/generated_image.png" -F "token={token}" https://目标网站.com/api/upload'
)
```

#### 方式C：批量上传多个文件

```javascript
// 多文件上传
const fileInput = document.querySelector('input[type="file"][multiple]');
const dt = new DataTransfer();
['image1.png', 'image2.png', 'image3.png'].forEach(name => {
  // 实际使用时从本地路径读取
  dt.items.add(new File(['mock'], name, { type: 'image/png' }));
});
fileInput.files = dt.files;
fileInput.dispatchEvent(new Event('change', { bubbles: true }));
```

---

### 完整示例：生成图片并上传到商品编辑页

场景：用户说"生成一张红色跑鞋的产品图，上传到店铺商品编辑页"

```
$ 步骤1：扫描上传页面
   打开 https://seller.example.com/product/123/edit
   → 发现：
     - 图片上传区：拖拽区域 + input[type="file"][multiple]（最多5张）
     - 附带字段：标题、价格、描述
     - 限制：仅支持 JPG/PNG，单张不超过5MB
     - UI框架：Ant Design（.ant-upload-drag）

$ 步骤2：生成图片
   用 ComfyUI 生成 "红色跑鞋 侧面45度 白色背景 商业摄影风格"

$ 步骤3：下载到本地
   已生成到 /tmp/comfyui_output/red_shoe.png

$ 步骤4：上传到商品页
   打开上传页面 → 设 file input → 填写标题字段 → 提交
```

**执行代码：**
```python
from hermes_tools import terminal, write_file

# 1. 检查生成的图片
check = terminal("ls -lh /tmp/comfyui_output/red_shoe.png")
print(f"图片已生成: {check['output']}")

# 2. 记录上传信息供 Browser 使用
info = {
    "file_path": "/tmp/comfyui_output/red_shoe.png",
    "file_name": "red_shoe_product.png",
    "file_type": "image/png",
    "target_url": "https://seller.example.com/product/123/edit",
    "upload_selector": "input[type='file']",
    "additional_fields": {
        "title": "2025新款红色跑鞋",
        "price": "299"
    }
}
write_file("/tmp/upload_info.json", json.dumps(info, indent=2))

print(f"上传信息已保存到 /tmp/upload_info.json")
print(f"下一步：用 Browser 打开目标页面，用 file input 上传")
```

---

### 常见陷阱

| 陷阱 | 解决方案 |
|------|----------|
| Browser 中文件路径不是 file:// 协议 | 用终端先拷贝到可访问路径 |
| file input 设了 files 但不上传 | 确保 dispatchEvent('change') |
| 图片尺寸/格式不符合上传限制 | 生成时指定参数，或用 ImageMagick 转换 |
| 上传前需要先登录 | 先完成登录流程再上传 |
| 拖拽区域没有直连 file input | 拖拽区域可能用 XMLHttpRequest 直传，需要触发正确事件 |
| 上传进度不可见 | 等待页面出现"上传成功"提示 |
| 上传后需要裁剪/编辑 | 等裁剪完成再继续后续填写 |
| 批量上传有数量限制 | 分批上传，每次不超过限制 |

---

## 严格围绕用户要求填写信息

这是最关键的执行纪律。**根据用户指令的明确程度，采用不同的填充策略。**

---

### 三种填充模式

| 模式 | 用户语气 | 行为 |
|------|---------|------|
| **A：精确模式** | 用户给出具体清单，或说"只填这些" | 只填用户指定的，其余留空/不动 |
| **B：智能补全模式（默认）** | 用户说"帮我填一下"、"发布商品"等同义表达 | 按用户指示填指定字段 + 自动补全合理的辅助信息 |
| **C：完全委托模式** | 用户说"帮我搞定"、"你看着办" | 全权代理，自行完善所有必要信息 |

**判断方法：** 看用户有没有给出**具体且完整的字段列表**。如果用户一一列举了所有要填的字段 → 模式A。如果用户只说了部分信息或模糊描述 → 模式B或C。

---

### 模式A：精确模式 — 只填用户说的

当用户给出**具体清单**或明确说**"只填这些"**时：

```
用户："标题写'2025新款'，价格299，颜色选红色，其他不用填"

操作：
  ✓ 标题：2025新款
  ✓ 价格：299
  ✓ 颜色：红色
  ✗ 不动：描述、库存、标签、分类等其他所有字段
```

**适用场景：**
- 用户列出了1-2-3-4的具体清单
- 用户说"只要填xx和xx"
- 用户说"其他不用管"
- 用户说"只填这些就行"

---

### 模式B：智能补全模式（默认） — 填用户说的 + 自动补全合理的辅助信息

当用户**没有明确说"只填"或"不填"**时，自动帮用户完善页面信息。

**核心逻辑：**

```
用户说了什么 → 精确填入这些字段
     +
用户没说什么 → 但页面有这些字段时，自动补全合理的内容
     +
用户说的信息可以推导出什么 → 自动推导填入
```

**何时自动补全：**
- 用户说"发布一个商品" → 自动填：库存(默认999)、运费(包邮)、上架时间(立即)
- 用户说"上传这张图片" → 自动填：图片标题(文件名)、ALT文本(商品名)
- 用户说"填一下商品信息" → 自动填：描述(根据标题/价格/类型自动生成简介)、标签(根据分类推导)

**何时不自动补全（需确认）：**
- 涉及金钱/定价的敏感字段（折扣、优惠券、活动价）
- 涉及法律/合规的字段（认证、资质、声明）
- 涉及用户隐私的字段（联系方式、地址）
- 有明显正确/错误选择的字段（选错了会出问题）

**补全内容的依据：**
- 从用户已提供的信息推导（如用户说了商品类型，推导出合理分类）
- 使用通用默认值（库存=999，运费=包邮）
- 使用合理的合理推断（电动牙刷 → 分类选"个人护理"）
- 使用上下文信息（用户在编辑已有商品 → 保留原有值）

---

### 补全内容的话术策略

这是填充质量和用户体验的关键。**所有自动生成的内容必须模拟人类的自然表达，而不是机器人式的生硬文本。** 同时根据不同的场合（平台类型、商品性质、用户风格）调整话术风格。

---

#### 核心原则

```
不写机械感文本
  → 不说"本产品是一款..."、"此商品适用于..."
  → 说"这款真的绝了"、"闭眼入"

不写SEO堆砌
  → 不说"高品质优质好物性价比超高..."
  → 说人话，有真实的推荐感

根据不同场合切换语气
  → 不同的平台、商品、用户习惯 → 不同的话术风格
```

---

#### 场合分类与对应话术

| 场合 | 语气特征 | 描述风格 | 示例 |
|------|---------|---------|------|
| **拼多多/低价电商** | 亲切、接地气、有促销感 | 简短有力、突出优惠、用感叹号和emoji | "家人们冲！限时特价！🔥" |
| **淘宝/综合电商** | 专业、详细、有信任感 | 突出材质/功能/售后，适当用"亲" | "亲，这款面料超舒服，透气不闷热~" |
| **京东/3C数码** | 参数清晰、理性、专业 | 突出规格、性能、质保 | "搭载最新芯片，性能提升30%，享官方质保" |
| **小红书/种草平台** | 朋友推荐感、真实分享 | 第一人称、带个人体验、自然语气 | "用了半个月真的回不去了，姐妹们听劝！" |
| **抖音/快手电商** | 快节奏、有冲击力 | 短句、痛点切入、促单话术 | "还在用普通款？试试这个，差距太大了" |
| **闲鱼/二手平台** | 真实个人卖家、随意 | 简单描述、说明成色、理由自然 | "买来就用过一次，跟新的一样，放着落灰了" |
| **知乎/专业社区** | 理性分析、有深度 | 逻辑清晰、引用数据或经验 | "从参数来看这款在同价位确实能打" |
| **微信公众号/微商** | 有温度、故事感 | 软文风格、情感共鸣 | "第一次拿到手就被质感惊艳到了" |

#### 不同商品类型的话术侧重

| 商品类型 | 侧重点 | 示例话术 |
|---------|-------|---------|
| **服装/穿搭** | 版型、材质、搭配、显瘦 | "版型超正！微胖姐妹也能穿" |
| **数码/3C** | 性能参数、功能、适用场景 | "续航一整天，出差不用带充电器" |
| **美妆/护肤** | 肤质、效果、成分 | "油皮亲妈！上脸瞬间哑光" |
| **食品/零食** | 口感、口味、食材 | "一口下去酥掉渣，追剧必备" |
| **家居/日用** | 实用性、颜值、质量 | "租房党必备，瞬间提升幸福感" |
| **母婴/儿童** | 安全、材质、舒适 | "A类纯棉，宝宝穿超安心" |
| **虚拟/服务** | 便捷、省时、效果 | "码住！每天10分钟轻松搞定" |

---

#### 不同用户习惯的话术匹配

如果用户历史中自己写过商品描述，可以在话语中参考用户自己的表达方式：

```
用户过往写过的： "质量很好，推荐购买"
  → Agent补全时用： 简洁、务实话术
  → 不写："姐妹们冲啊"（不符合用户风格）

用户过往写过的： "超级好用！姐妹们一定要入！"
  → Agent补全时用： 热情、种草风话术
  → 不写："质量可靠，值得购买"（太正经）

用户是第一次用，无历史：
  → 根据平台类型选择默认语气
```

---

#### 话术范例表（按场合）

**拼多多 / 低价电商：**
```
"家人们冲！🔥 限时秒杀价到手只要XX，错过等一年！"
"这个价格真的杀疯了，赶紧下手🛒"
"质量杠杠的，买过的都说好👍"
"真·白菜价，囤就完事了"
```

**淘宝 / 综合电商：**
```
"亲～这款XX真的绝了，面料超柔软，夏天穿巨凉爽"
"现货秒发，喜欢的亲不要错过哦～"
"性价比超高，质量完全对得起这个价格"
"好评如潮，回头客超多，闭眼入不会错"
```

**京东 / 3C数码 / 品牌店：**
```
"搭载XX处理器，性能释放充分，日常使用/游戏体验流畅"
"官方正品，享全国联保，品质有保障"
"X英寸2K屏，观影办公都很舒适"
"续航实测X小时，彻底告别电量焦虑"
```

**小红书 / 种草风格：**
```
"用了半个月真的回不去了！！姐妹们一定要试试"
"听劝！这个真的可以冲，谁用谁知道"
"被问了800遍的链接来了，真的巨好用"
"小个子福音！这个版型我吹爆"
```

**抖音 / 快手：**
```
"还在用普通款？试试这个，差距不是一点点"
"敢说不好用算我输！拍一发三，赶紧上车"
"懂行的都在抢，再犹豫就没了"
```

**闲鱼：**
```
"买来就用过两次，跟新的一样，闲置回血"
"公司发的用不上，全新未拆封，便宜出了"
"搬家清仓，都是自用正品"
```

---

#### 描述自动生成模板

根据不同场合，自动生成描述时：

```python
def generate_description(product_info, platform="taobao"):
    """根据商品信息和平台生成描述"""
    
    name = product_info.get("title", "新品")
    price = product_info.get("price", "")
    
    if platform == "pinduoduo":
        return f"🔥 {name}，限时特价仅{price}！质量杠杠的，买过的都说好！需要的赶紧下手🛒"
    
    elif platform == "taobao":
        return f"亲～{name}来啦！现货秒发，品质保证。超高性价比，喜欢不要错过哦～"
    
    elif platform == "jingdong":
        return f"{name}，官方正品保障。性能出色，品质可靠，享全国联保。值得入手。"
    
    elif platform == "xiaohongshu":
        return f"被问了N次！{name}真的绝了！！姐妹们相信我，用过就回不去了✨"
    
    elif platform == "xianyu":
        return f"自用{name}，买来用了几次就闲置了，成色很新，便宜回血。"
    
    else:
        return f"{name}，品质之选，值得拥有。"
```

---

### 模式C：完全委托模式 — 全权代理

当用户说"帮我搞定"、"你看着办"、"全部帮我填"时：

```
全权代理，自行完善所有必要信息，包括：
  ✓ 所有文本字段（标题、描述、备注）
  ✓ 所有选项选择（分类、标签、属性）
  ✓ 所有数值字段（价格、库存、重量）
  ✓ 所有配置项（上架时间、运费、促销）
```

**但需要注意：**
- 需要用户权限/密码的字段 → 问用户
- 涉及付款/扣费的设置 → 告知用户
- 不可逆的操作（如删除）→ 提前说明

---

### 判断流程图

```
用户说"帮我做XX"

1. 用户有没有给出具体清单或说"只填"？
   → YES → 模式A：精确模式
   → NO  → 跳到2

2. 用户是模糊表达还是完全委托？
   → "帮我填一下"、"发布个商品"、"填这些信息" → 模式B：智能补全
   → "帮我搞定"、"你看着办"、"全权处理"     → 模式C：完全委托

3. 模式B执行中，遇到不确定的字段怎么办？
   → 如果有合理默认值或可推导 → 自动填
   → 如果涉及金钱/法律/隐私 → 问用户
```

---

### 场景示例

#### 场景1：智能补全（默认模式）

```
用户： "帮我发布一个电动牙刷，标题叫'声波电动牙刷Pro'，价格129"

应填写：
  ✓ 标题：声波电动牙刷Pro（用户指定）
  ✓ 价格：129（用户指定）
  
  自动补全：
  ✓ 库存：999（默认）
  ✓ 分类：个人护理 → 口腔护理（根据商品名推导）
  ✓ 上架时间：立即上架（合理默认）
  ✓ 运费：包邮（常见默认）
  ✓ 描述：自动生成简短的产品简介"声波电动牙刷Pro，高效清洁，呵护口腔健康"
  
  留空（无合理默认）：
  ○ 优惠券（涉及促销策略，不自动设）
  ○ 活动价（涉及定价策略，不自动设）
```

#### 场景2：精确模式

```
用户： "帮我发布一个商品，标题叫'真丝连衣裙M码'，价格198，颜色选白色和黑色，只填这些"

应填写：
  ✓ 标题：真丝连衣裙M码（用户指定）
  ✓ 价格：198（用户指定）
  ✓ 颜色：白色、黑色（用户指定）

  ✗ 不填：库存（用户没说，且明确"只填这些"）
  ✗ 不填：描述（同上）
  ✗ 不填：分类（同上）
  ✗ 不填：运费（同上）
```

#### 场景3：用户修改部分内容（保留原有补全）

```
用户： "改一下价格改成168"

操作：
  ✓ 价格：198 → 168
  ○ 保留：之前自动补全的库存、分类、描述、运费等
  
理由：用户只说要改价格，其他不变。不因为用户改价格就重置其他字段。
```

#### 场景4：完全委托模式

```
用户： "帮我搞定这个商品页面，你看着办"

应填写：
  ✓ 标题：自动生成（如"新品上市" + 合理补充）
  ✓ 价格：自动设定（除非用户给了范围）
  ✓ 描述：自动生成完整描述
  ✓ 分类：自动选择最合适的
  ✓ 库存、运费、标签等：全部自动填
  ✓ 图片：如果有生成图片则上传，没有则留空
  
  但仍需问用户：
  ✗ 涉及金钱的优惠设置
  ✗ 涉及需要上传资质证照的
```

---

### 填写前检查

```
1. 判断当前是哪种模式？依据是什么？
2. 模式A → 严格按照清单，不多填
3. 模式B → 用户说了的精确填，用户没说的看能否合理补全
4. 模式C → 全权代理，但注意敏感字段
```

### 填写后验证

- [ ] 用户明确指定的内容是否已精确填入
- [ ] 自动补全的内容是否合理、不产生误导
- [ ] 敏感字段（金钱/法律/隐私）是否已确认
- [ ] 修改操作时是否保留了其他字段的值

---

## 账号注册流程

这是一个需要谨慎处理的场景。账号注册涉及用户隐私信息，且不同网站的注册规则差异很大。

---

### 核心原则

1. **先问用户：你来还是我来？** — 用户可以选择自己手动注册或让Agent自动操作
2. **询问注册所需信息** — 至少需要：注册邮箱/手机号、用户名、注册目的
3. **注册成功立即告知账号密码** — 保存到文件+告诉用户
4. **严格遵循网站的密码规则** — 长度、特殊字符、大写字母等要求必须满足
5. **密码冲突自动重试** — 如果报错"密码已使用"，自动生成新的重试

---

### 流程总览

```
用户说"帮我注册账号"
       ↓
第1步：询问「你手动注册还是我来自动注册？」
       ↓ (用户选自动)
第2步：询问注册需要的资料
       - 邮箱/手机号（必需）
       - 用户名（如果网站需要）
       - 注册这个账号的用途/目的（用于填资料）
       ↓
第3步：打开注册页面 → 扫描密码规则
第4步：生成符合规则的密码
第5步：填写注册表单
第6步：提交 → 如果报错则处理 → 直到成功
第7步：保存账号密码 → 告知用户
```

---

### 第1步：询问用户操作方式

**话术：**
```
"这个账号你是想自己手动注册，还是我来自动帮你注册？

1. 我自己手动注册（我引导你操作）
2. 你来自动注册（告诉我需要的资料就行）
```

**如果选1（手动）：** 打开注册页面，引导用户填写，用户在关键步骤自己操作，Agent在旁边提示需要填什么。

**如果选2（自动）：** 继续下面的流程。

---

### 第2步：询问注册所需信息

至少需要向用户问清楚：

```
1. 注册邮箱或手机号（哪个用来接收验证？）
2. 用户名（想用什么名？没有的话我随机生成）
3. 这个账号打算用来做什么？（影响后续填写资料）
4. 有没有特殊的密码偏好？（没有就自动生成）
```

**注意：**
- 邮箱/手机号必须用户提供，Agent不能替用户决定
- 用户名可以询问用户偏好，也可以自动生成（如 `user_XXXX`）
- 用途用来填写注册时的必填资料
- 不要问多余的隐私信息

---

### 第3步：扫描注册页面 → 获取密码规则

打开注册页面后，先扫描获取密码规则：

```javascript
// 扫描密码输入框的规则
const pwdInput = document.querySelector('input[type="password"]');

// 方法A：从HTML属性获取
const minLength = pwdInput?.getAttribute('minlength');  // 最小长度
const maxLength = pwdInput?.getAttribute('maxlength');  // 最大长度
const pattern = pwdInput?.getAttribute('pattern');       // 正则要求

// 方法B：从页面文字获取（更可靠）
// 查找密码规则提示，常见的文案：
// "8-16位字符，包含字母和数字"
// "至少8位，包含大写字母、小写字母和数字"
// "密码长度6-20位，不能为纯数字或纯字母"

// 方法C：从placeholder获取
const placeholder = pwdInput?.placeholder;  // 如 "8-16位字母数字组合"

// 输出到控制台供Agent分析
console.log({
  minLength, maxLength, pattern, placeholder,
  pageText: document.body.innerText.match(/密码.{0,50}[:：]?[^。\n]{3,60}/g)
});
```

**需要确认的密码规则：**
- ✅ 最小长度
- ✅ 最大长度
- ✅ 是否需要大写字母
- ✅ 是否需要小写字母
- ✅ 是否需要数字
- ✅ 是否需要特殊字符（!@#$%^&*等）
- ✅ 是否禁止特殊字符
- ✅ 是否不能和用户名相同
- ✅ 是否禁止连续字符（如123、abc）
- ✅ 是否禁止重复字符（如111、aaa）

---

### 第4步：生成符合规则的密码

```python
import random
import string

def generate_password(min_len=8, max_len=20, 
                       require_upper=True, require_lower=True,
                       require_digit=True, require_special=True,
                       forbidden_chars="", similar_to_user="",
                       no_consecutive=True, no_repeat=True):
    """根据密码规则生成符合要求的密码"""
    
    # 确保每种必需字符类型至少有一个
    chars_pool = ""
    password_chars = []
    
    if require_lower:
        chars_pool += string.ascii_lowercase
        password_chars.append(random.choice(string.ascii_lowercase))
    if require_upper:
        chars_pool += string.ascii_uppercase
        password_chars.append(random.choice(string.ascii_uppercase))
    if require_digit:
        chars_pool += string.digits
        password_chars.append(random.choice(string.digits))
    if require_special:
        special = "!@#$%^&*_+-="
        chars_pool += special
        password_chars.append(random.choice(special))
    
    # 排除禁止字符
    for c in forbidden_chars:
        chars_pool = chars_pool.replace(c, "")
    
    # 填充剩余长度
    length = random.randint(min_len, max_len)
    password_chars.extend(random.choice(chars_pool) for _ in range(length - len(password_chars)))
    random.shuffle(password_chars)
    
    password = ''.join(password_chars)
    
    # 检查是否包含连续字符/重复字符（如需要）
    if no_consecutive:
        consecutives = ['123', '234', '345', '456', '567', '678', '789',
                       'abc', 'bcd', 'cde', 'def', 'efg', 'fgh', 'ghi']
        for c in consecutives:
            if c in password.lower():
                return generate_password(min_len, max_len, require_upper, require_lower,
                                       require_digit, require_special, forbidden_chars,
                                       similar_to_user, no_consecutive, no_repeat)
    
    # 检查是否与用户名相似
    if similar_to_user and similar_to_user.lower() in password.lower():
        return generate_password(min_len, max_len, require_upper, require_lower,
                               require_digit, require_special, forbidden_chars,
                               similar_to_user, no_consecutive, no_repeat)
    
    return password
```

**生成策略：**
1. 先按基本规则生成一个密码
2. 检查是否满足所有规则
3. 填进去试试
4. 如果报错 → 分析错误信息 → 调整规则重新生成

---

### 第5步：填写注册表单

```javascript
// 按扫描到的规则批量填写注册表单
const formData = {
    'email': 'user@example.com',
    'username': 'user_2024', 
    'password': 'Abc123!@#xyz',
    'confirm_password': 'Abc123!@#xyz',
    // 其他必要字段根据注册页面扫描结果补充
};

// 批量填写
Object.entries(formData).forEach(([name, value]) => {
    const el = document.querySelector(`[name="${name}"], #${name}, [placeholder*="${name}"]`);
    if (el) {
        el.value = value;
        el.dispatchEvent(new Event('input', { bubbles: true }));
        el.dispatchEvent(new Event('change', { bubbles: true }));
    }
});
```

---

### 第6步：提交并处理常见报错

```
提交表单
  ↓ 成功 → 跳转到成功页 → 记录账号密码
  ↓
报错！分析错误：
  
  "密码不符合要求" → 重新扫描密码规则 → 重新生成 → 重试
  "密码已被使用"   → 重新生成新密码 → 重试  
  "用户名已存在"   → 生成新的用户名 → 重试
  "邮箱已被注册"   → 告诉用户换邮箱
  "验证码错误"     → 引导用户手动处理验证码
```

**错误处理脚本：**
```javascript
// 获取页面上的错误信息
const errorEl = document.querySelector('.error-message, .el-form-item__error, [class*="error"]');
if (errorEl) {
  console.log("注册错误:", errorEl.textContent);
  // 返回错误类型给Agent分析
  return errorEl.textContent;
}
```

**常见错误及对策速查：**

| 错误信息 | 原因 | 对策 |
|---------|------|------|
| "密码长度不足" / "至少X位" | 密码太短 | 增加长度重新生成 |
| "密码不能包含空格" | 含空格 | 去掉空格重新生成 |
| "需包含大写字母" | 缺大写 | 加上大写字母重新生成 |
| "需包含特殊字符" | 缺特殊字符 | 加上!@#$等重新生成 |
| "密码与用户名相似" | 含用户名 | 重新生成，排除用户名 |
| "不能包含连续字符" | 含123/abc | 重新生成，排除连续 |
| "该密码已被使用" | 密码被用过了 | 重新生成一个完全不同的 |
| "用户名已存在" | 用户名被占 | 加数字后缀重试 |
| "邮箱已被注册" | 邮箱已存在 | 提醒用户换邮箱或找回密码 |
| "验证码错误" | 验证码输错 | 引导用户手动操作 |

---

### 第7步：注册成功 → 保存并告知用户

**保存账号信息到文件：**
```json
{
  "site": "https://example.com",
  "register_time": "2025-06-01 12:00:00",
  "email": "user@example.com",
  "username": "user_2024",
  "password": "Abc123!@#xyz"
}
```

**告知用户：**
```
✅ 账号注册成功！
网站：example.com
账号：user@example.com
密码：Abc123!@#xyz

建议你现在登录一次确认可用，或者修改成自己常用的密码。
密码已保存在 /tmp/account_example.json
```

---

### 特殊情况处理

#### 情况1：注册需要手机验证码

```
遇到手机验证码 → 自动无法绕过

处理方式：
1. 在手机号输入框填入用户提供的手机号
2. 告诉用户：「验证码已发送到你的手机，请输入验证码」
3. 等待用户输入验证码
4. 用户填入后继续下一步
```

#### 情况2：注册需要邮箱验证

```
邮箱验证分两种情况：

A. 发送验证码到邮箱 → 用户自己查邮件 → 输入验证码
B. 发送确认链接 → 用户自己点链接 → Agent继续操作

流程：
1. 填写邮箱并提交
2. 提示用户：「验证邮件已发送到你的邮箱，请查收并确认」
3. 如果是验证码，等用户输入后继续
4. 如果是链接，等用户确认已点击后继续
```

#### 情况3：注册失败多次被限

```
连续注册失败3次以上 → 暂停并告知用户：
"注册多次失败，可能是触发了频率限制。建议：
1. 等15-30分钟后再试
2. 或者你手动注册"
```

---

## 陷阱与注意事项

| 陷阱 | 解决方案 |
|------|----------|
| web_extract 对 JS 渲染页面无效 | 改用 Browser |
| 请求太频繁被 ban | 随机间隔 1-3 秒，20步后大暂停 |
| 页面结构变化导致解析失败 | 加异常处理，输出原始HTML调试 |
| 翻页链接是 JS 生成的 | 用 Browser 模拟点击 |
| Cookie 过期 | 定时刷新或重新登录 |
| 编码问题 | 设置 `encoding="utf-8"` |
| 提交表单后页面跳转慢 | 设置合理的等待时间 |
| 浏览器页面被意外关闭 | 重新打开并恢复会话 |
| 用户中途打断 | 实时保存确保数据不丢 |
| 同时操作多个域名 | 按顺序处理，不混用 |
| 填写下拉/日期选择器反复尝试 | 优先 URL 参数预填或 DOM 直接设值 |
| 自定义下拉选项不出现 | 点击触发器后等 0.5-1 秒再选 |
| 级联选择中间级没有自动展开 | 手动触发下一级的展开事件 |
| 自动补全下拉搜不到结果 | 改用选择器直接点击显示所有选项 |
| 控件是 iframe 内嵌的 | 先切换到 iframe 再操作 |
| 填表后提交按钮是灰色 | 检查是否有字段未填写或校验未通过 |
| 页面用了 shadow DOM | 用 `el.shadowRoot.querySelector()` 穿透 |
| 富文本编辑器无法设值 | 找 contenteditable 元素直接设 innerHTML |
| 某个选项点了没反应 | 检查是否有 event listener 丢了，试试 dispatchEvent |
| 弹窗挡住了操作目标 | 先关闭或确认弹窗再继续 |
| WSL→Windows 连不上 Chrome 调试端口 | 用 `netsh interface portproxy` 做端口转发 + 防火墙放行 |
| CDP WebSocket 地址含 `localhost` 连不上 | 替换为宿主机的网关 IP（WSL2 中通过 `ip route` 获取） |
| CDP `json/new` 返回 405 | 改用 CDP `Page.navigate` 在现有标签页导航，或用 `Target.createTarget` |
| 富文本编辑器内容在 iframe 里取不到 | 用 `iframe.contentDocument.body.innerHTML` 穿透 iframe |
| 页面扫描到导航栏/侧边栏等非表单元素 | 用 `el.closest('.el-form-item, .form-group')` 过滤，只处理表单内的字段 |
| 元素在 shadow DOM 内 | 用 `el.shadowRoot.querySelector()` 穿透 |
| 多个 Chrome 进程同时占用了调试端口 | 先用 `netstat -ano | findstr 9222` 定位旧进程 PID，`Stop-Process -Id` 清理 |
| 端口转发被 Windows 防火墙拦截 | 用 `netsh advfirewall firewall add rule` 添加入站规则放行 |

---

## WSL 操作 Windows Chrome 指南

当 Agent 运行在 WSL（Windows Subsystem for Linux）中，而用户已经在 Windows Chrome 打开了目标页面时，需要解决跨系统浏览器控制问题。

### 网络拓扑

```
WSL2 (Linux)                  Windows
┌──────────┐              ┌──────────┐
│ Python   │  172.28.x.x  │ Chrome   │
│ CDP脚本  │ ───────────→ │ :9222    │
│ :19222   │   portproxy  │ 127.0.0.1│
└──────────┘              └──────────┘
```

### 一键连接脚本

```powershell
# 在 WSL 中执行（需要管理员权限）
# 第1步：确保 Chrome 已以远程调试模式运行
Start-Process -FilePath "C:\Users\%USERNAME%\AppData\Local\Google\Chrome\Application\chrome.exe" `
  -ArgumentList "--remote-debugging-port=9222"

# 第2步：端口转发（管理员权限）
netsh interface portproxy add v4tov4 `
  listenaddress=0.0.0.0 listenport=19222 `
  connectaddress=127.0.0.1 connectport=9222

# 第3步：防火墙放行
netsh advfirewall firewall add rule `
  name="Chrome_CDP_19222" dir=in action=allow protocol=TCP localport=19222
```

然后在 WSL 中通过 `172.28.16.1:19222`（或 `ip route` 获取的网关IP）访问 Chrome 的 DevTools。

### CDP 连接要点

**获取标签页列表：**
```python
import urllib.request, json
resp = urllib.request.urlopen('http://172.28.16.1:19222/json')
targets = json.loads(resp.read())
```

**连接目标页面：**
```python
# 替换 WebSocket URL 中的 localhost 为网关 IP
ws_url = target['webSocketDebuggerUrl']
ws_url = ws_url.replace('ws://localhost:', 'ws://172.28.16.1:')
ws_url = ws_url.replace('ws://127.0.0.1:', 'ws://172.28.16.1:')
```

**导航到目标URL（而非新建标签页）：**
```python
# CDP json/new 在某些 Chrome 版本不可用，用 Page.navigate 代替
await ws.send(json.dumps({
    'id': 1,
    'method': 'Page.navigate',
    'params': {'url': 'https://目标网址'}
}))
```

**穿透 iframe 获取富文本编辑器内容：**
```python
# UEditor / TinyMCE / CKEditor 等编辑器内容在 iframe 内
content = await eval_js('''
document.getElementById('ueditor_0')
  .contentDocument.body.innerHTML
''')
```

**过滤表单内元素（排除导航栏/侧边栏）：**
```javascript
// 只处理 .el-form-item 内的表单字段
document.querySelectorAll('.el-form-item input').forEach(el => {
  const label = el.closest('.el-form-item')
    .querySelector('.el-form-item__label')?.textContent;
});
```

---

这是提升处理能力和效率的关键。无论页面多复杂，以下技巧能大幅减少操作次数、降低失败率。

---

### 技巧1：批量操作 > 逐个操作

**核心思想：** 能用一次操作搞定的事，绝不用多次。

```
❌ 错误做法：
   逐个勾选50个checkbox → 50次操作，5分钟

✅ 正确做法：
   document.querySelectorAll('input[type="checkbox"]').forEach(cb => cb.checked = true);
   // 一次操作搞定
```

**批量操作的3个常用场景：**

**场景A：批量选中/取消**
```javascript
// 批量选中特定值的checkbox
const targetValues = ['红色', '黑色', '白色'];
document.querySelectorAll('input[type="checkbox"]').forEach(cb => {
  cb.checked = targetValues.includes(cb.value);
  cb.dispatchEvent(new Event('change', { bubbles: true }));
});
```

**场景B：批量填写同类字段**
```javascript
// 所有input一次性填完
const fields = {
  'title': '2025新款跑鞋',
  'price': '299',
  'stock': '999'
};
Object.entries(fields).forEach(([name, value]) => {
  const el = document.querySelector(`[name="${name}"], #${name}`);
  if (el) {
    el.value = value;
    el.dispatchEvent(new Event('input', { bubbles: true }));
  }
});
```

**场景C：批量循环翻页**
```python
# 一次性翻完所有页，不每页手动点
for page in range(2, total_pages + 1):
    browser.goto(f"{base_url}?page={page}")
    # 提取本页数据
    save_progress(data)
```

---

### 技巧2：直接DOM操作 > 模拟点击

**核心思想：** DOM操作比模拟点击快10倍，也更稳定。

```
❌ 点击下拉 → 等选项出现 → 找到选项 → 点击 → 等下拉关闭
   （4步，2-3秒）

✅ selectElement.value = '目标值'; dispatchEvent('change')
   （2步，0.01秒）
```

**所有控件的DOM操作速查：**

| 控件 | DOM操作 | 模拟点击操作 | 速度提升 |
|------|---------|-------------|---------|
| select | `.value = 'x'` | 点击→选→等→关闭 | 100x |
| checkbox | `.checked = true` | 逐个点击 | 50x |
| radio | `.checked = true` | 点击 | 50x |
| text input | `.value = 'x'` | 逐字输入 | 20x |
| date input | `.value = '2025-01-01'` | 打开日历→翻月→点日 | 100x |
| file input | `.files = dt.files` | 打开文件选择器 | 无限 |

**什么时候必须用模拟点击：**
- 框架自定义控件，DOM操作不生效
- 需要触发特定JS事件链条
- 点击会触发页面跳转/表单提交

---

### 技巧3：预分析 → 一次性执行

**核心思想：** 先扫描页面获取所有元信息，然后一次性执行，不在执行中反复停下查看。

**预分析阶段收集的信息：**
```
1. 所有字段的 name/id 和类型
2. 所有下拉菜单的选项列表
3. 翻页的URL模式（?page=N 还是 /page/N）
4. UI框架类型（决定使用哪种selector）
5. 表单验证规则（必填字段、格式要求）
6. 页面是否有弹窗/确认框
```

**一次性执行：**
```javascript
// 收集完所有信息后，一次性执行
const formData = {
  title: '2025新款跑鞋',
  price: '299',
  category: '3',     // 提前知道值
  color: ['红','黑'], // 提前知道值
};

// 批量填写
Object.entries(formData).forEach(([name, value]) => {
  const el = document.querySelector(`[name="${name}"]`);
  if (!el) return;
  
  if (el.tagName === 'SELECT') {
    el.value = value;
    el.dispatchEvent(new Event('change', { bubbles: true }));
  } else if (el.type === 'checkbox' && Array.isArray(value)) {
    // 批量处理
  } else {
    el.value = value;
    el.dispatchEvent(new Event('input', { bubbles: true }));
  }
});
```

---

### 技巧4：URL直接导航 > 页面内导航

**核心思想：** 如果能直接通过URL跳转到目标页面，就不在页面内点击导航。

```
❌ 在页面内找"下一页"按钮 → 等待加载 → 找"第2页"
✅ window.location = 'https://example.com/products?page=2'

❌ 点筛选 → 选品类 → 点确认 → 等结果
✅ window.location = 'https://example.com/products?category=shoes&sort=new'
```

**常见URL模式：**

| 目的 | URL模式 | 示例 |
|------|---------|------|
| 翻页 | `?page=N` / `&page=N` / `/page/N` | `?page=3` |
| 筛选 | `?category=X&brand=Y` | `?category=shoes&color=red` |
| 搜索 | `?q=关键词` | `?q=电动牙刷` |
| 排序 | `?sort=price_asc` | `?sort=newest` |
| 分页大小 | `&per_page=100` | `&per_page=100` |

**技巧：** 先用 `?per_page=100` 或 `?limit=100` 尝试一次获取更多数据，减少翻页次数。

---

### 技巧5：并发处理 > 串行处理

**核心思想：** 对于独立页面的数据提取，可以并发，不用逐一等待。

**使用 delegate_task 并行抓取：**
```python
from hermes_tools import delegate_task

# 并行处理多个独立任务
results = delegate_task(tasks=[
    {"goal": "抓取页面A的所有数据并保存", "toolsets": ["web", "terminal"]},
    {"goal": "抓取页面B的所有数据并保存", "toolsets": ["web", "terminal"]},
    {"goal": "抓取页面C的所有数据并保存", "toolsets": ["web", "terminal"]},
])
```

**使用 execute_code 内的并发：**
```python
import concurrent.futures
from hermes_tools import web_extract, write_file

urls = ['https://a.com/1', 'https://a.com/2', 'https://a.com/3']

def fetch(url):
    result = web_extract(urls=[url])
    return {"url": url, "data": result}

with concurrent.futures.ThreadPoolExecutor(max_workers=3) as exe:
    results = list(exe.map(fetch, urls))

write_file("/tmp/parallel_results.json", json.dumps(results, indent=2))
```

**注意事项：**
- 并发数不要超过3-5，避免触发反爬
- 只对独立页面使用并发（不依赖前序结果的页面）
- 带有登录态的不要用并发（cookie共享问题）

---

### 技巧6：错误预判 + 自动降级

**核心思想：** 操作前预判可能失败的点，提前准备降级方案。

**常见失败模式与降级方案：**

```
操作A（最快方案）
  ↓ 失败？
操作B（备用方案）
  ↓ 再失败？
操作C（兜底方案）
```

**具体实现：**
```javascript
function trySetValue(selector, value, options = {}) {
  const {
    method = 'value',          // 'value' | 'click' | 'keyboard'
    eventType = 'change',      // 事件类型
    waitMs = 500,              // 失败重试等待
    retry = 2,                 // 重试次数
  } = options;

  const el = document.querySelector(selector);
  if (!el) return false;

  for (let i = 0; i <= retry; i++) {
    try {
      if (method === 'value') {
        el.value = value;
        el.dispatchEvent(new Event(eventType, { bubbles: true }));
        return el.value === value;
      } else if (method === 'click') {
        el.click();
        return true;
      }
    } catch(e) {
      if (i < retry) {
        // 等一会再重试
        await new Promise(r => setTimeout(r, waitMs));
      }
    }
  }
  return false;
}

// 使用：先试最快方案
if (!trySetValue('select[name="cat"]', '3', { method: 'value' })) {
  // 不行就模拟点击
  trySetValue('.ant-select-selector', null, { method: 'click' });
  await new Promise(r => setTimeout(r, 500));
  // 再选选项
  document.querySelector('.ant-select-item-option[title="电子产品"]')?.click();
}
```

---

### 技巧7：页面状态快照

**核心思想：** 关键操作前保存页面状态快照，操作失败时可以回溯。

**快照内容：**
```python
import json
from hermes_tools import write_file

snapshot = {
    "timestamp": "2025-01-01T12:00:00",
    "current_url": "https://example.com/products/123/edit",
    "form_values": {
        "title": "当前标题值",
        "price": "当前价格值",
    },
    "step": "填写表单第3步",
    "next_action": "点击提交按钮"
}

write_file("/tmp/page_snapshot.json", json.dumps(snapshot, indent=2))
```

**什么时候保存快照：**
- 填写表单前
- 点击提交前
- 每次翻页前（批量翻页时）
- 关键数据提取后

---

### 技巧8：简化表单 → 只操作必要字段

页面可能有大量字段，但很多可以：
1. **直接忽略非必填** — 除非用户指定要填
2. **使用表单的预设值** — 不修改已有内容
3. **批量跳过无关字段** — 聚焦用户关心的

```javascript
// 只处理与用户任务相关的字段
const REQUIRED_FIELDS = ['title', 'price', 'category'];

document.querySelectorAll('input, select, textarea').forEach(el => {
  const name = el.name || el.id;
  if (!REQUIRED_FIELDS.includes(name)) {
    // 跳过非必要字段
    return;
  }
  // 处理必要字段
});
```

---

### 效率提升总结

| 技巧 | 效率提升 | 适用场景 |
|------|---------|---------|
| 批量操作 | 5-50x | checkbox/radio、批量填表 |
| 直接DOM操作 | 10-100x | 所有表单控件 |
| 预分析+一次性执行 | 3-10x | 复杂表单填写 |
| URL直接导航 | 2-5x | 翻页、筛选、搜索 |
| 并发处理 | 2-3x | 独立页面抓取 |
| 错误预判+自动降级 | 减少50%失败 | 不稳定页面 |
| 页面状态快照 | 减少重做 | 长流程任务 |
| 简化表单 | 减少50%操作 | 表单填写 |

---

## 相关参考文件

本技能目录下提供了以下参考文件：

| 文件 | 说明 |
|------|------|
| `references/connect-windows-chrome.md` | 从 WSL 连接到 Windows Chrome 的远程调试设置流程 |
| `references/ui-framework-selectors.md` | UI框架选择器速查表（待完善） |
| `references/copywriting-tones.md` | 话术风格参考（待完善） |

## 质量检查清单

- [ ] 数据完整性：确认所有目标条目都已获取（数量对齐）
- [ ] 数据格式：JSON/CSV 可正常读取
- [ ] 防检测：请求间隔随机化，不固定频率
- [ ] 延迟策略：请求间隔合理且随机
- [ ] 实时保存：中途数据已写入文件
- [ ] 单页面：未重复打开同域名页面
- [ ] 表单验证：提交前字段已正确填写
- [ ] 错误处理：网络错误、解析错误有捕获
- [ ] 用户提示：需要用户操作的步骤有清晰说明
- [ ] 结果文件：最终数据已保存到文件系统
- [ ] 媒体文件：生成的文件已确认存在且格式正确
- [ ] 上传验证：页面上已出现上传成功的提示/预览
- [ ] 严格遵循用户要求：所有填写内容都能从用户原话中找到出处
- [ ] 无自创内容：没有自己编造任何文案、数据或选择
- [ ] 补全模式判断正确：已按用户指令程度选择A/B/C模式
- [ ] 话术自然：所有描述内容模拟人类语气，无机器感
- [ ] 场合匹配：话术风格与平台/商品类型一致
- [ ] 批量操作优先：能用DOM操作不模拟点击
- [ ] 预分析先行：已先扫描页面再一次性执行
- [ ] URL导航利用：翻页/筛选优先URL参数而非点击
- [ ] 状态快照保存：关键操作前已保存页面状态
- [ ] 自动降级就绪：最快方案失败后有备用方案
- [ ] 图片识别：已用豆包视觉模型分析图片内容
- [ ] 图填联动：图片识别结果已用于填写相关表单字段
- [ ] 先问需求：进入新页面后先确认用户要什么
- [ ] 先规划再执行：输出计划后让用户确认
- [ ] 流程不乱：严格按照需求→方案→扫描→计划→执行→验证的顺序
- [ ] 每步验证：每一步完成后检查是否成功
- [ ] 信息提取精准：只提取用户需要的相关字段，不提取无关数据
