<template>
  <div id="changeContab">
    <div class="quick-select">
      <el-button size="small" type="text" @click="setQuick(1)">每天1点</el-button>
      <el-button size="small" type="text" @click="setQuick(2)">每天2点</el-button>
      <el-button size="small" type="text" @click="setQuick(3)">每天3点</el-button>
      <el-button size="small" type="text" @click="setQuick(4)">每天4点</el-button>
    </div>

    <el-tabs type="border-card" v-model="activeTab">
      <el-tab-pane label="小时" name="hour">
        <div class="tabBody">
          <el-row>
            <el-radio v-model="hour.cronEvery" :label="1">每小时</el-radio>
          </el-row>
          <el-row>
            <el-radio v-model="hour.cronEvery" :label="2">
              从
              <el-input-number size="small" v-model.number="hour.incrementStart" :min="0" :max="23"></el-input-number>
              点开始，每隔
              <el-input-number size="small" v-model.number="hour.incrementIncrement" :min="1" :max="23"></el-input-number>
              小时
            </el-radio>
          </el-row>
          <el-row>
            <el-radio v-model="hour.cronEvery" :label="3" class="long">
              指定时间
              <el-select size="small" multiple v-model="hour.specificHours" style="width: 100px">
                <el-option v-for="h in 24" :key="h" :label="h - 1 + '点'" :value="h - 1"></el-option>
              </el-select>
            </el-radio>
          </el-row>
        </div>
      </el-tab-pane>

      <el-tab-pane label="天" name="day">
        <div class="tabBody">
          <el-row>
            <el-radio v-model="day.cronEvery" :label="1">每天</el-radio>
          </el-row>
          <el-row>
            <el-radio v-model="day.cronEvery" :label="2">
              从
              <el-input-number size="small" v-model.number="day.incrementStart" :min="1" :max="31"></el-input-number>
              号开始，每隔
              <el-input-number size="small" v-model.number="day.incrementIncrement" :min="1" :max="31"></el-input-number>
              天
            </el-radio>
          </el-row>
        </div>
      </el-tab-pane>
    </el-tabs>

    <div class="bottom">
      <div class="value">{{ cronDescription }}</div>
      {{ cron }}
    </div>
  </div>
</template>
<script>
export default {
  name: "vueCron",
  props: { value: { type: String, default: "" } },
  data() {
    return {
      activeTab: "hour",
      second: { value: 0 },
      minute: { value: 0 },
      hour: { cronEvery: 1, incrementStart: 0, incrementIncrement: 1, specificHours: [] },
      day: { cronEvery: 1, incrementStart: 1, incrementIncrement: 1 },
      month: { value: "*" },
      // year 字段在你的逻辑中并未使用，可以移除或保留
    };
  },
  mounted() {
    // 组件加载时，如果外部没有传入 value，则发送初始化的默认 cron
    if (this.value === null || this.value === undefined || this.value === "") {
      this.$emit("input", this.cron);
    }
  },
  methods: {
    setQuick(h) {
      this.day.cronEvery = 1;
      this.hour.cronEvery = 3;
      this.hour.specificHours = [h];
    },
    // 【已移除】setCheck 方法是多余的，它的功能被合并到 watch 中
  },

  // 【已移除】mounted 钩子也是多余的，因为 watch 的 immediate: true 会在初始化时执行
  // mounted() {
  //   this.setCheck(this.value);
  // },

  computed: {
    hoursText() {
      if (this.hour.cronEvery === 1) return "*";
      if (this.hour.cronEvery === 2) return `${this.hour.incrementStart}/${this.hour.incrementIncrement}`;
      if (this.hour.cronEvery === 3 && this.hour.specificHours.length) return this.hour.specificHours.join(",");
      return "*";
    },
    daysText() {
      return this.day.cronEvery === 1 ? "*" : `${this.day.incrementStart}/${this.day.incrementIncrement}`;
    },
    cron() {
      // 生成的cron表达式是6个字段
      return [this.second.value, this.minute.value, this.hoursText, this.daysText, this.month.value, '?'].join(" ");
    },
    cronDescription() {
      const parts = [];
      if (this.day.cronEvery === 1) parts.push("每天");
      else parts.push(`从${this.day.incrementStart}号开始，每隔${this.day.incrementIncrement}天`);

      if (this.hour.cronEvery === 1) parts.push("每小时");
      else if (this.hour.cronEvery === 2) parts.push(`从${this.hour.incrementStart}点开始，每隔${this.hour.incrementIncrement}小时`);
      else if (this.hour.cronEvery === 3 && this.hour.specificHours.length > 0)
        parts.push(`在${this.hour.specificHours.join(",")}点`);

      return `在 ${parts.join("，")} 执行`;
    },
  },
  watch: {
    "hour.cronEvery"(newVal) {
      if (newVal === 3 && this.hour.specificHours.length === 0) {
        this.hour.specificHours = [0];
      }
    },
    "hour.specificHours"(newVal, oldVal) {
      if (this.hour.cronEvery === 3 && newVal.length === 0 && oldVal.length > 0) {
        this.hour.specificHours = oldVal;
      }
    },
    cron(val) {
      this.$emit("input", val);
    },
    // 【核心修改】将原有的 value 监听器修改如下
    value: {
      immediate: true, // 【关键】添加 immediate: true，组件初始化时立即执行一次
      handler(newVal) {
        if (!newVal || newVal === this.cron) {
          return; // 防止空值和“回声”更新
        }

        const parts = newVal.split(" ");
        // 【关键】修正长度检查，应该检查6个字段
        if (parts.length !== 6) {
          console.error("传入的cron表达式格式不正确，需要6个字段。当前值:", newVal);
          return;
        }

        // 【关键】按6个字段进行解构
        const [s, m, h, d, mon, dow] = parts;

        // --- 小时字段解析 ---
        if (h === "*") {
          this.hour.cronEvery = 1;
        } else if (h.includes("/")) {
          const [start, inc] = h.split("/").map(Number);
          this.hour.cronEvery = 2;
          this.hour.incrementStart = isNaN(start) ? 0 : start;
          this.hour.incrementIncrement = isNaN(inc) ? 1 : inc;
        } else {
          this.hour.cronEvery = 3;
          this.hour.specificHours = h.split(",").map(Number).filter(num => !isNaN(num));
        }

        // --- 天字段解析 ---
        if (d === "*") {
          this.day.cronEvery = 1;
        } else if (d.includes("/")) {
          const [start, inc] = d.split("/").map(Number);
          this.day.cronEvery = 2;
          this.day.incrementStart = isNaN(start) ? 1 : start;
          this.day.incrementIncrement = isNaN(inc) ? 1 : inc;
        }
      },
    },
  },
};
</script>

<style lang="less">
#changeContab {
  position: relative;
  .tabBody {
    .el-row {
      margin: 10px 0;
    }
  }
  .bottom {
    width: 100%;
    text-align: left;
    margin-top: 5px;
    .value {
      font-size: 14px;
    }
  }
  .quick-select {
    display: flex;
    align-items: center;
    position: absolute;
    right: 8px;
    z-index: 1;
    right: 12px;
    top: 4px;

    .el-button {
      margin-left: 13px;
    }

    .el-button--small {
      font-size: 14px;
    }
  }
}
</style>
