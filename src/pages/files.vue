<template>
  <div class="h-flex" style="height: 100vh">
    <div class="pa-2">
      <div class="al-c pos-r">
        <div class="hover-1" @click="$router.back()">
          <i class="el-icon-back fz-18"></i>
        </div>

        <div class="ml-2">
          <el-button
            v-if="checkedList.length"
            type="danger"
            size="mini"
            @click="delFiles"
            >删除文件</el-button
          >
          <el-button
            v-else
            type="primary"
            size="mini"
            @click="showPop = true"
            :disabled="!client"
            >上传文件</el-button
          >
        </div>

        <div :class="asMobile ? 'ml-auto' : 'pos-center'">
          <el-radio-group v-model="showType" size="mini">
            <el-radio-button
              v-for="(it, i) in showTypeList"
              :label="it.value"
              :key="i"
              >{{ it.text }}</el-radio-button
            >
          </el-radio-group>
        </div>
        <!-- <div class="ml-auto">
          <el-button
            type="text"
            size="small"
            @click="showCheckbox = !showCheckbox"
            >{{ showCheckbox ? "取消" : "选择" }}</el-button
          >
        </div> -->
      </div>
      <div v-show="isInit">
        <el-drawer :visible.sync="showPop" :size="asMobile ? '100%' : '85%'">
          <div class="al-c" slot="title">
            <span>上传至</span>
            <span class="ml-2">{{ curPath }}</span>
          </div>
          <upload-panel
            :path="curPath"
            @close="showPop = false"
            @pick="showPop = true"
            @uploaded="onUploaded"
          />
        </el-drawer>
      </div>
    </div>

    <div class="pa-4" v-if="loading">
      <el-skeleton />
    </div>
    <div
      v-if="client"
      class="flex-1 bdt-1 d-flex ov-a file-list"
      ref="listWrap"
    >
      <div
        v-if="showType == 'tree'"
        class="shrink-0 h100p ov-a"
        style="min-width: 300px"
      >
        <file-tree
          ref="fileList"
          @file="onFile"
          :showCheckbox1="showCheckbox"
          v-model="checkedList"
        />
      </div>
      <div class="shrink-0" v-else-if="showType == 'cascader'">
        <file-cascader
          ref="fileList"
          @file="onFile"
          :showCheckbox="showCheckbox"
          v-model="checkedList"
        />
      </div>

      <div class="flex-1 shrink-0 bdl-1 h100p">
        <file-info class="h100p ovy-a" :file="curFile" @refresh="onRefresh" />
      </div>
    </div>
  </div>
</template>

<script>
import { mapState } from "vuex";

export default {
  computed: {
    ...mapState({
      asMobile: (s) => s.asMobile,
      curPath: (s) => s.curPath,
      client: (s) => s.client,
      bucketList: (s) => s.bucketList,
    }),
    query() {
      return this.$route.query;
    },
  },
  data() {
    return {
      showType: localStorage.showType || "tree",
      showTypeList: [
        {
          value: "tree",
          text: "树形",
        },
        {
          value: "cascader",
          text: "分栏",
        },
      ],
      loading: true,
      errMsg: "",
      isInit: false,
      showPop: true,
      curFile: null,
      showCheckbox: true,
      checkedList: [],
    };
  },
  watch: {
    showType(val) {
      localStorage.showType = val;
      this.curFile = null;
      this.checkedList = [];
    },
    checkedList(val, old) {
      if (val.length != old.length) console.log(val);
    },
  },
  mounted() {
    this.$nextTick(() => {
      this.showPop = false;
      setTimeout(() => {
        this.isInit = true;
      }, 500);
    });
    this.getClient();
  },
  destroyed() {
    this.$setState({
      client: null,
    });
  },
  methods: {
    async delFiles() {
      try {
        await this.$confirm(
          "暂未实现批量删除，可删除单个文件",
          "是否确认删除？"
        );
        console.log(this.checkedList);
        this.$message("未执行");
      } catch (error) {
        console.log(error);
      }
    },
    async getClient() {
      try {
        const { plat, bucket } = this.query;
        const row = this.bucketList.find(
          (it) => it.plat == plat && it.bucket == bucket
        );
        if (!row) throw new Error("未找到bucket信息");
        this.loading = true;
        this.$setState({
          curBucket: row,
          client: null,
        });
        // https://help.aliyun.com/document_detail/32077.htm?spm=a2c4g.11186623.0.0.65556cf0tSiy9K#section-zkq-3rq-dhb
        let opts = row;
        if (row.plat != "4everland") {
          const { data } = await this.$http.get(row.stsApi);
          const {
            AccessKeyId,
            AccessKeySecret,
            SecurityToken: stsToken,
          } = data;
          opts = {
            region: row.region,
            bucket: row.bucket,
            accessKeyId: AccessKeyId,
            accessKeySecret: AccessKeySecret,
            stsToken,
            // refreshSTSToken: () => {
            //   location.reload();
            // },
          };
        }
        const client = await this.$store.dispatch("s3/getClient", opts);
        this.$setState({
          client,
        });
      } catch (error) {
        console.log(error);
        this.errMsg = error.message;
        this.$message(this.errMsg);
      }
      this.loading = false;
    },
    onUploaded() {
      this.onRefresh();
    },
    onRefresh(name) {
      this.$refs.fileList.setRefresh(name);
    },
    onFile(file) {
      this.curFile = file;
      this.$nextTick(() => {
        if (this.showType == "cascader") this.autoScroll();
      });
    },
    autoScroll() {
      const wrap = this.$refs.listWrap;
      const toLeft = wrap.children[0].offsetWidth;
      // if (toLeft < wrap.offsetWidth) return console.log("no scroll");
      const curLeft = wrap.scrollLeft;
      const diff = toLeft - curLeft;
      const beginTime = Date.now();
      const rAF = window.requestAnimationFrame;
      const frameFunc = () => {
        let progress = (Date.now() - beginTime) / 300;
        if (progress > 1) progress = 1;
        const x = curLeft + progress * diff; //easeInOutCubic
        wrap.scrollTo(x, 0);
        if (progress < 1) {
          rAF(frameFunc);
        }
      };
      rAF(frameFunc);
    },
  },
};
</script>