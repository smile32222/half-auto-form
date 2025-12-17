# FilterTreeSelector 组件使用文档

## 组件简介

`FilterTreeSelector` 是一个灵活的多级树形筛选组件,支持前 N 级平铺展示(Radio 单选),并可通过弹窗浏览更深层级的树形结构。组件支持两种数据加载模式:懒加载和全量数据。

### 核心特性

- ✅ **可配置层级数**:通过 `levels` prop 自定义平铺展示的层级数量
- ✅ **自定义层级名称**:支持数组或对象格式定义每级名称
- ✅ **双模式支持**:
  - `lazy` 模式:按需异步加载数据
  - `full` 模式:一次性加载全量数据,支持关键词搜索
- ✅ **面包屑导航**:显示当前筛选路径,支持点击回溯
- ✅ **更多子级弹窗**:树形结构浏览更深层级
- ✅ **无障碍支持**:完善的 ARIA 标签和键盘导航

---

## Props 属性

### 基础配置

| 属性           | 类型              | 默认值   | 说明                        |
| -------------- | ----------------- | -------- | --------------------------- |
| `levels`       | `Number`          | `3`      | 平铺展示的层级数量,最小为 1 |
| `levelNames`   | `Array \| Object` | `{}`     | 各级名称配置,见下方说明     |
| `defaultLabel` | `String`          | `"全部"` | 未选择时的默认标签文本      |
| `filterTitle`  | `String`          | `"筛选"` | 组件标题                    |

#### levelNames 配置说明

支持两种格式:

```javascript
// 数组格式(索引从 0 开始)
levelNames: ['产业', '板块', '产线']

// 对象格式(键为层级,从 1 开始)
levelNames: {
  1: '产业',
  2: '板块',
  3: '产线'
}

// 如果不配置,默认显示"1级分类"、"2级分类"等
```

### 数据模式配置

| 属性            | 类型       | 默认值   | 说明                              |
| --------------- | ---------- | -------- | --------------------------------- |
| `mode`          | `String`   | `"lazy"` | 数据加载模式:`"lazy"` 或 `"full"` |
| `treeData`      | `Array`    | `[]`     | `full` 模式下的完整树数据         |
| `loadChildren`  | `Function` | `null`   | `lazy` 模式下的异步加载函数       |
| `initialLevel1` | `Array`    | `[]`     | 第一级的初始数据(可选)            |

#### loadChildren 函数说明

在 `lazy` 模式下,`loadChildren` 函数用于异步加载子节点数据。

**函数签名:**

```javascript
/**
 * @param {String|Number|null} parentId - 父节点 ID,顶层为 null
 * @param {Number} level - 要加载的层级(从 1 开始)
 * @returns {Promise<Array>} 返回子节点数组
 */
loadChildren(parentId, level);
```

**示例:**

```javascript
async loadChildren(parentId, level) {
  const response = await api.getChildren({ parentId, level });
  return response.data; // 返回子节点数组
}
```

### 字段映射配置

| 属性            | 类型     | 默认值       | 说明               |
| --------------- | -------- | ------------ | ------------------ |
| `idField`       | `String` | `"id"`       | 节点唯一标识字段名 |
| `labelField`    | `String` | `"label"`    | 节点显示文本字段名 |
| `childrenField` | `String` | `"children"` | 子节点数组字段名   |

### UI 配置

| 属性                  | 类型               | 默认值    | 说明                           |
| --------------------- | ------------------ | --------- | ------------------------------ |
| `moreDialogWidth`     | `String`           | `"800px"` | "更多子级"弹窗宽度             |
| `radiosVisibleLimit`  | `Number`           | `8`       | 单选项超过此数量时显示滚动遮罩 |
| `radiosVisibleHeight` | `Number`           | `100`     | 折叠状态下的最大高度(px)       |
| `resetKey`            | `String \| Number` | `null`    | 外部重置控制键,改变时触发重置  |

---

## Events 事件

### filter-change

当筛选条件改变时触发。

**事件参数:**

```javascript
{
  path: [             // 完整路径数组
    { id: '1', label: '产业A' },
    { id: '11', label: '板块B' },
    { id: '111', label: '产线C' }
  ],
  filterPathIds: ['1', '11', '111']  // 路径 ID 数组
}
```

### filter-cleared

当点击"清除"按钮时触发,无参数。

---

## 使用示例

### 示例 1: Lazy 模式(按需加载)

```vue
<template>
  <FilterTreeSelector
    :levels="3"
    :levelNames="['产业', '板块', '产线']"
    mode="lazy"
    :loadChildren="loadProductChildren"
    :initialLevel1="level1Data"
    @filter-change="handleFilterChange"
    @filter-cleared="handleFilterCleared"
  />
</template>

<script>
import FilterTreeSelector from "./FilterTreeSelector.vue";
import { getProductChildren } from "@/api/product";

export default {
  components: { FilterTreeSelector },
  data() {
    return {
      level1Data: [],
    };
  },
  methods: {
    async loadProductChildren(parentId, level) {
      try {
        const response = await getProductChildren({
          parentId,
          level,
        });
        return response.data;
      } catch (error) {
        console.error("加载失败:", error);
        return [];
      }
    },

    handleFilterChange({ path, filterPathIds }) {
      console.log("当前筛选路径:", path);
      console.log("路径 IDs:", filterPathIds);
      // 根据筛选条件更新列表数据
      this.fetchData(filterPathIds);
    },

    handleFilterCleared() {
      console.log("筛选已清除");
      this.fetchData([]);
    },
  },
};
</script>
```

### 示例 2: Full 模式(全量数据 + 搜索)

```vue
<template>
  <FilterTreeSelector
    :levels="3"
    :levelNames="{ 1: '一级', 2: '二级', 3: '三级' }"
    mode="full"
    :treeData="fullTreeData"
    @filter-change="handleFilterChange"
  />
</template>

<script>
import FilterTreeSelector from "./FilterTreeSelector.vue";

export default {
  components: { FilterTreeSelector },
  data() {
    return {
      fullTreeData: [
        {
          id: "1",
          label: "分类A",
          children: [
            {
              id: "11",
              label: "子分类A1",
              children: [
                { id: "111", label: "明细A11" },
                { id: "112", label: "明细A12" },
              ],
            },
          ],
        },
        {
          id: "2",
          label: "分类B",
          children: [
            // ...
          ],
        },
      ],
    };
  },
  methods: {
    handleFilterChange({ path, filterPathIds }) {
      // 处理筛选变化
    },
  },
};
</script>
```

### 示例 3: 自定义字段名

```vue
<template>
  <FilterTreeSelector
    :levels="2"
    :levelNames="['类别', '子类']"
    mode="full"
    :treeData="customData"
    idField="code"
    labelField="name"
    childrenField="subItems"
    @filter-change="handleFilterChange"
  />
</template>

<script>
export default {
  data() {
    return {
      customData: [
        {
          code: "A",
          name: "类别A",
          subItems: [{ code: "A1", name: "子类A1", subItems: [] }],
        },
      ],
    };
  },
};
</script>
```

### 示例 4: 外部重置控制

```vue
<template>
  <div>
    <el-button @click="resetFilter">重置筛选</el-button>
    <FilterTreeSelector
      :levels="3"
      mode="lazy"
      :loadChildren="loadChildren"
      :resetKey="resetKey"
      @filter-change="handleFilterChange"
    />
  </div>
</template>

<script>
export default {
  data() {
    return {
      resetKey: 0,
    };
  },
  methods: {
    resetFilter() {
      this.resetKey++; // 改变 resetKey 触发组件重置
    },
  },
};
</script>
```

---

## 数据结构说明

### 节点数据格式

默认格式:

```javascript
{
  id: 'unique-id',           // 唯一标识
  label: '显示文本',          // 显示名称
  children: [                // 子节点数组
    { id: 'child-1', label: '子节点1', children: [] }
  ]
}
```

自定义格式(通过 `idField`、`labelField`、`childrenField` 配置):

```javascript
{
  code: 'A001',              // 自定义ID字段
  name: '产品A',             // 自定义名称字段
  subItems: [                // 自定义子节点字段
    { code: 'A001-1', name: '子产品1', subItems: [] }
  ]
}
```

---

## 样式定制

组件使用 scoped 样式,如需自定义样式,可使用 `::v-deep` 或 `/deep/`:

```vue
<style>
/* 修改选中项背景色 */
::v-deep .level-radio.is-checked {
  background: rgba(64, 158, 255, 0.1);
  border-color: rgba(64, 158, 255, 0.3);
}

/* 修改标题样式 */
::v-deep .filter-label {
  font-size: 14px;
  color: #303133;
}
</style>
```

---

## 无障碍支持

组件内置完善的无障碍功能:

- **ARIA 标签**: 所有交互元素都有适当的 `aria-label`
- **键盘导航**:
  - `Tab` 键切换焦点
  - `Enter` / `Space` 激活选项
  - 支持屏幕阅读器
- **语义化 HTML**: 使用 `role` 属性标识列表和列表项

---

## 注意事项

1. **层级数配置**: `levels` 至少为 1,建议不超过 5 级以保证良好的用户体验
2. **数据模式选择**:
   - 数据量小(<1000 节点)且需要搜索功能 → 使用 `full` 模式
   - 数据量大或层级深 → 使用 `lazy` 模式
3. **loadChildren 函数**: 必须返回 Promise,即使加载失败也应 resolve 空数组
4. **性能优化**: 在 `lazy` 模式下,组件会缓存已加载的数据,避免重复请求
5. **面包屑限制**: 面包屑仅显示前 N 级平铺选择的路径,弹窗选择的深层路径会追加显示

---

## 版本兼容

- Vue: 2.x
- Element UI: 2.x

---

## 常见问题

### Q: 如何获取当前选中的完整节点对象?

A: 目前组件只通过 `filter-change` 事件返回 `path` 和 `filterPathIds`,如需完整节点对象,可在父组件中根据 ID 查找。

### Q: 可以禁用某些选项吗?

A: 当前版本不支持禁用选项,可通过过滤数据源实现。

### Q: 弹窗中的树默认展开所有节点吗?

A: 默认不展开,用户需手动展开。如需默认展开,可在弹窗打开时调用 `el-tree` 的 `expandAll` 方法。

### Q: 如何设置初始选中值?

A: 组件当前不支持通过 prop 设置初始选中值,建议在数据加载后通过编程方式模拟用户选择。

---

## 更新日志

### v2.0.0

- 支持动态配置层级数(`levels` prop)
- 新增 `levelNames` 自定义层级名称
- 新增 `full` 模式支持全量数据 + 搜索
- 完善无障碍支持

### v1.0.0

- 初始版本
- 支持 3 级平铺筛选
- 支持懒加载模式
