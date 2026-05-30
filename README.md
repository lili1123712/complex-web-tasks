# Complex Web Tasks

> 处理复杂的网页任务 — 多步骤网页交互、表单填写、数据爬取、登录流程、分页抓取、动态内容提取等。

适用于 Hermes Agent 的网页自动化处理技能。

## 功能

- **深度页面扫描** — 区块划分法逐块分析页面，标注每个元素的类型和用途
- **复杂控件处理** — 16种表单控件处理方案，10+ UI框架适配
- **防人机验证** — 随机延迟、模拟真人操作节奏
- **数据采集与实时保存** — 分页抓取、并发抓取，每步保存到临时文件
- **媒体生成与自动上传** — ComfyUI/fal-ai/MiniMax生成 → 自动上传
- **三种填充模式** — 精确模式 / 智能补全 / 完全委托
- **话术策略** — 8种平台 × 7类商品的自然语言自动生成
- **账号注册** — 自动扫描密码规则、生成合规密码、处理报错
- **效率技巧** — 批量操作(5-50x)、DOM优先(10-100x)、URL导航、并发处理

## 安装

将本仓库克隆到 Hermes Agent 的 skills 目录下：

```bash
git clone https://github.com/lili1123712/complex-web-tasks.git \
  ~/.hermes/skills/web-scraping-skills/complex-web-tasks
```

或在 Hermes Agent 中加载：

```
/skill load complex-web-tasks
```
