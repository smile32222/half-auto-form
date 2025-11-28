# DynamicFormDialog 组件使用说明

## 概述

`DynamicFormDialog` 是一个高度可配置的动态表单对话框组件，支持多种表单控件类型、级联选择、条件显示、自定义验证等功能。通过配置化的方式快速生成表单，适用于新增、编辑等场景。

## 功能特性

- ✅ 支持多种表单控件类型（输入框、下拉选择、文本域、单选按钮等）
- ✅ 支持级联下拉选择（异步加载数据）
- ✅ 支持条件显示/隐藏字段
- ✅ 支持自定义验证规则
- ✅ 支持行内表单布局（两列布局）
- ✅ 支持自定义按钮样式和额外操作按钮
- ✅ 支持 Cron 表达式选择器
- ✅ 支持编辑模式下数据回显

---

## Props 参数

| 参数名              | 类型          | 默认值     | 必填 | 说明                         |
| ------------------- | ------------- | ---------- | ---- | ---------------------------- |
| `visible`           | Boolean       | `false`    | 是   | 控制对话框显示/隐藏          |
| `formConfig`        | Array         | -          | 是   | 表单配置项数组，定义表单结构 |
| `initialData`       | Object        | `{}`       | 否   | 初始数据（用于编辑场景）     |
| `title`             | String        | `'表单'`   | 否   | 对话框标题                   |
| `dialogWidth`       | String        | `'600px'`  | 否   | 对话框宽度                   |
| `labelWidth`        | String/Number | `'100px'`  | 否   | 表单项 label 宽度            |
| `inlineTag`         | Boolean       | `false`    | 否   | 是否启用两列布局             |
| `submitButtonText`  | String        | `'确 定'`  | 否   | 提交按钮文本                 |
| `cancelButtonStyle` | Object        | 见默认值   | 否   | 取消按钮样式                 |
| `submitButtonStyle` | Object        | 见默认值   | 否   | 确定按钮样式                 |
| `extraButton`       | Object        | 见配置说明 | 否   | 额外按钮配置                 |

---

## Events 事件

| 事件名               | 参数       | 说明                                      |
| -------------------- | ---------- | ----------------------------------------- |
| `submit`             | `formData` | 表单验证通过后触发，返回表单数据          |
| `update:visible`     | `boolean`  | 关闭对话框时同步更新 visible 状态         |
| `extra-button-click` | `formData` | 点击额外按钮时触发（如果未设置 callback） |

---

## formConfig 配置项说明

### 基础配置

每个表单项的基础配置：

```javascript
{
  prop: 'fieldName',        // 字段名（必填）
  label: '字段标题',         // 显示标签（必填）
  type: 'input',            // 控件类型（见支持的类型）
  placeholder: '请输入',     // 占位符
  defaultValue: '',         // 默认值
  disabled: false,          // 是否禁用
  labelWidth: '120px',      // 单独设置该项的 label 宽度
  rules: [/* 验证规则 */],  // Element UI 验证规则
}
```

### 支持的控件类型

#### 1. 输入框 (`input`)

```javascript
{
  type: 'input',           // 或不设置 type，默认为 input
  prop: 'name',
  label: '姓名',
  placeholder: '请输入姓名',
  maxlength: 50,           // 最大长度
  inputType: 'text',       // 输入类型：text/password/number 等
  onChange: (value) => {   // 值变化回调
    console.log('输入值：', value);
  }
}
```

#### 2. 下拉选择 (`select`)

```javascript
{
  type: 'select',
  prop: 'category',
  label: '分类',
  placeholder: '请选择分类',
  clearable: true,         // 是否可清空
  options: [               // 选项数组
    { label: '选项1', value: 1 },
    { label: '选项2', value: 2, disabled: true }
  ],
  labelKey: 'label',       // 显示字段名（默认 'label'）
  valueKey: 'value',       // 值字段名（默认 'value'）
  onChange: (value, form) => {  // 选中值变化回调
    console.log('选中值：', value);
    console.log('整个表单：', form);
  }
}
```

#### 3. 级联下拉 (`cascade-select`)

支持异步加载的级联选择器：

```javascript
// 第一级（根级）
{
  type: 'cascade-select',
  prop: 'province',
  label: '省份',
  placeholder: '请选择省份',
  clearable: true,
  children: 'city',         // 指定子级字段名
  fetchOptions: async () => {  // 异步获取选项
    const res = await getProvinceList();
    return res.data;
  },
  labelKey: 'name',
  valueKey: 'id'
},
// 第二级（依赖于第一级）
{
  type: 'cascade-select',
  prop: 'city',
  label: '城市',
  placeholder: '请选择城市',
  dependsOn: 'province',    // 依赖的父级字段
  fetchOptions: async (provinceId) => {  // 参数为父级选中值
    const res = await getCityList(provinceId);
    return res.data;
  },
  pushSelectCurrent: true,  // 编辑时如果当前值不在选项中，显示该值
  selectCurrentStr: '当前选中（已失效）',  // 占位文本
  labelKey: 'name',
  valueKey: 'id'
}
```

#### 4. 文本域 (`textarea`)

```javascript
{
  type: 'textarea',
  prop: 'description',
  label: '描述',
  placeholder: '请输入描述',
  rows: 4                  // 文本域行数
}
```

#### 5. 单选按钮 (`radio`)

带边框样式的单选按钮组：

```javascript
{
  type: 'radio',
  prop: 'status',
  label: '状态',
  options: [
    { label: '启用', value: 1 },
    { label: '禁用', value: 0, disabled: true }
  ],
  onChange: (value) => {
    console.log('选中状态：', value);
  }
}
```

#### 6. Cron 表达式选择器 (`corn-select`)

```javascript
{
  type: 'corn-select',
  prop: 'cronExpression',
  label: '定时策略',
  placeholder: '请选择定时策略'
}
```

#### 7. 纯文本 (`text`)

```javascript
{
  type: 'text',
  label: '',
  text: '这是一段提示文字'
}
```

### 高级配置

#### 条件显示

使用 `visibleOn` 函数动态控制字段显示：

```javascript
{
  type: 'input',
  prop: 'email',
  label: '邮箱',
  visibleOn: (form) => {    // 只有选中"启用"时才显示
    return form.status === 1;
  }
}
```

#### 选项禁用控制

```javascript
{
  type: 'select',
  prop: 'role',
  label: '角色',
  options: [
    { label: '管理员', value: 'admin' },
    { label: '普通用户', value: 'user' }
  ],
  disabled: (item, form) => {  // 动态禁用某些选项
    return item.value === 'admin' && form.userType !== 'super';
  }
}
```

#### 验证规则

```javascript
{
  type: 'input',
  prop: 'phone',
  label: '手机号',
  rules: [
    { required: true, message: '请输入手机号', trigger: 'blur' },
    {
      pattern: /^1[3-9]\d{9}$/,
      message: '请输入正确的手机号',
      trigger: 'blur'
    }
  ]
}
```

---

## extraButton 额外按钮配置

```javascript
{
  visible: true,           // 是否显示
  text: '测试连接',         // 按钮文字
  validate: true,          // 点击时是否需要验证表单
  callback: (formData) => { // 点击回调函数
    console.log('额外按钮点击', formData);
  },
  style: {                 // 按钮样式
    minWidth: '100px',
    margin: '0 12px'
  }
}
```

---

## 完整使用示例

### 1. 基础新增表单

```vue
<template>
  <div>
    <el-button @click="openDialog">新增用户</el-button>

    <DynamicFormDialog
      :visible.sync="dialogVisible"
      :formConfig="formConfig"
      title="新增用户"
      @submit="handleSubmit"
    />
  </div>
</template>

<script>
import DynamicFormDialog from "@/components/dynamicFormDialog.vue";

export default {
  components: { DynamicFormDialog },
  data() {
    return {
      dialogVisible: false,
      formConfig: [
        {
          prop: "username",
          label: "用户名",
          type: "input",
          placeholder: "请输入用户名",
          rules: [{ required: true, message: "请输入用户名", trigger: "blur" }],
        },
        {
          prop: "role",
          label: "角色",
          type: "select",
          placeholder: "请选择角色",
          options: [
            { label: "管理员", value: "admin" },
            { label: "普通用户", value: "user" },
          ],
          rules: [{ required: true, message: "请选择角色", trigger: "change" }],
        },
        {
          prop: "email",
          label: "邮箱",
          type: "input",
          placeholder: "请输入邮箱",
          rules: [
            { type: "email", message: "请输入正确的邮箱", trigger: "blur" },
          ],
        },
        {
          prop: "description",
          label: "描述",
          type: "textarea",
          placeholder: "请输入描述",
          rows: 3,
        },
      ],
    };
  },
  methods: {
    openDialog() {
      this.dialogVisible = true;
    },
    handleSubmit(formData) {
      console.log("提交数据：", formData);
      // 调用 API 保存数据
      this.dialogVisible = false;
    },
  },
};
</script>
```

### 2. 编辑表单（数据回显）

```vue
<template>
  <DynamicFormDialog
    :visible.sync="dialogVisible"
    :formConfig="formConfig"
    :initialData="editData"
    title="编辑用户"
    @submit="handleUpdate"
  />
</template>

<script>
export default {
  data() {
    return {
      dialogVisible: false,
      editData: {
        id: 1,
        username: "张三",
        role: "admin",
        email: "zhangsan@example.com",
      },
      formConfig: [
        /* 同上 */
      ],
    };
  },
  methods: {
    handleUpdate(formData) {
      console.log("更新数据：", formData);
      // formData 包含 id 字段
    },
  },
};
</script>
```

### 3. 级联选择示例

```vue
<script>
export default {
  data() {
    return {
      formConfig: [
        {
          type: "cascade-select",
          prop: "industryId",
          label: "行业",
          placeholder: "请选择行业",
          children: "productLineId",
          fetchOptions: async () => {
            const res = await this.$axios.get("/api/industries");
            return res.data;
          },
          labelKey: "name",
          valueKey: "id",
        },
        {
          type: "cascade-select",
          prop: "productLineId",
          label: "产品线",
          placeholder: "请选择产品线",
          dependsOn: "industryId",
          fetchOptions: async (industryId) => {
            const res = await this.$axios.get(
              `/api/product-lines?industryId=${industryId}`
            );
            return res.data;
          },
          labelKey: "name",
          valueKey: "id",
          pushSelectCurrent: true, // 当前给form的值不在 options 中，并且选中值不是空，则注入一个临时选项
          selectCurrentStr: "当前选中（可能已失效）", // pushSelectCurrent为true时，选择显示的文字
        },
      ],
    };
  },
};
</script>
```

### 4. 两列布局 + 额外按钮

```vue
<template>
  <DynamicFormDialog
    :visible.sync="dialogVisible"
    :formConfig="formConfig"
    :inlineTag="true"
    dialogWidth="800px"
    :extraButton="extraButtonConfig"
    @extra-button-click="handleTest"
    @submit="handleSubmit"
  />
</template>

<script>
export default {
  data() {
    return {
      extraButtonConfig: {
        visible: true,
        text: "测试连接",
        validate: false, // 不验证表单
        callback: (formData) => {
          this.testConnection(formData);
        },
      },
    };
  },
  methods: {
    testConnection(formData) {
      console.log("测试连接", formData);
    },
  },
};
</script>
```

### 5. 条件显示字段

```vue
<script>
export default {
  data() {
    return {
      formConfig: [
        {
          type: "radio",
          prop: "notifyType",
          label: "通知方式",
          options: [
            { label: "邮箱", value: "email" },
            { label: "短信", value: "sms" },
          ],
        },
        {
          type: "input",
          prop: "email",
          label: "邮箱地址",
          placeholder: "请输入邮箱",
          visibleOn: (form) => form.notifyType === "email",
        },
        {
          type: "input",
          prop: "phone",
          label: "手机号",
          placeholder: "请输入手机号",
          visibleOn: (form) => form.notifyType === "sms",
        },
      ],
    };
  },
};
</script>
```

---

## 注意事项

1. **级联选择的数据缓存**：组件会自动缓存已加载的级联数据，避免重复请求
2. **编辑模式下的级联数据**：使用 `pushSelectCurrent: true` 可以在编辑时显示当前值（即使不在新加载的选项中）
3. **表单验证**：提交时会自动验证，只有通过验证才会触发 `submit` 事件
4. **双向绑定**：使用 `:visible.sync="dialogVisible"` 实现对话框状态的双向绑定
5. **ID 字段**：编辑模式下，`initialData` 中的 `id` 字段会自动添加到表单数据中

---

## 样式定制

组件提供了一些内置样式类：

- `.dynamic-form`：表单容器
- `.radio-customize-check`：单选按钮组的自定义样式

可以通过全局样式或 scoped 样式进行覆盖。

---

## 常见问题

### Q1: 如何动态更新 formConfig？

A: 直接修改 `formConfig` 数组，组件会自动响应变化。

### Q2: 级联选择如何处理异步加载失败？

A: 在 `fetchOptions` 函数中添加 try-catch 处理错误情况。

### Q3: 如何禁用某些选项？

A: 在选项对象中添加 `disabled: true`，或使用 `disabled` 函数动态控制。

### Q4: 表单数据如何重置？

A: 关闭对话框时会自动调用 `resetFields()` 重置表单。

---

## 更新日志

- 支持 Cron 表达式选择器
- 支持级联下拉选择（异步加载）
- 支持条件显示字段
- 支持额外操作按钮配置
- 支持两列布局模式
