<template>
  <el-container>
    <el-main v-show="!showResponse">
      <div class="block" v-if="pptUrl">
        <pptcontent :url="pptUrl" :teacher="true" />
        <el-pagination
          style="line-height: 50px"
          background
          small
          layout="prev, pager, next"
          @current-change="pageChange"
          :current-page="0"
          :page-count="slides.length"
        ></el-pagination>
        <el-button type="primary" class="counts">当前人数：{{studentCounts}}</el-button>
        <el-button type="primary" class="invite" @click="openShare">Share</el-button>
        <el-button type="primary" class="Presenting">Presenting</el-button>
        <el-button type="primary" class="Show" @click="showres">Show Responses</el-button>
        <el-button
          type="primary"
          class="noShow gray"
        >{{ getResponeCount() > 0 ? `${getResponeCount()} Responses` : `no Responses`}}</el-button>
      </div>
    </el-main>
    <el-main v-if="showResponse">
      <el-button type="primary" @click="hideRes">hide Responses</el-button>
      <template v-if="options&&options.length > 0">
        <teacherItem
          v-if="answerList.length > 0"
          :options="options"
          :title="title"
          :answerList="answerList"
          :pageId="getPid"
        />
      </template>
      <template v-else-if="textList.length>0">
        <teacherTextItem  :textList="textList" />
      </template>
      <div v-else>Waiting For Responses</div>
    </el-main>
  </el-container>
</template>
<style scoped>
.block {
  position: relative;
}
.invite {
  position: absolute;
  right: 10px;
  bottom: 20px;
}
.counts {
  position: absolute;
  left: 10px;
  bottom: 20px;
  background-color: transparent !important;
  border: none !important;
  color: #333;
}
.Show {
  position: absolute;
  right: 100px;
  bottom: 20px;
}
.noShow {
  position: absolute;
  right: 300px;
  bottom: 20px;
}
.Presenting {
  position: absolute;
  left: 100px;
  bottom: 20px;
  background-color: transparent !important;
  border: none !important;
  color: #333;
}
.gray {
  background-color: transparent !important;
  border: none !important;
  color: #333;
}
.modal {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 999;
  background-color: rgba(0, 0, 0, 0.2);
}
.shareBox {
  width: 300px;
  height: 200px;
  background-color: #fff;
  position: relative;
  margin: 0 auto;
}
</style>
<script>
import { MessageBox } from "element-ui";
import copy from "copy-to-clipboard";
import pptcontent from "../components/pptcontent";
import { getAllPPTS } from "../model/index";
import { showLoading, hideLoading, showToast } from "../utils/loading";
import teacherItem from "../components/teacherItem";
import teacherTextItem from "../components/teacherTextItem";
import { createSo } from "../socket/socket.teacher";
import { SocketEventsEnum } from "../socket/socketEvents";
import {
  getTeacherUid,
  setStundentUidAndName,
  setTeacherUid,
  getStundentUidAndName
} from "../utils/user";
import {
  saveTeacherAlist,
  saveTeacherDatalist,
  getTeacherAlist,
  getTeacherDatalist
} from "../utils/store";
import { generateUuid } from "@/utils/help";
import { createLogger } from "vuex";

export default {
  data() {
    return {
      showResponse: false, // 默认不展示老师的回答
      studentCounts: 0,
      pptUrl: null,
      title: "",
      options: [],
      slides: [],
      current: 0,
      slide_id: 0,
      currentSo: null,
      answerList: [], // 用户回答的内容
      textList: [], //用户输入的内容
      uid: getTeacherUid(), // uid
      current_user_id: "",
      type:"",
    };
  },
  mounted() {
    if (this.uid === null || this.uid === undefined) {
      this.uid = generateUuid("t_", 16);
      setTeacherUid(this.uid);
      this.joinRoom();
    } else {
      this.joinRoom();
    }
    this.openShare();
  },
  computed: {
    getPid() {
      return this.slides[this.current].page_id;
    },
    getUserName() {
      return getStundentUidAndName(this.current_user_id);
    }
  },
  components: {
    pptcontent,
    teacherItem,
    teacherTextItem
  },
  beforeRouteEnter(to, from, next) {
    next(vm => {
      vm.slide_id = to.query.slide_id;
      vm.getAllSlides();
    });
  },
  methods: {
    setCurrentUserId(user_id) {
      this.current_user_id = user_di;
    },
    getResponeCount(){
      console.log("getResponeCount=="+this.type)
      if(this.type == 'choice'){
        return this.answerList.length
      }else if(this.type == SocketEventsEnum.TEXT_INPUT||this.type==SocketEventsEnum.NUMBER_INPUT){
        console.log(this.textList.length)
        return this.textList.length;
      }else{
        return 0;
      }
    },
    getAllSlides() {
      showLoading();
      getAllPPTS(this.slide_id).then(list => {
        console.log(list);
        // this.contentUrl = d;
        // hideLoading()
        this.slides = list;
        this.getCurrentPPT();
        this.getItemData();
        hideLoading();
      });
    },
    getItemData() {
      this.options = [];
      this.$nextTick(() => {
        const choice = this.slides[this.current].items;
        if (choice && choice.data) {
          const { title, options } = choice.data;
          this.title = title;
          this.type = choice.type;
          console.log("item type =="+ this.type)
          this.options = options;
          this.textList = getTeacherDatalist(this.getPid,this.type)
          this.answerList = getTeacherAlist(this.getPid);
        }
      });
    },
    getCurrentPPT() {
      this.pptUrl = this.slides[this.current].thumbnail_url;
    },
    pageChange(value) {
      this.current = value - 1;
      this.getCurrentPPT();
      this.options = [];
      this.answerList = getTeacherAlist(this.getPid);
      this.textList = getTeacherDatalist(
        this.getPid,
        this.type
      );
      this.getItemData();
      // 换页命令
      // '{"type":"change_page", "params": {"page": 3}}'
      this.emitSo(
        `{"room":"${this.slide_id}", "type": "${SocketEventsEnum.GO_PAGE}", "params": {"page": "${this.current}"}}`
      );
    },
    copyUrl() {
      copy(
        `${location.href.replace(/teacher/, "students")}&page=${this.current}`
      );
      showToast("copy link success");
    },
    joinRoom() {
      this.currentSo = createSo(this.slide_id, this.uid, this.msgListener);
    },
    msgListener(d = {}) {
      // answer: "Lily"
      // item_id: "item_1"
      // page_id: "page_1"
      // room: "1KxKT-_j8Z1L4ag4waifI9hnDRm0C9yNnFt7VKwVVqCg"
      // user_id: "slidec3dcef92c1cf458c"
      console.log(d);
      if (d.type === SocketEventsEnum.ANSWER_QUESTION) {
        const { answer, page_id, room, user_id, action_type, user_name } = d;
        const currentPageId = this.slides[this.current].page_id;
        if (page_id === currentPageId && room === this.slide_id) {
          const filterData = this.answerList.filter(
            item => item.user_id !== user_id
          );
          filterData.push({
            user_id,
            answer
          });
          this.answerList = filterData;
          console.log(this.answerList);
          saveTeacherAlist(currentPageId, filterData);
          this.$forceUpdate();
        }
      } else if (d.type === SocketEventsEnum.STUDENTS_COUNTS) {
        this.studentCounts = d.student_count;
      } else if (d.type === SocketEventsEnum.RENAME) {
        const { user_id, user_name_new } = d;
        setStundentUidAndName(user_id, user_name_new);
        this.user_name = user_name_new;
        for (let i = 0; i < this.textList.length; i++) {
          if (user_id === this.textList[i].user_id) {
            let newVaule = this.textList[i];
            newVaule.user_name = user_name_new;
            //   Vue.set(this.textList, i, newValue);
          }
        }
      } else if (d.type == SocketEventsEnum.TEXT_INPUT||d.type === SocketEventsEnum.NUMBER_INPUT) {
        //接收到text input或者number input的值
        const {
          content,
          page_id,
          room,
          user_id,
          action_type,
          user_name,
          item_id
        } = d;
        const currentPageId = this.slides[this.current].page_id;
        console.log("currentPageId=="+currentPageId+" cuttentType =="+this.type)
        if (page_id === currentPageId && room === this.slide_id) {
          let textList = this.textList;
          if (!textList || textList.length == 0) {
            textList.push({
              content,
              user_id,
              user_name,
              item_id
            });
          } else {
            let found = false;
            for (let i = 0; i < textList.length; i++) {
              if (
                textList[i].user_id === user_id &&
                textList[i].item_id === item_id
              ) {
                found = true;
                textList[i].content = content;
                textList[i].user_name = user_name;
              }
            }
            if (!found) {
              textList.push({
                content,
                user_id,
                user_name,
                item_id
              });
            }
          }
          this.user_name = user_name;
          this.textList = textList;
          saveTeacherDatalist(
            currentPageId,
            textList,
            d.type
          );
          this.$forceUpdate();
        }
      }
    },
    // '{"type":"change_page", "params": {"page": 3}}'
    emitSo(message) {
      if (this.currentSo) {
        // this.currentSo.emit('control', JSON.stringify(data));
        this.currentSo.emit("control", message);
      }
    },
    // enterUid() {
    //   MessageBox.prompt('建议请输入手机号', '为了更好的演示体验，记录当前用户选中的内容信息，仅做demo参考', {
    //     confirmButtonText: '确定',
    //     showCancelButton: false,
    //     showClose: false,
    //     inputPattern: /[0-9]/,
    //     inputErrorMessage: '至少输入0-9的任意数字'
    //   }).then(({ value }) => {
    //     console.log(value)
    //     setTeacherUid(setUid)
    //     this.uid = value
    //     this.joinRoom()
    //   }).catch(() => {

    //   });
    // }
    openShare() {
      const url = `${location.href.replace(/teacher/, "students")}&page=${
        this.current
      }`;
      MessageBox.confirm(url, "Share this link with your students", {
        distinguishCancelAndClose: true,
        confirmButtonText: "copy",
        cancelButtonText: "Enter classroom"
      })
        .then(() => {
          this.copyUrl();
        })
        .catch(action => {});
    },
    showres() {
      this.showResponse = true;
    },
    hideRes() {
      this.showResponse = false;
    }
  }
};
</script>