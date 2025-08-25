```
 <!-- 添加 -->
<dynamic-form-dialog
      :visible.sync="dialogFormVisible"
      :form-config="formConfig"
      :initial-data="editData"
      :title="formTitle"
      :initialData="dialogFormObj"
      :label-width="'162px'"
      :inlineTag="false"
      :dialogWidth="'700px'"
      :extra-button="{
        visible: true,
        text: '测试',
        callback: handleTestConnection,
        style: {
          margin: '0 14px 0 47px',
        },
      }"
      :cancel-button-style="{
        marginRight: '0px',
      }"
      :submit-button-style="{
        minWidth: '120px',
        marginLeft: '0px',
      }"
      @extra-button-click="handleDefaultTest"
      @submit="handleSubmit"
    />


formConfig: [
        {
          label: "任务名称",
          prop: "taskName",
          type: "input",
          placeholder: "请输入内容",
          maxlength: 30,
          rules: [{ required: true, message: "不能为空", trigger: "blur" }],
        },
        {
          label: "是否在图谱探索中展示",
          prop: "isShowInExplore",
          type: "radio",
          options: [
            { value: 1, label: "显示" },
            { value: 0, label: "不显示" },
          ],
          rules: [{ required: true, message: "请选择", trigger: "change" }],
        },

        {
          label: "选择模型",
          prop: "moduleId",
          type: "cascade-select",
          valueKey: "id",
          labelKey: "label",
          disabled: false, // 是否禁用
          rules: [{ required: true, message: "请选择", trigger: "change" }],
          fetchOptions: async () => {
            return new Promise((resolve, reject) => {
              RelationGraphService.getModelList({ pageSize: 1000, pageNo: 1 })
                .then((response) => {
                  const enhancedList = response.data.data.map((item) => ({
                    ...item,
                    label: `${item.schemaName || ""}${item.version ? " " + item.version : ""}`,
                  }));
                  resolve(enhancedList || []);
                  // return response.data.data || []; // 返回实际数据
                })
                .catch((error) => {
                  resolve([]); // 错误时返回空数组兜底（避免页面崩溃）
                  // console.error("接口请求失败:", error);
                  // return []; // 错误时返回空数组兜底（避免页面崩溃）
                });
            });
          },
          children: "productSubCategory",
        },

        {
          label: "选择图空间",
          prop: "spaceId",
          type: "cascade-select",
          placeholder: "请选择",
          valueKey: "id",
          labelKey: "spaceName",
          disabled: false, // 是否禁用
          pushSelectCurrent: true,
          selectCurrentStr: "--",
          options: [{ value: 1, label: "名称" }], // 需要动态传入实体列表
          rules: [{ required: true, message: "请选择", trigger: "change" }],
          onChange: (value, form) => {
            const sel = this.entityList.find((o) => o.id === value);
            if (sel) {
              form.headEntityKey = sel.data.entityKey;
            }
          },
          fetchOptions: () => {
            return new Promise((resolve, reject) => {
              RelationGraphService.getAvailableSpaceList({ pageSize: 1000, pageNo: 1 })
                .then((response) => {
                  resolve(response.data || []);
                })
                .catch((error) => {
                  resolve([]); // 错误时返回空数组兜底（避免页面崩溃）
                });
            });
          },
        },
        // {
        //   prop: "synchronizeTime",
        //   label: "同步时间",
        //   type: "radio",
        //   options: [
        //     { value: 10, label: "10分钟" },
        //     { value: 60, label: "1小时" },
        //     { value: 720, label: "12小时" },
        //     { value: 1440, label: "24小时" },
        //   ],
        //   rules: [{ required: true, message: "请选择", trigger: "change" }],
        // },
        {
          prop: "corn",
          label: "同步时间",
          type: "corn-select",

          rules: [{ required: true, message: "请选择", trigger: "change" }],
        },
      ],
      formTitle: "新增",

      dialogForm: [
        { label: "模型名称", value: "", prop: "schemaName" },
        { label: "英文名称", value: "", prop: "schemaKey" },
        { label: "版本", value: "", prop: "version" },
        // { label: "所属场景", value: "", prop: "belongingscene " },
        // { label: "所属业务", value: "", prop: "belongingbusiness" },
      ],
      dialogFormObj: {},
      rules: {
        schemaName: [{ required: true, message: "请输入模型名称", trigger: "blur" }],
        schemaKey: [{ required: true, message: "请输入英文名称", trigger: "blur" }],
        version: [{ required: true, message: "请输入版本", trigger: "blur" }],
      },

```
