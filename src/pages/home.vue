<style lang="scss">
.add-bucket-form {
  .el-form-item__label {
    padding-right: 18px;
  }
}
</style>

<template>
  <div>
    <el-dialog
      title="添加Bucket"
      :visible.sync="showPop"
      append-to-body
      :width="asMobile ? '90%' : '600px'"
    >
      <div class="pa-1 add-bucket-form">
        <el-form :model="form" ref="form" size="small" label-width="100px">
          <el-form-item label="存储平台">
            <el-radio v-model="form.plat" label="4everland">4EVERLAND</el-radio>
            <el-radio v-model="form.plat" label="oss">OSS</el-radio>
          </el-form-item>
          <template v-if="form.plat == 'oss'">
            <el-form-item label="STS接口">
              <el-input v-model="form.stsApi" placeholder="https://" clearable>
                <!-- <template slot="prepend">https://</template> -->
              </el-input>
              <el-link
                href="https://www.tablofun.com/api/v1/user/test/role"
                type="primary"
                target="_blank"
                >接口示例</el-link
              >
            </el-form-item>
            <el-form-item label="Region">
              <el-input v-model="form.region" clearable></el-input>
            </el-form-item>
            <el-form-item label="Bucket" v-show="!checkBuckets.length">
              <el-input
                v-model="form.bucket"
                placeholder="多个Bucket可用逗号分隔"
                clearable
              ></el-input>
            </el-form-item>
          </template>
          <template v-else>
            <el-form-item label="API Key">
              <el-input v-model="form.accessKeyId" clearable></el-input>
              <el-link
                href="https://dashboard.4everland.org/#/settings?tab=auth_tokens&sub=bucket_auth_tokens"
                type="primary"
                target="_blank"
                >如何获取？</el-link
              >
            </el-form-item>
            <el-form-item label="API Secrect">
              <el-input v-model="form.secretAccessKey" clearable></el-input>
            </el-form-item>
          </template>
          <el-form-item label="Bucket" v-show="checkBuckets.length">
            <el-checkbox-group v-model="checkBuckets">
              <el-checkbox
                :label="name"
                v-for="name in apiBuckets"
                :key="name"
                :disabled="checkBuckets.length == 1"
              ></el-checkbox>
            </el-checkbox-group>
          </el-form-item>
          <el-form-item class="mt-7">
            <el-button
              size="small"
              :loading="stsChecking"
              @click="checkSts()"
              :disabled="posting"
              >{{ stsChecked ? "接口已校验" : "检测接口" }}</el-button
            >
            <el-button
              size="small"
              type="primary"
              :disabled="!stsChecked"
              :loading="posting"
              @click="onSubmit"
              >确认添加</el-button
            >
          </el-form-item>
        </el-form>
        <div class="mt-8"></div>
      </div>
    </el-dialog>

    <div class="pa-2 bdb-1 pos-r al-c">
      <div>
        <el-button
          icon="el-icon-plus"
          type="primary"
          size="mini"
          @click="showPop = true"
          >Bucket</el-button
        >
      </div>
      <div class="pos-center">
        <h2 class="fz-18">
          <i><span class="color-1">Q</span>uick S3</i>
        </h2>
      </div>
    </div>

    <div class="pa-4">
      <el-table size="small" :data="bucketList" @row-click="onRow">
        <el-table-column
          v-for="(it, i) in fields"
          :key="i"
          :prop="it.field"
          :label="it.label"
        >
          <template slot-scope="scope">
            <div v-if="it.field == 'act'">
              <el-button
                type="text"
                size="mini"
                @click.stop="onDel(scope.$index)"
                >删除</el-button
              >
            </div>
            <div v-else-if="it.field == 'accessKeyId'">
              <el-button
                :title="scope.row.stsApi"
                type="text"
                v-if="scope.row.stsApi"
                @click.stop="openLink(scope.row.stsApi)"
                size="mini"
                >{{
                  scope.row.stsApi.replace("https://", "").cutStr(10, 10)
                }}</el-button
              >
              <span v-else>
                {{ scope.row.accessKeyId }}
              </span>
            </div>
            <div v-else>
              {{ scope.row[it.field] }}
            </div>
          </template>
        </el-table-column>
      </el-table>
    </div>
  </div>
</template>

<script>
import { mapState } from "vuex";

const initForm = {
  plat: "4everland",
  stsApi: "",
  region: "", //oss-cn-hangzhou
  bucket: "",
  accessKeyId: "",
  secretAccessKey: "",
};
export default {
  computed: {
    ...mapState({
      asMobile: (s) => s.asMobile,
      bucketList: (s) => s.bucketList,
    }),
  },
  data() {
    return {
      showPop: false,
      form: initForm,
      checkBuckets: [],
      apiBuckets: [],
      stsChecking: false,
      stsChecked: false,
      stsAdding: false,
      posting: false,
      fields: [
        {
          field: "bucket",
          label: "Bucket",
        },
        {
          field: "plat",
          label: "存储平台",
        },
        {
          field: "accessKeyId",
          label: "API Key",
        },
        // {
        //   field: "stsApi",
        //   label: "STS接口",
        // },
        {
          field: "act",
          label: "操作",
        },
      ],
    };
  },
  watch: {
    bucketList(val) {
      localStorage.bucketList = JSON.stringify(val);
    },
    "form.plat"() {
      this.checkBuckets = [];
      this.stsChecked = false;
    },
  },
  methods: {
    openLink(link) {
      window.open(link);
    },
    async onDel(i) {
      console.log(i);
      try {
        await this.$confirm("是否确认删除？");
        this.bucketList.splice(i, 1);
      } catch (error) {
        //
      }
    },
    onRow(it) {
      // console.log(it);
      this.$router.push(`/files?plat=${it.plat}&bucket=${it.bucket}`);
    },
    async onSubmit() {
      const form = {
        ...this.form,
      };
      // console.log(form);
      try {
        this.posting = true;
        await this.checkSts();
        if (!this.stsChecked) throw new Error("检测未通过");
        const added = [],
          oldArr = [];
        for (const bucket of this.checkBuckets) {
          const old = this.bucketList.filter(
            (it) => it.plat == form.plat && it.bucket == bucket
          )[0];
          if (old) {
            oldArr.push(bucket);
            continue;
          }
          added.push(bucket);
          this.bucketList.push({
            ...this.s3Opts,
            bucket,
            createAt: Date.now(),
          });
        }
        if (added.length) {
          this.showPop = false;
          // this.$message.success("添加成功");
          this.form = {
            ...initForm,
          };
          this.checkBuckets = [];
        } else if (oldArr.length) {
          this.$message("已存在：" + oldArr.join(", "));
        }
      } catch (error) {
        console.log(error);
      }
      this.posting = false;
    },
    async checkSts() {
      let msg = "";
      try {
        const form = { ...this.form };
        const { plat, stsApi: api } = form;
        if (plat == "4everland") {
          delete form.stsApi;
          if (!form.accessKeyId) msg = "请输入accessKeyId";
          else if (!form.secretAccessKey) msg = "请输入secretAccessKey";
          if (msg) throw new Error(msg);
          if (!this.posting) this.stsChecking = true;
          this.s3Opts = {
            ...form,
            type: "s3",
          };
          const s3 = await this.$store.dispatch("s3/getClient", this.s3Opts);
          const data = await s3.listBuckets({});
          this.apiBuckets = data.Buckets.map((it) => {
            return it.Name;
          });
          if (
            !this.posting ||
            this.apiBuckets.filter((name) => this.checkBuckets.includes(name))
              .length == 0
          ) {
            this.checkBuckets = [...this.apiBuckets];
          }
          if (this.checkBuckets.length) {
            this.stsChecked = true;
          } else {
            throw new Error("API Key有效，但未找到Bucket");
          }
        } else {
          if (!api) msg = "请输入STS接口";
          else if (!/^http/.test(api)) msg = "请输入正确的STS接口链接";
          else if (!form.region) msg = "请输入Region";
          else if (!form.bucket) msg = "请输入Bucket";
          if (msg) throw new Error(msg);

          if (!this.posting) this.stsChecking = true;
          const { data } = await this.$http.get(api);
          console.log(data);
          const arr = [
            "AccessKeyId",
            "AccessKeySecret",
            "SecurityToken",
            "Expiration",
          ];
          let isOk = true;
          for (const key of arr) {
            if (!isOk) break;
            if (!data[key]) isOk = false;
          }
          if (!isOk) {
            throw new Error("接口需返回字段：" + arr.join(", "));
          }
          const buckets = form.bucket.split(/,|，/);
          const apiBuckets = [];
          const failBuckets = [];
          this.s3Opts = {
            ...form,
            accessKeyId: data.AccessKeyId,
          };
          for (let bucket of buckets) {
            bucket = bucket.trim();
            console.log(bucket);
            try {
              const s3 = await this.$store.dispatch("s3/getClient", {
                ...this.s3Opts,
                accessKeySecret: data.AccessKeySecret,
                bucket,
                stsToken: data.SecurityToken,
              });
              await s3.list({
                prefix: "",
                delimiter: "/",
                "max-keys": 5,
              });
              this.stsChecked = true;
              apiBuckets.push(bucket);
            } catch (error) {
              console.log(error);
              failBuckets.push(bucket);
            }
          }
          this.apiBuckets = apiBuckets;
          if (!apiBuckets.length) {
            throw new Error("请检查Region与Bucket是否匹配");
          } else {
            this.checkBuckets = [...apiBuckets];
            if (failBuckets.length && !this.posting)
              this.$message(
                "Bucket不存在或与Region不匹配：" + failBuckets.join(", ")
              );
          }
        }
      } catch (error) {
        this.stsChecked = false;
        if (error) {
          let msg = error.message || "未知错误";
          if (msg.length > 15) this.$alert(msg);
          else this.$message(msg);
        }
      }
      this.stsChecking = false;
    },
  },
};
</script>