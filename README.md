```
 <!-- 添加 -->
<dynamic-form-dialog
  :visible.sync="dialogFormVisible"
  :form-config="formConfig"
  :title="formTitle"
  :initialData="dialogFormObj"
  :label-width="'106px'"
  :dialogWidth="'800px'"
  submitButtonText="下一步"
  :submit-button-style="{
    minWidth: '120px',
  }"
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
          label: "选择模型",
          prop: "moduleId",
          type: "cascade-select",
          valueKey: "id",
          labelKey: "schemaName",
          rules: [{ required: true, message: "请选择", trigger: "change" }],
          fetchOptions: async () => {
            // console.log("chufa1111");

            return new Promise((resolve, reject) => {
              RelationGraphService.getModelList({ pageSize: 1000, pageNo: 1 })
                .then((response) => {
                  resolve(response.data.data || []);
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
        // {
        //   label: "选择版本",
        //   prop: "productSubCategory",
        //   type: "cascade-select",
        //   dependsOn: "productCategory",
        //   rules: [{ required: true, message: "请选择", trigger: "change" }],
        //   fetchOptions: (categoryId) => {
        //     return Promise.resolve([
        //       { value: "p3", label: "手机" },
        //       { value: "p4", label: "家具" },
        //     ]);
        //   },
        // },

        {
          label: "选择图空间",
          prop: "spaceId",
          type: "cascade-select",
          placeholder: "请选择",
          valueKey: "id",
          labelKey: "spaceName",
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
              RelationGraphService.getSpaceList({ pageSize: 1000, pageNo: 1 })
                .then((response) => {
                  resolve(response.data || []);
                })
                .catch((error) => {
                  resolve([]); // 错误时返回空数组兜底（避免页面崩溃）
                });
            });
          },
        },
        {
          prop: "synchronizeTime",
          label: "同步时间",
          type: "radio",
          options: [
            { value: 10, label: "10分钟" },
            { value: 60, label: "1小时" },
            { value: 720, label: "12小时" },
            { value: 1440, label: "24小时" },
          ],
          rules: [{ required: true, message: "请选择", trigger: "change" }],
        },
        {
          label: "备注",
          prop: "remarks",
          type: "textarea",
          placeholder: "请输入内容",
        },
      ],

```
