<template>
  <!--
    FilterTreeSelector 组件（
    - 支持前 N 级平铺选择（radio），N 由 levels prop 指定
    - 第 N 级可打开"更多"弹窗选择更深层级（el-tree）
    - 支持两种数据模式（mode prop）：
        'lazy'：按需加载（使用 loadChildren）—— 默认（向后兼容）
        'full'：一次性全量数据（通过 treeData 提供），弹窗内可筛选
  -->
  <div class="filter-tree-selector flat">
    <!-- header：标题 + 面包屑 + 清除按钮 -->
    <div class="filter-header">
      <div class="filter-label">{{ filterTitle }}</div>
      <div class="breadcrumbs-wrap" role="region" aria-label="当前筛选路径">
        <el-breadcrumb separator="/">
          <!-- 面包屑项（点击回溯） -->
          <el-breadcrumb-item v-for="(item, idx) in breadcrumbs" :key="'crumb-' + idx + '-' + item.id">
            <a
              href="#"
              @click.prevent="onBreadcrumbClick(idx)"
              class="crumb-link"
              :aria-current="idx === breadcrumbs.length - 1 ? 'true' : null"
            >
              {{ item.label }}
            </a>
          </el-breadcrumb-item>

          <!-- 如果没有面包屑则展示"全部" -->
          <el-breadcrumb-item v-if="breadcrumbs.length === 0">
            <span class="crumb-empty">{{ defaultLabel }}</span>
          </el-breadcrumb-item>
        </el-breadcrumb>

        <!-- 清除按钮：快速重置筛选 -->
        <el-button
          size="mini"
          type="text"
          class="clear-btn"
          @click="handleClearClick"
          aria-label="清除筛选"
          v-if="breadcrumbs.length > 0"
          >清除</el-button
        >
      </div>
    </div>

    <!-- 各级平铺展示区（role=list，便于无障碍理解为列表） -->
    <div class="tiled-levels-container" role="list" aria-label="分类级别列表">
      <!-- filterLevels 中的每一数组项（最多展示前 levels 级） -->
      <div
        class="level-row"
        v-for="(items, idx) in filterLevels"
        :key="'level-row-' + idx"
        v-if="items.length > 0"
        role="listitem"
        tabindex="0"
        @keydown.enter.prevent="onRowActivate(idx)"
        @keydown.space.prevent="onRowActivate(idx)"
        :title="`第 ${idx + 1} 级${getLevelName(idx)}，按回车进入选项`"
      >
        <!-- 左侧：级别标签 -->
        <div class="level-title" aria-hidden="true">
          <span class="level-index">{{ idx + 1 }}</span>
          <span class="level-sub">{{ getLevelName(idx) }}</span>
        </div>

        <!-- 右侧：radio 列表区域与更多按钮 -->
        <div class="level-content">
          <div
            class="level-radio-group"
            :class="{ 'has-overlay': items.length > radiosVisibleLimit }"
            :style="{
              maxHeight: radiosVisibleLimit
                ? radiosVisibleHeight + 'px'
                : 'auto',
            }"
          >
            <!-- el-radio-group 增加 aria-label 以说明这一组 radio 的含义 -->
            <el-radio-group
              v-model="selectedIds[idx]"
              @change="(val) => onLevelSelect(idx + 1, val)"
              :aria-label="`第${idx + 1}级${getLevelName(idx)}选项`"
            >
              <!-- key/label 使用 getId/getLabel 统一访问器，支持 idField/labelField 自定义 -->
              <el-radio
                v-for="opt in items"
                :key="'level' + (idx + 1) + '-' + getId(opt)"
                :label="getId(opt)"
                class="level-radio"
                :title="getLabel(opt)"
              >
                <span class="radio-label-text">{{ getLabel(opt) }}</span>
              </el-radio>
            </el-radio-group>
          </div>

          <!-- 最后一级的"查看下级"按钮：只要最后一级已选就允许打开（也会根据 mode 决定是否能打开弹窗） -->
          <div
            class="more-action"
            v-if="
              idx === levels - 1 &&
              selectedNodes[idx] &&
              selectedNodes[idx].children &&
              selectedNodes[idx].children.length > 0
            "
          >
            <el-tooltip
              :content="`打开树形弹窗，选择${moreButtonText}`"
              placement="top"
            >
              <el-button
                size="mini"
                type="text"
                class="more-btn"
                @click="openMoreDialog"
                aria-haspopup="dialog"
                :aria-label="`查看${moreButtonText}`"
                >查看下级</el-button
              >
            </el-tooltip>
          </div>
        </div>
      </div>

      <!-- 新增：无分类时的提示（当整个组件没有任何分类时显示） -->
      <div
        v-if="!hasCategories"
        class="no-categories-wrap"
        aria-live="polite"
        style="padding: 0px 4px 4px 4px"
      >
        <!-- 使用 Element UI 的空状态组件展示友好提示 -->
        <div>暂无{{ getLevelName(0) || '分类' }}</div>
      </div>
    </div>

    <!-- 弹窗：更多子级 -->
    <el-dialog
      :title="`更多子级（${getDialogTitle()}）`"
      :visible.sync="moreDialogVisible"
      :width="moreDialogWidth"
      aria-label="更多子级对话框"
    >
      <!-- 当 mode === 'full' 时显示搜索输入（全量数据下启用筛选） -->
      <div class="search-input-wrapper" v-if="mode === 'full'">
        <el-input
          v-model="searchKeyword"
          :placeholder="`输入关键词搜索${getLevelName(levels) || '分类'}`"
          clearable
          size="small"
          prefix-icon="el-icon-search"
        ></el-input>
      </div>

      <div
        class="more-tree-wrapper"
        ref="moreTreeWrapper"
        v-if="moreDialogVisible"
      >
        <div class="more-tree-inner">
          <!-- el-tree：混合模式（lazy / full）
               - :lazy 由 mode 决定（mode === 'lazy' 时为 true）
               - :load 仅在 lazy 模式下会被调用（回调内已判断）
               - :data 在 full 模式下从 moreTreeData 取值（即 selectedNodes[levels-1].childrenField）
               - filter-node-method 委托 filterNode：当输入关键词时树会被过滤（full 模式有效）
          -->
          <el-tree
            ref="moreTree"
            :node-key="idField"
            :props="treeProps"
            :load="loadTreeNode"
            :lazy="mode === 'lazy'"
            :data="moreTreeData"
            :expand-on-click-node="false"
            :highlight-current="true"
            :filter-node-method="filterNode"
            @node-click="handleTreeNodeClick"
            :aria-label="`选择${moreButtonText}`"
          />
        </div>
      </div>

      <div slot="footer" class="dialog-footer">
        <el-button @click="cancelMoreSelection">取消</el-button>
        <el-button type="primary" @click="confirmMoreSelection">确定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
export default {
  name: "FilterTreeSelector",
  props: {
    // 新增：指定要展示的平铺层级数
    levels: {
      type: Number,
      default: 3,
      validator: (val) => val >= 1,
    },

    // 新增：各级名称，可以传入数组或对象
    // 数组格式：['产业', '板块', '产线'] 或 ['一级', '二级', '三级']
    // 对象格式：{1: '产业', 2: '板块', 3: '产线'}
    // 如果不传，则使用默认格式：'1级分类', '2级分类'等
    levelNames: {
      type: [Array, Object],
      default: () => ({}),
    },

    // 新增：未选择时的默认标签
    defaultLabel: {
      type: String,
      default: "全部",
    },

    // 支持两种工作模式：'lazy'（默认）或 'full'
    mode: {
      type: String,
      default: "lazy",
      validator: (val) => ["lazy", "full"].includes(val),
    },

    // 当 mode === 'full' 时，父组件可以直接传入完整树数据（顶层数组）
    treeData: {
      type: Array,
      default: () => [],
    },

    // loadChildren：异步函数，仅在 lazy 模式下使用（或父组件需要兼容）
    // 现在接收两个参数：parentId 和 level（当前层级）
    loadChildren: { type: Function, default: null },

    // 当父组件修改 resetKey 时触发组件重置（提供简单外部控制）
    resetKey: { type: [String, Number], default: null },

    // initialLevel1：可选的第一级初始数据（避免首次网络请求）
    initialLevel1: { type: Array, default: () => [] },

    // 字段名自定义
    idField: { type: String, default: "id" },
    labelField: { type: String, default: "label" },
    childrenField: { type: String, default: "children" },

    moreDialogWidth: { type: String, default: "800px" },

    // 当 radio 项数量超过这个阈值时，显示遮罩（提示可滚动）
    radiosVisibleLimit: { type: Number, default: 8 },

    // 折叠状态下 radio 区域的最大高度（px）
    radiosVisibleHeight: { type: Number, default: 100 },
    // 标题
    filterTitle: { type: String, default: "筛选" },
  },

  data() {
    // 根据 levels 动态生成初始数据
    return {
      // 前 N 级的候选项数组
      filterLevels: Array.from({ length: this.levels }, () => []),

      // 每级当前选中 id，使用数组索引
      selectedIds: Array.from({ length: this.levels }, () => null),

      // 每级当前选中的完整节点对象
      selectedNodes: Array.from({ length: this.levels }, () => null),

      // 面包屑路径（包含合并第 N+1 级及更深层级的结果）
      breadcrumbs: [],

      // 更多子级弹窗是否可见
      moreDialogVisible: false,

      // 弹窗中临时选择的树路径
      tempTreeSelectedPath: [],

      // full 模式下用于树筛选的关键词
      searchKeyword: "",
    };
  },

  computed: {
    // 计算更多按钮的文本
    moreButtonText() {
      const nextLevel = this.levels + 1;
      const levelName = this.getLevelName(this.levels) || `${nextLevel}级及更深层级`;
      return levelName;
    },
  },

  watch: {
    resetKey(newVal, oldVal) {
      if (newVal !== oldVal) this.resetFilters();
    },

    searchKeyword(val) {
      this.$nextTick(() => {
        if (
          this.$refs.moreTree &&
          typeof this.$refs.moreTree.filter === "function"
        ) {
          this.$refs.moreTree.filter(val);
        }
      });
    },

    // 监听 levels 变化，重新初始化数据结构
    levels: {
      immediate: true,
      handler(newLevels, oldLevels) {
        if (newLevels !== oldLevels) {
          this.resetFilters();
        }
      },
    },
  },

  mounted() {
    this.initFilters();
  },

  computed: {
    treeProps() {
      return {
        children: this.childrenField,
        label: this.labelField,
      };
    },

    // 在 full 模式下，弹窗要渲染的 tree data
    moreTreeData() {
      if (this.mode === "full" && this.selectedNodes[this.levels - 1]) {
        return (
          this.selectedNodes[this.levels - 1][this.childrenField] || []
        );
      }
      return [];
    },

    hasCategories() {
      if (this.mode === "full") {
        if (Array.isArray(this.treeData) && this.treeData.length > 0)
          return true;
      }
      return (
        this.filterLevels &&
        this.filterLevels.some((arr) => Array.isArray(arr) && arr.length > 0)
      );
    },
  },

  methods: {
    // 获取层级名称的方法
    getLevelName(levelIndex) {
      const level = levelIndex + 1; // 转换为1开始的索引

      if (Array.isArray(this.levelNames)) {
        // 如果是数组，直接按索引获取
        if (this.levelNames[levelIndex] !== undefined) {
          return this.levelNames[levelIndex];
        }
      } else if (typeof this.levelNames === 'object' && this.levelNames !== null) {
        // 如果是对象，按层级获取
        if (this.levelNames[level] !== undefined) {
          return this.levelNames[level];
        }
      }

      // 默认返回"x级分类"
      return `${level}级分类`;
    },

    getId(node) {
      if (!node) return null;
      return node[this.idField];
    },

    getLabel(node) {
      if (!node) return "";
      return node[this.labelField];
    },

    // 获取弹窗标题
    getDialogTitle() {
      const lastNode = this.selectedNodes[this.levels - 1];
      return lastNode ? this.getLabel(lastNode) : "更多子级";
    },

    resetFilters() {
      // 动态生成数据结构
      this.filterLevels = Array.from({ length: this.levels }, () => []);
      this.selectedIds = Array.from({ length: this.levels }, () => null);
      this.selectedNodes = Array.from({ length: this.levels }, () => null);
      this.breadcrumbs = [];
      this.moreDialogVisible = false;
      this.tempTreeSelectedPath = [];
      this.initFilters();
      this.$emit("filter-change", { path: [], filterPathIds: [] });
    },

    initFilters() {
      this.filterLevels = Array.from({ length: this.levels }, () => []);

      if (this.mode === "full") {
        this.$set(this.filterLevels, 0, this.treeData || []);
        return;
      }

      if (this.initialLevel1 && this.initialLevel1.length) {
        this.$set(this.filterLevels, 0, this.initialLevel1);
        return;
      }

      if (this.loadChildren) {
        // 修改：传入当前层级（1）作为第二个参数
        this.loadChildren(null, 1)
          .then((res) => {
            const data = res && res.data ? res.data : res;
            this.$set(this.filterLevels, 0, data || []);
          })
          .catch(() => {
            this.filterLevels = Array.from({ length: this.levels }, () => []);
          });
      } else {
        this.filterLevels = Array.from({ length: this.levels }, () => []);
      }
    },

    findNodeInLevel(level, id) {
      if (!id) return null;
      const arr = this.filterLevels[level - 1] || [];
      return arr.find((it) => this.getId(it) === id) || null;
    },

    onLevelSelect(level, id) {
      // 更新选中 id
      this.$set(this.selectedIds, level - 1, id);

      // 查找并设置对应节点对象
      const node = this.findNodeInLevel(level, id);
      this.$set(this.selectedNodes, level - 1, node);

      // 清空后续级别
      for (let i = level; i < this.levels; i++) {
        this.$set(this.selectedIds, i, null);
        this.$set(this.selectedNodes, i, null);
        this.$set(this.filterLevels, i, []);
      }

      // 生成面包屑（只包含前 N 级）
      const path = [];
      for (let i = 0; i < this.levels; i++) {
        if (this.selectedNodes[i]) {
          path.push({
            id: this.getId(this.selectedNodes[i]),
            label: this.getLabel(this.selectedNodes[i]),
          });
        }
      }
      this.breadcrumbs = path;

      // 如果不是最后一级，尝试加载下一层数据
      if (level < this.levels && node) {
        if (this.mode === "full") {
          const children = node[this.childrenField] || [];
          this.$set(this.filterLevels, level, children);
        } else {
          if (this.loadChildren) {
            // 修改：传入下一层级（level + 1）作为第二个参数
            this.loadChildren(this.getId(node), level + 1)
              .then((res) => {
                const data = res && res.data ? res.data : res;
                this.$set(this.filterLevels, level, data || []);
              })
              .catch(() => {
                this.$set(this.filterLevels, level, []);
              });
          } else {
            this.$set(this.filterLevels, level, []);
          }
        }
      }

      // 通知父组件当前筛选
      this.$emit("filter-change", {
        path: this.breadcrumbs.map((i) => ({ id: i.id, label: i.label })),
        filterPathIds: this.breadcrumbs.map((i) => i.id),
      });
    },

    openMoreDialog() {
      const lastLevelIndex = this.levels - 1;
      const hasChildrenInFullMode =
        this.mode === "full" &&
        this.selectedNodes[lastLevelIndex] &&
        Array.isArray(
          this.selectedNodes[lastLevelIndex][this.childrenField]
        ) &&
        this.selectedNodes[lastLevelIndex][this.childrenField].length > 0;
      const canOpenInLazyMode = this.mode === "lazy" && this.loadChildren;

      if (
        !this.selectedNodes[lastLevelIndex] ||
        !(hasChildrenInFullMode || canOpenInLazyMode)
      )
        return;

      this.moreDialogVisible = true;
      this.tempTreeSelectedPath = [];
      this.searchKeyword = "";
    },

    loadTreeNode(node, resolve) {
      if (this.mode === "full" || !this.loadChildren) {
        resolve([]);
        return;
      }

      const lastLevelIndex = this.levels - 1;
      const parentId =
        node.level === 0
          ? this.getId(this.selectedNodes[lastLevelIndex])
          : this.getId(node.data);

      // 修改：计算当前树节点的层级并传入
      // 树节点层级 = 平铺层级数 + 树节点层级
      const treeNodeLevel = this.levels + node.level;
      // 要加载的子节点层级是当前节点层级 + 1
      const childLevel = treeNodeLevel + 1;

      this.loadChildren(parentId, childLevel)
        .then((res) => {
          const data = res && res.data ? res.data : res;
          resolve(data || []);
        })
        .catch(() => {
          resolve([]);
        });
    },

    handleTreeNodeClick(nodeData, node) {
      const path = [];
      let cur = node;
      while (cur && cur.level > 0) {
        if (cur.data) {
          path.unshift({
            id: this.getId(cur.data),
            label: this.getLabel(cur.data),
          });
        }
        cur = cur.parent;
      }
      this.tempTreeSelectedPath = path;
    },

    cancelMoreSelection() {
      this.moreDialogVisible = false;
      this.tempTreeSelectedPath = [];
    },

    confirmMoreSelection() {
      if (
        !this.tempTreeSelectedPath ||
        this.tempTreeSelectedPath.length === 0
      ) {
        this.moreDialogVisible = false;
        return;
      }

      // basePath：前 N 级
      const basePath = [];
      for (let i = 0; i < this.levels; i++) {
        if (this.selectedNodes[i]) {
          basePath.push({
            id: this.getId(this.selectedNodes[i]),
            label: this.getLabel(this.selectedNodes[i]),
          });
        }
      }

      const fullPath = basePath.concat(this.tempTreeSelectedPath);
      this.breadcrumbs = fullPath;

      // 保持前 N 级 selectedIds 的一致性
      for (let i = 0; i < this.levels; i++) {
        this.$set(
          this.selectedIds,
          i,
          this.selectedNodes[i] ? this.getId(this.selectedNodes[i]) : null
        );
      }

      this.moreDialogVisible = false;
      const filterPathIds = fullPath.map((i) => i.id);
      this.$emit("filter-change", {
        path: fullPath.map((i) => ({ id: i.id, label: i.label })),
        filterPathIds,
      });

      this.tempTreeSelectedPath = [];
    },

    onBreadcrumbClick(index) {
      const kept = this.breadcrumbs.slice(0, index + 1);
      this.breadcrumbs = kept;
      const level = index + 1;

      // 清空后续级别的选中和选项
      for (let i = level; i < this.levels; i++) {
        this.$set(this.selectedIds, i, null);
        this.$set(this.selectedNodes, i, null);
        this.$set(this.filterLevels, i, []);
      }

      // 恢复 selectedIds
      for (let i = 0; i < kept.length; i++) {
        this.$set(this.selectedIds, i, kept[i] ? kept[i].id : null);
      }

      // 尝试从已有 filterLevels 恢复 selectedNodes
      for (let i = 0; i < this.levels; i++) {
        this.$set(
          this.selectedNodes,
          i,
          this.selectedIds[i] ? this.findNodeInLevel(i + 1, this.selectedIds[i]) : null
        );
      }

      // 若需要，重新加载当前最后项的下一层
      const last = kept[kept.length - 1];
      if (last && level < this.levels) {
        const lastNode = this.selectedNodes[level - 1];
        if (lastNode) {
          if (this.mode === "full") {
            const children = lastNode[this.childrenField] || [];
            this.$set(this.filterLevels, level, children);
          } else if (this.loadChildren) {
            // 修改：传入下一层级（level + 1）作为第二个参数
            this.loadChildren(lastNode[this.idField], level + 1)
              .then((res) => {
                const data = res && res.data ? res.data : res;
                this.$set(this.filterLevels, level, data || []);
              })
              .catch(() => {
                this.$set(this.filterLevels, level, []);
              });
          }
        }
      }

      this.$emit("filter-change", {
        path: this.breadcrumbs.map((i) => ({ id: i.id, label: i.label })),
        filterPathIds: this.breadcrumbs.map((i) => i.id),
      });
    },

    onRowActivate(idx) {
      const rows = this.$el.querySelectorAll(".level-row");
      const row = rows && rows[idx];
      if (!row) return;
      const input = row.querySelector("input[type='radio'], input");
      if (input) {
        input.focus();
      } else {
        const firstRadio = this.filterLevels[idx] && this.filterLevels[idx][0];
        if (firstRadio) {
          this.onLevelSelect(idx + 1, this.getId(firstRadio));
        }
      }
    },

    handleClearClick() {
      this.resetFilters();
      this.$emit("filter-cleared");
    },

    filterNode(value, data) {
      if (!value) return true;
      return (
        data[this.labelField] &&
        String(data[this.labelField]).indexOf(value) !== -1
      );
    },
  },
};
</script>

<style lang="scss" scoped>
.filter-tree-selector.flat {
  background: #ffffff;
  border-radius: 4px;
  padding: 8px;
  border: 1px solid #e5e6e7;
  font-family: "PingFang SC", "Helvetica Neue", Arial, sans-serif;
  color: #28323a;
}

.filter-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 6px;
}
.filter-label {
  font-weight: 600;
  color: #606266;
  font-size: 13px;
  min-width: 72px;
}
.breadcrumbs-wrap {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 8px;
}
.clear-btn {
  color: #909399;
  font-size: 12px;
}

.level-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 0;
  background: transparent;
  border-bottom: 1px solid #f1f3f4;
  font-size: 13px;
  outline: none;
}

.level-row:last-child {
  border-bottom: none;
}

.level-title {
  flex: 0 0 64px;
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12.5px;
  color: #394b59;
}
.level-index {
  font-weight: 700;
  font-size: 13px;
  color: #22303a;
}
.level-sub {
  font-size: 11.5px;
  color: #8a9aa6;
}

.level-content {
  flex: 1 1 auto;
  display: flex;
  align-items: center;
  gap: 8px;
}

.level-radio-group {
  flex: 1 1 auto;
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  position: relative;
  padding-right: 6px;
  gap: 4px;
}

.level-radio-group.has-overlay::after {
  content: "";
  position: absolute;
  right: 0;
  top: 0;
  bottom: 0;
  width: 18px;
  pointer-events: none;
  background: linear-gradient(
    90deg,
    rgba(255, 255, 255, 0),
    rgba(255, 255, 255, 1)
  );
}

.level-radio {
  display: inline-flex;
  align-items: center;
  margin: 4px 6px 4px 0 !important;
  padding: 4px 8px;
  border-radius: 999px;
  background: transparent;
  border: 1px solid transparent;
  transition: background-color 120ms linear, border-color 120ms linear;
  cursor: pointer;
  font-size: 12px;
  color: #606266;
}

::v-deep(.level-radio.is-checked),
::v-deep(.level-radio .el-radio__input.is-checked) {
  background: rgba(38, 127, 254, 0.08);
  border-color: rgba(38, 127, 254, 0.18);
  color: #267fee;
}

.level-radio:hover {
  background: rgba(16, 118, 215, 0.04);
}

::v-deep(.level-radio .el-radio__label) {
  padding: 0;
  margin-left: 6px;
}

.radio-label-text {
  display: inline-block;
  max-width: 180px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.more-action {
  flex: 0 0 auto;
  display: flex;
  align-items: center;
}
.more-btn {
  color: #267fee;
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 11px;
}

.search-input-wrapper {
  margin-bottom: 12px;
}

.no-categories-wrap {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  color: #5e6d82;
}
</style>
