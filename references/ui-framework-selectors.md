# UI Framework Selectors Cheat Sheet

Quick reference for common UI framework selectors used in web automation.

---

## 下拉菜单 (Select / Dropdown)

| 框架 | 触发器点击 | 选项面板 | 选项项 |
|------|-----------|---------|-------|
| Ant Design | `.ant-select-selector` | `.ant-select-dropdown` | `.ant-select-item-option` |
| Ant Design Vue | `.ant-select-selector` | `.ant-select-dropdown` | `.ant-select-item-option` |
| Element UI | `.el-select .el-input` | `.el-select-dropdown` | `.el-select-dropdown__item` |
| Element Plus | `.el-select-v2` | `.el-select-dropdown` | `.el-select-dropdown__item` |
| Naive UI | `.n-base-selection` | `.n-select-menu` | `.n-select-option` |
| React Select | `.css-xxx-control` | `.css-xxx-menu` | `.css-xxx-option` |
| Vuetify | `.v-select` | `.v-menu__content` | `.v-list-item` |
| Chakra UI | `.chakra-select__wrapper` | `.chakra-select__menu` | `[role="option"]` |
| MUI (Material) | `.MuiSelect-select` | `.MuiPaper-root` | `.MuiMenuItem-root` |
| Headless UI | `[role="combobox"]` | `[role="listbox"]` | `[role="option"]` |
| Bootstrap | `.dropdown-toggle` | `.dropdown-menu` | `.dropdown-item` |
| 原生 | `select[name="x"]` | n/a | `option[value="x"]` |

## 级联选择器 (Cascader)

| 框架 | 触发器 | 选项菜单 | 选项项 |
|------|--------|---------|-------|
| Ant Design | `.ant-cascader-picker` | `.ant-cascader-menus` | `.ant-cascader-menu-item` |
| Element UI | `.el-cascader` | `.el-cascader__dropdown` | `.el-cascader-menu__item` |
| Element Plus | `.el-cascader` | `.el-cascader__dropdown` | `.el-cascader-node__label` |

## 日期选择器 (DatePicker)

| 框架 | 触发器 | 日历面板 | 日期单元格 |
|------|--------|---------|-----------|
| Ant Design | `.ant-picker` | `.ant-picker-dropdown` | `.ant-picker-cell` |
| Element UI | `.el-date-editor` | `.el-picker-panel` | `.el-date-table-cell` |
| flatpickr | `.flatpickr-input` | `.flatpickr-calendar` | `.flatpickr-day` |
| laydate | `.layui-input` | `.layui-laydate` | `.laydate-day` |

## 复选框 (Checkbox)

| 框架 | 包裹器 | 选中态class |
|------|--------|-----------|
| Ant Design | `.ant-checkbox-wrapper` | `.ant-checkbox-checked` |
| Element UI | `.el-checkbox` | `.is-checked` |
| 原生 | `input[type="checkbox"]` | `:checked` |

## 单选 (Radio)

| 框架 | 包裹器 | 选中态 |
|------|--------|-------|
| Ant Design | `.ant-radio-wrapper` | `.ant-radio-checked` |
| Element UI | `.el-radio` | `.is-checked` |
| 原生 | `input[type="radio"]` | `:checked` |

## 文件上传 (Upload)

| 框架 | 拖拽区域 | 隐藏input |
|------|---------|----------|
| Ant Design | `.ant-upload-drag` | `.ant-upload input[type="file"]` |
| Element UI | `.el-upload-dragger` | `.el-upload__input` |
| 原生 | n/a | `input[type="file"]` |

## 弹窗 / 模态框 (Modal / Dialog)

| 框架 | 关闭按钮 | 确认按钮 | 取消按钮 |
|------|---------|---------|---------|
| Ant Design | `.ant-modal-close` | `.ant-modal-confirm-btns .ant-btn-primary` | `.ant-modal-confirm-btns .ant-btn:not(.ant-btn-primary)` |
| Element UI | `.el-dialog__close` | `.el-message-box__btns .el-button--primary` | `.el-message-box__btns .el-button:not(.el-button--primary)` |

## Tab 切换

| 框架 | Tab标题 | 激活态class |
|------|---------|-----------|
| Ant Design | `.ant-tabs-tab` | `.ant-tabs-tab-active` |
| Element UI | `.el-tabs__item` | `.is-active` |

## 树形选择器 (TreeSelect)

| 框架 | 展开箭头 | 节点标题 |
|------|---------|---------|
| Ant Design | `.ant-select-tree-switcher` | `.ant-select-tree-title` |
| Element UI | `.el-tree-node__expand-icon` | `.el-tree-node__label` |

## 标签输入 (Tag Input)

| 框架 | 输入框 |
|------|--------|
| Ant Design | `.ant-select-selection-search-input` (mode="tags") |
| Element UI | `.el-tag__input input` |
| Naive UI | `.n-tag-input__input` |

## 富文本编辑器

| 编辑器 | 内容区 | 设值方法 |
|--------|-------|---------|
| Quill | `.ql-editor` | `el.innerHTML = html` |
| ProseMirror / Tiptap | `.ProseMirror` | `el.innerHTML = html` |
| TinyMCE | 通过API | `tinymce.get('id').setContent(html)` |
| CKEditor | 通过API | `CKEDITOR.instances.id.setData(html)` |
| 通用 contenteditable | `[contenteditable="true"]` | `el.innerHTML = html` |

## UI框架检测脚本

```javascript
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
  tailwind: !!document.querySelector('[class*="select"], [class*="dropdown"]'),
};
console.log('Detected:', Object.entries(frameworks).filter(([k,v]) => v).map(([k]) => k));
```

## 列出所有表单元素

```javascript
document.querySelectorAll('input, select, textarea, [role="combobox"], [role="textbox"], [contenteditable="true"]')
  .forEach(el => console.log({
    tag: el.tagName,
    type: el.type || el.getAttribute('role'),
    name: el.name || el.id,
    required: el.required || el.getAttribute('aria-required'),
    placeholder: el.placeholder,
    value: el.value || el.textContent?.slice(0, 50),
  }));
```
