<template>
  <div class="duty-record">
    <base-nav title="值班记录"></base-nav>

    <van-cell-group>
      <van-cell title="提交时间" :value="time"></van-cell>
      <van-cell title="值班人员" :value="form.dutyUser"></van-cell>
      <van-cell>
        <van-row slot="title" type="flex" justify="space-between">
          <van-col>值班记录</van-col>
          <van-col v-if="!id" style="color: #969799"
            >请填写纸质值班记录、并拍照存档</van-col
          >
        </van-row>
        <shot-photo
          :disabled="id"
          slot="label"
          v-model="photoList1"
        ></shot-photo>
      </van-cell>

      <van-switch-cell
        v-if="!id"
        v-model="form.hasMatter"
        title="发现问题"
      ></van-switch-cell>
      <!--      查看-->

      <div v-else>
        <van-cell title="发现问题">{{
          form.dutyStatus === 1 ? "否" : "是"
        }}</van-cell>

        <!--        <van-cell title="问题类型">-->
        <!--          <div :style="{ color: $store.state.getStatusColor[form.dutyStatus] }">-->
        <!--            {{ $store.state.getStatus[form.dutyStatus] }}-->
        <!--          </div>-->
        <!--        </van-cell>-->
        <describe-qusetion
          v-if="form.voice || form.content"
          :isEdit="id"
          :voice="form.voice"
          :content.sync="form.content"
        ></describe-qusetion>
        <van-cell v-if="photoList2.length" title="现场问题图片">
          <shot-photo
            slot="label"
            :disabled="id"
            v-model="photoList2"
          ></shot-photo>
        </van-cell>
      </div>

      <!--      添加、语音、备注-->
      <div v-show="form.hasMatter">
        <describe-qusetion
          v-model="question"
          :voice.sync="form.voice"
          :content.sync="form.content"
        ></describe-qusetion>

        <van-switch-cell
          v-model="form.isSolve"
          title="是否已解决"
        ></van-switch-cell>

        <van-cell title="附件现场问题图片">
          <shot-photo slot="label" v-model="photoList2"></shot-photo>
        </van-cell>
      </div>
    </van-cell-group>

    <base-button @click="submit" v-if="!id">提交</base-button>
  </div>
</template>

<script>
/**
 *  作者：0          时间：2019/7/9 13:45
 *  1,常量从js文件引入，不要定义魔术变量
 */
import ShotPhoto from "../../components/ShotPhoto";
import DescribeQusetion from "../../components/DescribeQusetion";
import { Toast } from "vant";
import Vue from "vue";
Vue.use(Toast);
export default {
  name: "DutyRecord",
  components: {
    ShotPhoto,
    DescribeQusetion
  },
  props: {},
  data() {
    return {
      time: "", // 提交时间
      question: {},
      photoList1: [],
      photoList2: [],
      form: {},
      status: 0,
      id: 0
    };
  },
  computed: {},
  watch: {},
  created() {
    let { id, status } = this.$route.params;
    this.time = this.$route.query.creationTime;
    this.status = status * 1;
    this.id = +id;
    +id
      ? this.getinfo()
      : (this.form.dutyUser = this.$store.state.userInfo.name); // 查看状态才进行查询
  },
  mounted() {},
  methods: {
    //  todo 获取详情
    getinfo() {
      this.$axios
        .get(this.$api.GET_DUTY_INFO, {
          params: { DutyId: this.id }
        })
        .then(res => {
          if (res.success) {
            this.form = res.result;
            this.form.content = res.result.problemRemark;
            //  值班记录照片
            if (this.form.dutyPhtosPath.length) {
              for (let i in this.form.dutyPhtosPath) {
                this.photoList1.push(
                  `${this.$url}${this.form.dutyPhtosPath[i]}`
                );
              }
            }
            //   问题照片
            if (this.form.problemPhtosPath.length) {
              for (let i in this.form.problemPhtosPath) {
                this.photoList2.push(
                  `${this.$url}${this.form.problemPhtosPath[i]}`
                );
              }
            }
            //  文字还是语音
            if (this.form.problemRemarkType === 2) {
              this.form.voice = `${this.$url}${this.form.problemRemark}`;
            } else {
              this.form.content = res.result.dutyRemark;
            }
            console.log(this.form);
          }
        });
    },
    //  todo 新增值班记录
    submit() {
      console.log(this.question);
      console.log("语音地址：", this.form);
      console.log("照片地址：", this.photoList1);
      console.log("照片地址：", this.photoList2);
      if (!this.photoList1.length) {
        this.$toast("值班记录不能为空！");
        return;
      }
      let that = this;
      let task = plus.uploader.createUpload(
        `http://fd.sctsjkj.com:5081${this.$api.ADD_DUTY_INFO}`,
        {
          method: "POST"
        },
        function(t, status) {
          console.log("请求成功后的返回数据：", t, status);
          // 上传完成
          if (status === 200) {
            Toast.clear();
            that.$toast.success("提交成功");
            that.$router.back();
          } else {
            that.$toast.fail("提交失败");
          }
        }
      );
      task.addFile(this.question.voice, { key: "RemarkVioce" });
      task.addData("FireUnitUserId", String(this.$store.state.userInfo.userId));
      task.addData("FireUnitId", String(this.$store.state.userInfo.fireUnitID));
      task.addData("CheckId", String(this.checkId));
      task.addData(
        "DutyStatus",
        this.form.hasMatter ? (this.form.isSolve ? "2" : "3") : "1"
      );
      task.addData("DutyRemark", this.question.content);
      task.addData("ProblemRemark", this.question.content);
      // 类型 语音、文字
      task.addData("ProblemRemarkType", this.question.voice ? "2" : "1");
      // 值班记录图片
      if (this.photoList1.length) {
        for (let i in this.photoList1) {
          task.addFile(this.photoList1[i], {
            key: `DutyPicture${Number(i) + 1}`
          });
        }
      }
      //现场问题图片
      if (this.photoList2.length) {
        for (let i in this.photoList2) {
          task.addFile(this.photoList2[i], {
            key: `ProblemPicture${Number(i) + 1}`
          });
        }
      }
      task.start();
      Toast.loading({
        duration: 0,
        mask: true,
        message: "提交中"
      });
    }
  }
};
</script>

<style lang="scss">
.duty-record {
  height: 100%;
  background-color: #fff;
  display: flex;
  flex-direction: column;
  & > :nth-child(2) {
    flex: 2 0 auto;
  }
}
</style>
