<style lang="scss">
.e-file-info {
  max-width: 500px;
  .media {
    min-width: 300px;
    min-height: 100px;
    img,
    video {
      display: block;
      max-height: 390px;
      max-width: 100%;
    }
    img {
      background: url(../../assets/bg-trans.svg);
      background-size: 30px;
    }
    .el-image:hover {
      box-shadow: 0 0 5px #9995;
    }
  }
  label {
    min-width: 80px;
  }
}
</style>

<template>
  <div class="e-file-info" v-if="file">
    <div class="pa-4">
      <div class="al-c f-center media" v-loading="loading">
        <div v-if="!src"></div>
        <el-image
          class="trans-200"
          @load="onImgLoad"
          :src="src"
          :preview-src-list="[src]"
          v-else-if="isImg"
          fit="contain"
        />
        <video
          ref="video"
          v-else-if="isVideo"
          :src="src"
          controls
          webkit-playsinline
          playsinline
          preload="metadata"
        />
        <audio ref="audio" v-else-if="isAudio" :src="src" controls />
        <div class="fz-12 bg-e pa-3" v-else>
          {{ info.name }}
        </div>
      </div>
      <div class="mt-5">
        <p class="al-c">
          <span class="fz-18">{{ info.name }}</span>
        </p>
        <div v-if="info.etag" class="mt-2">
          <el-link
            :href="src"
            type="primary"
            target="_blank"
            icon="el-icon-bottom"
            >下载</el-link
          >
          <el-dropdown @command="onCommand">
            <span
              ><el-link
                class="ml-6"
                type="text"
                :icon="actLoading ? 'el-icon-loading' : 'el-icon-more'"
              >
              </el-link
            ></span>
            <el-dropdown-menu slot="dropdown">
              <el-dropdown-item command="rename" icon="el-icon-edit"
                >重命名</el-dropdown-item
              >
              <el-dropdown-item command="delete" icon="el-icon-delete"
                >删除</el-dropdown-item
              >
              <!-- <el-dropdown-item icon="el-icon-document-copy"
                >复制链接</el-dropdown-item
              > -->
            </el-dropdown-menu>
          </el-dropdown>
        </div>
      </div>
      <div class="mt-6">
        <h3 class="fz-14 op-8">信息</h3>
        <div class="fz-13 mt-3">
          <div
            class="d-flex"
            :class="it.cls || 'mt-3'"
            v-for="(it, i) in infoList"
            :key="i"
          >
            <label class="gray">{{ it.label }}：</label>
            <span class="wb-all">{{ it.value }}</span>
            <i
              v-clipboard="it.value"
              @success="$message.success('已复制')"
              class="el-icon-document-copy color-1 pa-1 hover-1"
              v-if="it.paste"
            ></i>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { mapState } from "vuex";
import { calcGCD } from "../../utils/helper";

export default {
  props: {
    file: null,
  },
  data() {
    return {
      src: null,
      loadMeta: {},
      loading: false,
      actLoading: false,
    };
  },
  computed: {
    ...mapState({
      curBucket: (s) => s.curBucket,
    }),
    info() {
      if (!this.file) return {};
      let { name, url, size, type, lastModified, etag } = this.file;
      const key = name;
      let arr = name.split("/");
      if (arr.length > 1) {
        name = arr[arr.length - 1];
      }
      if (!type && this.src) {
        type = (this.src.split(";")[0] || "").split(":")[1] || "unknown";
      }
      let suffix = "";
      arr = name.split(".");
      if (arr.length > 1) {
        suffix = arr[arr.length - 1];
      }
      return {
        key,
        name,
        size,
        type,
        lastModified,
        suffix,
        etag,
        url,
        ...this.loadMeta,
      };
    },
    infoList() {
      const info = this.info;
      const list = [
        {
          label: "文件大小",
          value: this.$utils.getFileSize(info.size),
        },
        {
          label: "文件类型",
          value: !/\//.test(info.type) ? info.suffix : info.type,
        },
      ];
      if (info.width) {
        list.push({
          label: "宽高尺寸",
          value: `${info.width} × ${info.height} px`,
        });
        const gcd = calcGCD(info.width, info.height);
        list.push({
          label: "宽高比",
          value: `${info.width / gcd}:${info.height / gcd}`,
        });
      }
      if (info.duration) {
        list.push({
          label: "时长",
          value: info.duration.toFixed(1) + "s",
        });
      }
      if (info.etag) {
        list.push({
          label: "更新时间",
          value: info.lastModified.toDate().format(),
          cls: "mt-5",
        });
        list.push({
          label: "URL",
          value: info.url,
          paste: true,
        });
        list.push({
          label: "ETag",
          value: info.etag,
          paste: true,
        });
      }
      return list;
    },
    isOctet() {
      const { type } = this.info;
      return type == "application/octet-stream" || !/\//.test(type);
    },
    isImg() {
      const { type, suffix } = this.info;
      return (
        /image/.test(type) ||
        (this.isOctet && /ico|png|jpg|jpeg|gif|svg|webp/i.test(suffix))
      );
    },
    isAudio() {
      const { type, suffix } = this.info;
      return /audio/.test(type) || (this.isOctet && /mp3|wav/i.test(suffix));
    },
    isVideo() {
      const { type, suffix } = this.info;
      return (
        /video/.test(type) ||
        (this.isOctet && /mp4|avi|rmvb|flv|mov/i.test(suffix))
      );
    },
  },
  watch: {
    file: {
      handler() {
        this.onFile();
      },
      immediate: true,
    },
    async src() {
      setTimeout(() => {
        this.$emit("render");
      }, 20);
      if (this.isVideo || this.isAudio)
        this.$nextTick(() => {
          this.onMeta();
        });
    },
  },
  methods: {
    onCommand(val) {
      if (val == "rename") this.onRename();
      else if (val == "delete") this.onDelete();
    },
    async onRename() {
      const { name, key } = this.info;
      try {
        const { value } = await this.$prompt("请输入新文件名", "重命名", {
          inputValue: name,
          inputValidator(val) {
            if (!val) return "不能为空";
            if (/\//.test(val)) return "不能包含 /";
            if (val.length > 1017) return "长度不能超过1017";
          },
        });
        if (value == name) return;
        this.actLoading = true;
        const newKey = key.replace(new RegExp(name + "$"), value);
        await this.$store.dispatch("s3/copy", {
          Key: newKey,
          srcKey: key,
        });
        await this.$store.dispatch("s3/delete", key);
        this.$emit("refresh", newKey);
      } catch (error) {
        this.onError(error);
      }
      this.actLoading = false;
    },
    onError(error) {
      let msg = error.message;
      if (msg) this.$alert(msg);
    },
    async onDelete() {
      try {
        await this.$confirm("确定删除此文件？");
        this.actLoading = true;
        await this.$store.dispatch("s3/delete", this.info.key);
        this.$emit("refresh");
        this.$message.success("已删除");
      } catch (error) {
        this.onError(error);
      }
      this.actLoading = false;
    },
    onImgLoad(e) {
      const { width, height } = e.target || e.path[0];
      this.loadMeta = {
        width,
        height,
      };
    },
    onMeta() {
      const ref = this.isVideo ? "video" : "audio";
      const el = this.$refs[ref];
      if (!el) return;
      el.onloadedmetadata = (e) => {
        const { duration, videoWidth: width, videoHeight: height } = e.target;
        this.loadMeta = {
          duration,
          width,
          height,
        };
      };
    },
    onFile() {
      this.src = null;
      this.loadMeta = {};
      if (!this.file) return;
      if (this.file.size > 100 * Math.pow(1024, 2)) {
        return;
      }
      const { src, url, name } = this.file;
      // console.log(this.file);
      if (src) {
        this.src = src;
        this.url = src;
      } else if (url) {
        this.src = this.$store.dispatch("s3/signatureUrl", {
          name,
          opts: {
            expires: 3600,
          },
        });
      }
      if (!this.src) {
        this.loading = true;
        const reader = new FileReader();
        reader.readAsDataURL(this.file);
        reader.onload = (e) => {
          // console.log(e);
          this.loading = false;
          this.src = e.target.result;
        };
      }
    },
  },
};
</script>