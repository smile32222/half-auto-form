<!-- 子组件：DynamicFormDialog.vue -->
<template>
  <el-dialog :visible.sync="dialogVisible" :title="formTitle" :width="dialogWidth" append-to-body @close="handleClose">
    <el-form ref="form" class="dynamic-form" :model="form" :rules="rules" :label-width="labelWidth">
      <!-- 动态生成表单项 -->
      <template v-for="(field, index) in formConfig">
        <el-form-item
          :key="index + 'formConfig'"
          v-if="shouldShowField(field)"
          :label="field.label"
          :prop="field.prop"
          :rules="field.rules"
          :label-width="field.labelWidth || undefined"
        >
          <!-- 输入框 -->
          <el-input
            v-if="field.type === 'input' || !field.type"
            v-model="form[field.prop]"
            :placeholder="field.placeholder"
            :maxlength="field.maxlength"
            :type="field.inputType"
            @change="field.onChange && field.onChange(form[field.prop])"
          />

          <!-- 选择器 -->
          <el-select
            v-else-if="field.type === 'select'"
            v-model="form[field.prop]"
            :clearable="field.clearable"
            :placeholder="field.placeholder"
            @change="handleSelectChange(field, $event)"
          >
            <el-option
              v-for="item in field.options"
              :key="item[field.valueKey || 'value']"
              :label="item[field.labelKey || 'label']"
              :value="item[field.valueKey || 'value']"
              :disabled="item.disabled || checkDisabled(field, item)"
            />
          </el-select>

          <!-- 级联下拉 -->
          <!--:loading="isCascadeLoading(field)" -->
          <el-select
            v-else-if="field.type === 'cascade-select' && (!field.dependsOn || form[field.dependsOn])"
            v-model="form[field.prop]"
            :placeholder="field.placeholder"
            :clearable="field.clearable"
            @change="(val) => handleCascadeChange(field, val)"
          >
            <el-option
              v-for="opt in getCascadeOptions(field)"
              :key="opt[field.valueKey || 'value']"
              :label="opt[field.labelKey || 'label']"
              :value="opt[field.valueKey || 'value']"
            />
          </el-select>

          <!-- 文本域 -->
          <el-input v-else-if="field.type === 'textarea'" type="textarea" v-model="form[field.prop]" :rows="field.rows || 3" :placeholder="field.placeholder" />

          <!-- 单选按钮（带边框样式） -->
          <el-radio-group class="radio-customize-check" v-else-if="field.type === 'radio'" v-model="form[field.prop]" @change="field.onChange && field.onChange(form[field.prop])">
            <el-radio v-for="item in field.options" :key="item.value" :label="item.value" border :disabled="item.disabled || checkDisabled(field, item)">
              {{ item.label }}
            </el-radio>
          </el-radio-group>

          <!-- 纯文字 -->
          <div v-else-if="field.type === 'text'" style="font-size: 16px; font-weight: 400; color: #000">{{ field.text }}</div>
        </el-form-item>
      </template>
    </el-form>

    <div slot="footer" class="flex justify-end dialog-footer">
      <el-button :style="cancelButtonStyle" @click="cancel">取 消</el-button>
      <el-button v-if="extraButton && extraButton.visible" type="primary" plain :style="extraButtonStyle" @click="handleExtraButton">
        {{ extraButton.text || "测试" }}
      </el-button>
      <el-button type="primary" :style="submitButtonStyle" @click="submitForm"> {{ submitButtonText }}</el-button>
    </div>
  </el-dialog>
</template>

<script>
export default {
  name: "DynamicFormDialog",
  props: {
    visible: {
      type: Boolean,
      default: false,
    },
    // 表单配置项
    formConfig: {
      type: Array,
      required: true,
      validator: (value) => value.every((item) => item.prop && item.label),
    },
    // 初始值（用于编辑）
    initialData: {
      type: Object,
      default: () => ({}),
    },
    // 表单标题
    title: {
      type: String,
      default: "表单",
    },

    // 表单标题
    dialogWidth: {
      type: String,
      default: "600px",
    },

    // labelWidth prop
    labelWidth: {
      type: [String, Number],
      default: "100px",
    },
    // 按钮样式配置
    cancelButtonStyle: {
      type: Object,
      default: () => ({
        // 默认取消按钮样式
        minWidth: "80px",
        marginRight: "12px",
      }),
    },
    submitButtonStyle: {
      type: Object,
      default: () => ({
        // 默认确定按钮样式
        minWidth: "80px",
      }),
    },
    // 确定按钮
    submitButtonText: {
      type: String,
      default: "确 定",
    },
    // 配置
    extraButton: {
      type: Object,
      default: () => ({
        visible: false,
        text: "测试",
        validate: true,
        callback: null,
        // 新增样式配置
        style: {
          minWidth: "100px",
          margin: "0 12px",
        },
      }),
    },
  },
  data() {
    return {
      form: {},
      rules: {},
      cascadeOptionsCache: new Map(), // 缓存级联选项
      cascadeLoading: new Set(), // 加载状态
    };
  },
  computed: {
    dialogVisible: {
      get() {
        return this.visible;
      },
      set(val) {
        this.$emit("update:visible", val);
      },
    },
    // 处理额外按钮样式
    extraButtonStyle() {
      return {
        ...this.extraButton.style,
      };
    },
    formTitle() {
      return this.title;
    },
  },
  watch: {
    initialData: {
      handler(newVal) {
        this.initForm(newVal);
      },
      immediate: true,
      deep: true,
    },
    formConfig: {
      handler() {
        this.initRules();
        if (this.initialData && Object.keys(this.initialData).length) {
          this.initForm(this.initialData);
        }
        // this.initForm(this.initialData);
      },
      immediate: true,
      deep: true,
    },
  },
  methods: {
    // 新增方法：判断字段是否显示
    shouldShowField(field) {
      if (typeof field.visibleOn === "function") {
        return field.visibleOn(this.form);
      }
      return !field.dependsOn || !!this.form[field.dependsOn];
    },
    // 新增方法：判断是否应禁用下级选择器
    shouldDisableCascade(field) {
      if (!field.dependsOn) return false;
      // 两种情况禁用：
      // 1. 根级字段值为空
      // 2. 根级选项正在加载中
      const parentField = this.formConfig.find((f) => f.prop === field.dependsOn);
      return !this.form[field.dependsOn] || (parentField && parentField.loading);
    },

    // 当用户选了根项时触发子级加载
    async handleCascadeChange(field, val) {
      field.onChange?.(val, this.form);
      if (!field.children) return;
      const child = this.formConfig.find((f) => f.prop === field.children);
      if (child && typeof child.fetchOptions === "function") {
        await this.loadCascadeOptions(child, val);
        this.form[child.prop] = ""; // 重置子级值
      }
    },

    // 拉取 options，无论根/子都写回 field.options，并缓存
    async loadCascadeOptions(field, parentValue) {
      const key = `${field.prop}_${parentValue}`;
      if (this.cascadeOptionsCache.has(key)) return;
      this.cascadeLoading.add(key);

      // console.log("loadCascadeOptions");

      const opts = await field.fetchOptions(parentValue);
      field.options = opts;
      this.cascadeOptionsCache.set(key, opts);

      this.$forceUpdate();
    },

    // 返回当前层级（根/子）要渲染的 options
    getCascadeOptions(field) {
      if (!field.dependsOn) {
        return field.options || [];
      }
      const key = `${field.prop}_${this.form[field.dependsOn]}`;
      return this.cascadeOptionsCache.get(key) || [];
    },

    // 初始化表单值 + 根级预加载 + 如果是编辑态也同时预加载子级
    async initForm(data) {

      this.form = this.formConfig.reduce((acc, f) => {
        acc[f.prop] = data[f.prop] ?? f.defaultValue ?? "";
        return acc;
      }, {});
      // 根级
      const roots = this.formConfig.filter((f) => f.type === "cascade-select" && !f.dependsOn && typeof f.fetchOptions === "function").map((f) => this.loadCascadeOptions(f, null));
      // 编辑态的子级
      const childs = this.formConfig
        .filter((f) => f.type === "cascade-select" && f.dependsOn && this.form[f.dependsOn])
        .map((f) => this.loadCascadeOptions(f, this.form[f.dependsOn]));
      await Promise.all([...roots, ...childs]);
    },

    // 初始化级联数据
    async initCascadeOptions() {
      for (const field of this.formConfig) {
        if (field.dependsOn && this.form[field.dependsOn]) {
          await this.loadCascadeOptions(field, this.form[field.dependsOn]);
        }
      }
    },

    // 初始化验证规则
    initRules() {
      const rules = {};
      this.formConfig.forEach((field) => {
        if (field.rules) {
          rules[field.prop] = field.rules;
        }
      });
      this.rules = rules;
    },

    // 处理选择器变化
    handleSelectChange(f, v) {
      f.onChange?.(v, this.form);
    },

    isCascadeLoading(field) {
      const key = `${field.prop}_${this.form[field.dependsOn] || null}`;
      return this.cascadeLoading.has(key);
    },

    // 检查选项是否禁用
    checkDisabled(field, item) {
      if (typeof field.disabled === "function") {
        return field.disabled(item, this.form);
      }
      return false;
    },

    submitForm() {
      this.$refs.form.validate((valid) => {
        if (valid) {
          this.$emit("submit", { ...this.form });
          // this.dialogVisible = false;
        }
      });
    },

    handleExtraButton() {
      const needValidate = this.extraButton.validate !== false;

      const validateHandler = (valid) => {
        if (!valid) return;

        if (typeof this.extraButton.callback === "function") {
          this.extraButton.callback(this.form);
        } else {
          this.$emit("extra-button-click", this.form);
        }
      };

      needValidate ? this.$refs.form.validate(validateHandler) : validateHandler(true);
    },

    cancel() {
      this.dialogVisible = false;
      this.$refs.form.resetFields();
    },

    handleClose() {
      this.cancel();
    },
  },
};
</script>

<style lang="scss">
.dynamic-form {
  .el-select {
    width: 100%;
  }
}

.radio-customize-check {
  .el-radio {
    width: 96px !important;
    text-align: center !important;
    margin-right: 11px !important;

    .el-radio__inner {
      display: none;
    }
    .el-radio__label {
      padding-left: 0px !important;
    }
  }
}
</style>
