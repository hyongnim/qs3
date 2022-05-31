<template>
  <div>
    <el-tooltip v-if="row.error" :content="row.error" placement="top">
      <el-tag type="danger" effect="plain" size="small">上传出错</el-tag>
    </el-tooltip>
    <el-tag
      v-else-if="row.progress"
      :type="row.progress == 100 ? 'success' : ''"
      size="small"
      >{{ row.progress }}%</el-tag
    >
    <el-tag v-else type="info" size="small">待上传</el-tag>
  </div>
</template>

<script>
export default {
  props: {
    row: Object,
  },
};
</script>