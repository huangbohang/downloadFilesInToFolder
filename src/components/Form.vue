<template>
  <el-form
    ref="elform"
    class="form"
    :model="formData"
    label-position="top"
    :rules="rules"
  >
    <el-form-item label="选择数据表" size="large" prop="tableId">
      <el-select
        v-model="formData.tableId"
        placeholder="请选择数据表"
        style="width: 100%"
      >
        <el-option
          v-for="meta in tableMetaList"
          :key="meta.id"
          :label="meta.name"
          :value="meta.id"
        />
      </el-select>
    </el-form-item>
    <el-form-item label="请选择附件字段" size="large" prop="attachmentFileds">
      <el-select
        v-model="formData.attachmentFileds"
        placeholder="请选择附件字段"
        style="width: 100%"
      >
        <el-option
          v-for="meta in attachmentList"
          :key="meta.id"
          :label="meta.name"
          :value="meta.id"
        />
      </el-select>
    </el-form-item>
    <el-form-item label="请选择下载文件层级关系字段" size="large">
      <el-select
        v-model="formData.singleSelectKey"
        placeholder="请选择下载文件层级关系字段"
        style="width: 100%"
      >
        <el-option
          v-for="meta in singleSelectList"
          :key="meta.id"
          :label="meta.name"
          :value="meta.id"
        />
      </el-select>
    </el-form-item>
    <div class="btns">
      <!-- <el-button type="warning" plain size="large" @click="downFiles"
        >结构预览</el-button
      > -->
      <el-button
        type="primary"
        plain
        size="large"
        @click="submit"
        :loading="loading"
        >下载</el-button
      >
    </div>
    <div class="prompt">
      <p v-if="loading" v-for="(item, index) in loadingText" :key="index">
        {{ item }}
      </p>
    </div>
  </el-form>
</template>
<script>
import { bitable, FieldType } from "@lark-base-open/js-sdk";
import { ref, onMounted, reactive, watchEffect, toRefs, watch } from "vue";
import { ElButton, ElForm, ElFormItem, ElSelect, ElOption } from "element-plus";
import { saveAs } from "file-saver";
import JSZip from "jszip";
import { ElLoading } from "element-plus";

export default {
  components: {
    ElButton,
    ElForm,
    ElFormItem,
    ElSelect,
    ElOption,
  },
  setup() {
    let oTable = null;
    let recordList = null;
    const elform = ref(null);
    const datas = reactive({
      loadingText: [],
      loading: false,
      attachmentList: [],
      tableMetaList: [],
      singleSelectList: [],
      rules: {
        tableId: [
          {
            required: true,
            message: "请选择数据表",
            trigger: "change",
          },
        ],
        attachmentFileds: [
          {
            required: true,
            message: "请选择附件字段",
            trigger: "change",
          },
        ],
      },
      formData: { tableId: "", attachmentFileds: "", singleSelectKey: "" },
    });
    // 附件相关
    const getAttachmentDatas = async (record) => {
      const attachmentCells = await record.getCellByField(
        datas.formData.attachmentFileds
      );
      const cellValues = await attachmentCells.getValue();
      if (cellValues) {
        await getFilesAddress(
          cellValues,
          attachmentCells.field.id,
          attachmentCells.record.id
        );

        return cellValues;
      }
      return [];
    };
    // 文件夹相关
    const getSingleSelectDatas = async (record) => {
      const singleSelecCells = await record.getCellByField(
        datas.formData.singleSelectKey
      );
      const cellValue = await singleSelecCells.getValue();
      return cellValue || [];
    };
    const saveZip = async (tree) => {
      datas.loadingText.push("正在获取文件，请稍等...");

      const zip = new JSZip();

      for (const folderName in tree) {
        const folder = zip.folder(folderName);
        for (const file of tree[folderName]) {
          let data = await fetch(file.address);
          let res = await data.blob();
          let blod = new Blob([res]);
          folder.file(file.name, blod);
        }
      }

      zip.generateAsync({ type: "blob" }).then((content) => {
        saveAs(content, `压缩包.zip`);
      });
      datas.loadingText.push("文件下载完成...");

      setTimeout(() => {
        datas.loadingText = [];
        datas.loading = false;
      }, 3000);
    };
    const submit = async () => {
      if (!elform) return;
      await elform.value.validate((valid) => {
        if (valid) {
          downFiles();
        }
      });
    };
    const getFilesAddress = async (cellValues, fieldId, recordId) => {
      for (const cellValue of cellValues) {
        if (cellValue.token) {
          cellValue.address = (
            await oTable.getCellAttachmentUrls(
              [cellValue.token],
              fieldId,
              recordId
            )
          )[0];
        }
      }
    };

    const downFiles = async () => {
      // 结构树
      datas.loadingText.push("正在构建结构树...");
      datas.loading = true;
      const structureTree = {};

      for (const record of recordList) {
        const attachmentDatas = await getAttachmentDatas(record);
        const singleSelectData = await getSingleSelectDatas(record);
        const row = {
          files: attachmentDatas,
          folder: singleSelectData.text,
        };
        if (structureTree[row.folder]) {
          structureTree[row.folder].push(...row.files);
        } else {
          structureTree[row.folder] = [...row.files];
        }
      }
      saveZip(structureTree);
    };
    const previewStructure = () => {};
    watch(
      () => datas.formData.tableId,
      async (tableId) => {
        datas.formData.attachmentFileds = "";
        datas.formData.singleSelectKey = "";

        // 获取当前选择的数据表
        oTable = await bitable.base.getTableById(tableId);
        recordList = await oTable.getRecordList();
        let fieldMetaList = await oTable.getFieldMetaList();
        //  获取当前数据表中的单选字段
        datas.singleSelectList = fieldMetaList.filter(
          (e) => e.type === FieldType.SingleSelect
        );
        //  获取当前数据表中的附件字段列表
        datas.attachmentList = fieldMetaList.filter(
          (e) => e.type === FieldType.Attachment
        );
        setDefaultValue();
      }
    );
    const setDefaultValue = () => {
      if (datas.attachmentList.length) {
        datas.formData.attachmentFileds = datas.attachmentList[0].id;
      }
      if (datas.singleSelectList.length) {
        datas.formData.singleSelectKey = datas.singleSelectList[0].id;
      }
    };

    onMounted(async () => {
      // 获取激活的数据表
      const [tableList, selection] = await Promise.all([
        bitable.base.getTableMetaList(),
        bitable.base.getSelection(),
      ]);

      datas.formData.tableId = selection.tableId;
      datas.tableMetaList = tableList;
    });

    return {
      elform,
      ...toRefs(datas),
      previewStructure,
      submit,
    };
  },
};
</script>
<style scoped>
.form :deep(.el-form-item__label) {
  font-size: 16px;
  color: var(--el-text-color-primary);
  margin-bottom: 0;
}
.form :deep(.el-form-item__content) {
  font-size: 16px;
}
.btns {
  display: flex;
  justify-content: center;
  align-items: center;
  grid: 10px;
}
.btns .el-button {
  min-width: 40%;
  /* flex: 1; */
}
.prompt p {
  line-height: 20px;
  color: #333;
  font-size: 16px;
}
</style>
