<template>
  <div class="page">
    <header class="page-header">
      <div class="page-title">{{dataObj.title}}</div>
      <div class="page-created_at">
        <span>{{dataObj.dateTime}}</span>
        <span v-if="status === '进行中'" class="green">{{status}}</span>
        <span v-else-if="status === '已结束'" class="end">{{status}}</span>
      </div>
    </header>
    <div class="page-content">
      <img :src="dataObj.imgSrc" alt="">
      <p>{{dataObj.desc}}</p>
    </div>
    <div class="page-comments">
      <div class="page-comments-divider"></div>
      <!--      <div class="page-comments-header">-->
      <!--        <span v-if="!this.commentNum">评论</span>-->
      <!--        <span v-else>评论（{{commentNum}}）</span>-->
      <!--        <a @click="link()">写短评</a>-->
      <!--      </div>-->
      <div class="no-page-comments" v-show="showComments">
        <span>这篇文章还没有评论</span>
      </div>

      <div class="page-comments-list" v-show="!showComments">
        <div :key="item.id" class="page-comment-item" v-for="item in list">
          <div class="comment-author-header">
            <img :src="item.headimgurl" alt="author-header" />
          </div>
          <div class="comment-details">
            <div class="comment-author-name">{{item.author.name}}</div>
            <div class="comment-body">{{item.body}}</div>
          </div>
        </div>
      </div>
      <div id="writeRemote" v-if="status === '进行中'">
        <van-cell title="写评论..." icon="chat-o" @click="showWritePop()"></van-cell>
        <van-popup v-model="showPop" closeable round position="bottom">
          <div class="popTitle">写评论</div>
          <div class="commentSection">
              <textarea
                autofocus="autofocus"
                class="commentInput"
                placeholder="请输入..."
                rows="10"
                v-model="message"
              />
          </div>
          <div class="commentSend" @click="commentSend()">提交</div>
        </van-popup>
      </div>
    </div>
    <van-popup
      :style="{ height: '200px',width:'70%' }"
      round
      close-icon="none"
      closeable
      v-model="show"
    >
      <div class="popup">
        <h1 class="popup_h1">提示</h1>
        <p class="popup_title">需要实名认证才能继续，是否前往登录？</p>
        <div class="popup_botton">
          <div class="noLogin" @click="show=false">取消</div>
          <a class="goLogin" href="http://wsq.gxzzzx.cn/mobile/wsq/login">确定</a>
        </div>
      </div>
    </van-popup>
  </div>
</template>

<script>
import api from "@/api/api";

export default {
  data() {
    return {
      pageId: '',
      activityId: '',
      status: '',
      dataObj: "",
      userId: "",
      list: [],
      showComments: true,
      commentNum: 0,
      show: false,
      showPop: false,
      message: "",
    };
  },
  mounted() {
    this.pageId = parseInt(this.$route.query.tableId);
    this.activityId = this.$route.query.type
    this.status = this.$route.query.status
    let keyword = "";
    if (this.$route.query.mobile) {
      keyword = this.$route.query.mobile;
    }
    api.getUserAPI(keyword).then((res) => {
      this.userId = res.data[0].id;
      api.getCommentsAPI(this.userId, this.pageId).then((res) => {
        let len = res.data.length;
        if (len) {
          this.commentNum = len;
          this.showComments = false;
        }
        for (let i = 0; i < len; i++) {
          res.data[i].headimgurl = res.data[i].author.headimgurl + "/64";
        }
        this.list = res.data;
      });
    });
    api.getActivityAPILyp(this.activityId).then((res) => {
      res.data.forEach((ele) => {
        if (ele.id === this.pageId) {
          this.dataObj = ele;
          this.dataObj.dateTime = this.dateFormat("YYYY-mm-dd", this.dataObj.created_at);
          console.log(this.dataObj)
          this.dataObj.imgSrc = this.dataObj.cover
          this.dataObj.desc = this.toText(this.dataObj.html_body)
          document.title = this.dataObj.title;
          window.desc = this.dataObj.title;
        }
      });
    });
  },
  methods: {
    showWritePop() {
      this.showPop = true;
    },
    commentSend() {
      if (this.message) {
        let data = {
          comment: {
            body: this.message,
          },
          user_id: this.userId,
          cms_page_id: this.pageId,
        };
        api.postCommentsAPI(data).then((res) => {
          if (res.status === 200) {
            this.$toast.success("评论成功,积分+10");
            setTimeout(() => {
              // 延迟跳转
              this.$router.go(-1);
            }, 1500);
          }
        });
      } else {
        this.$toast('评论不能为空')
      }
    },
    //  重构时间（时间格式，时间）
    dateFormat(fmt, date) {
      date = date ? new Date(date) : new Date();
      var ret;
      const opt = {
        "Y+": date.getFullYear().toString(), // 年
        "m+": (date.getMonth() + 1).toString(), // 月
        "d+": date.getDate().toString(), // 日
        "H+": date.getHours().toString(), // 时
        "M+": date.getMinutes().toString(), // 分
        "S+": date.getSeconds().toString() // 秒
        // 有其他格式化字符需求可以继续添加，必须转化成字符串
      };
      for (var k in opt) {
        ret = new RegExp("(" + k + ")").exec(fmt);
        if (ret) {
          fmt = fmt.replace(ret[1], (ret[1].length === 1) ? (opt[k]) : (opt[k].padStart(ret[1].length, "0")))
        };
      };
      return fmt;
    },
    link() {
      if (this.$route.query.mobile) {
        this.$router.push({
          name: "comments",
          query: { pageId: this.pageId, userId: this.userId },
        });
      } else {
        this.show = !this.show;
      }
    },
    toText(text) {
      if (text) {
        return text
          .replace(/<(style|script|iframe)[^>]*?>[\s\S]+?<\/\1\s*>/gi, "")
          .replace(/<[^>]+?>/g, "")
          .replace(/\s+/g, " ")
          .replace(/ /g, " ")
          .replace(/>/g, " ");
      }
    },
    toImg(text) {
      text = text.slice(text.indexOf('src="') + 5);
      text = text.slice(0, text.indexOf('"'));
      return text;
    },
  },
};
</script>

<style lang="scss" scoped>
*{
  overflow-x: hidden;
}
#writeRemote {
  position:fixed;
  left:auto;
  right:auto;
  _position:absolute;
  z-index: 1000;
  margin:0 auto;
  background: #FBFCFF;
  bottom: 0;
  width: 100%;
  border-top: 1px solid #E9EAEB;
  padding: 13px 16px 30px 16px;
  .van-cell{
    width: 90%;
    border: 1px solid #F0EFF4;
    border-radius: 20px;
    color: #95A4B3;
  }
  .van-cell__title{
    color: #B7C0CA;
    padding-left: 6px;
    line-height: 26px;
  }
  .van-popup{
    padding: 0 20px;
  }
  .popTitle{
    font-family: PingFang SC;
    font-style: normal;
    font-weight: 600;
    font-size: 18px;
    line-height: 24px;
    text-align: center;
    color: #313C56;
    padding: 16px 0;
  }
  .commentSection{
    padding: 12px 0 16px;
    .commentInput{
      width: 90%;
      height: 97px;
      background: #FFFFFF;
      border: 1px solid #F0EFF4;
      box-sizing: border-box;
      border-radius: 8px;
      padding: 12px;
      font-family: PingFang SC;
      font-style: normal;
      font-weight: normal;
      font-size: 14px;
      line-height: 18px;
      color: #B7C0CA;
    }
  }
  .commentSend{
    width: 84%;
    background: #2196F3;
    border-radius: 8px;
    font-family: PingFang SC;
    font-style: normal;
    font-weight: normal;
    font-size: 16px;
    line-height: 16px;
    text-align: center;
    color: #FFFFFF;
    padding: 12px;
    margin-bottom: 24px;
  }
}

.page {
  flex-grow: 1;
  background-color: white;
}
.page-header {
  text-align: left;
  font-size: 18px;
  border-bottom: 1px solid #e9e9e9;
  padding: 0 15px;
  .page-title {
    font-size: 20px;
    color: #615f6c;
    font-weight: bold;
  }

  .page-created_at {
    font-family: PingFang SC;
    font-style: normal;
    font-weight: normal;
    display: flex;
    justify-content: space-between;
    span:first-child{
      font-size: 14px;
      line-height: 16px;
      color: #6B7885;
    }
    span:last-child{
      border-radius: 12px;
      font-size: 13px;
      line-height: 20px;
      padding: 0 8px;
    }
    .green{
      background-color: #68C635;
      color: #FFFFFF;
    }
    .end{
      background-color: #F4F4F8;
      color: #95A4B3;
    }
  }
}
.page-header > * {
  margin: 15px 0;
}

.page-content {
  padding: 24px 15px 0;
  img{
    width: 345px;
    height: 160px;
    object-fit: cover;
    background: #FFFFFF;
    border-radius: 8px;
  }
  p{
    margin: 16px 0 24px;
    font-family: PingFang SC;
    font-style: normal;
    font-weight: normal;
    font-size: 16px;
    line-height: 27px;
    text-align: justify;
    color: #313C56;
  }
}

.page-comments {
  margin-bottom: 92px;
  .page-comments-divider {
    margin: 0 -10px;
    background-color: #F4F4F8;
    height: 10px;
    box-shadow: inset 0 1px 1px 0 rgba(100, 100, 100, 0.1);
  }

  .page-comments-header {
    margin: 0 -10px;
    border-bottom: 1px solid #e9e9e9;
    padding: 15px 10px;
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    font-size: 18px;
    flex-direction: row;
    align-items: center;
    text-align: left;

    span {
      flex-grow: 1;
      width: 0.5rem;
      color: #615f6c;
    }

    a {
      font-size: 18px;
      color: #fd7d58;
    }
    .popup-title {
      display: flex;
      justify-content: center;
      margin-top: 15%;
    }

    .popup-buttom {
      width: 70%;
      border: 1px solid #4caf50;
      padding: 10px;
      margin: 20% auto 0px;
      border-radius: 4px;
      text-align: center;
      background: #4caf50;
      color: #fff;
      font-size: 16px;
      display: block;
    }
  }
  .no-page-comments {
    font-size: 16px;
    text-align: center;
    color: #888888;
    padding: 10px 0px 100px;
  }
}

.page-comment-item {
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  margin: 23px 0;

  .comment-author-header {
    margin: 0 16px 0 15px;
    width: 40px;
    height: 40px;
    img {
      width: 100%;
      display: block;
      border-radius: 8px;
    }
  }
  .comment-details {
    text-align: left;
    flex-grow: 1;
    width: 0.5rem;
    padding-right: 15px;
    font-family: PingFang SC;
    font-style: normal;
    .comment-author-name {
      position: relative;
      font-weight: 600;
      font-size: 16px;
      line-height: 16px;
      color: #313C56;
    }
    .comment-body {
      padding: 4px 0;
      font-weight: normal;
      font-size: 14px;
      line-height: 18px;
      color: #95A4B3;
      border-bottom: 1px solid #E9EAEB;
    }
  }
}
.popup {
  padding: 20px;
}
.popup_h1 {
  margin-top: 20px;
  font-size: 16px;
  text-align: left;
}
.popup_title {
  margin-top: 5%;
  display: flex;
  font-size: 14px;
}

.popup_botton {
  margin-top: 10%;
  display: flex;
  justify-content: flex-end;
  margin-right: 10%;
  .noLogin {
    font-size: 14px;
    margin-right: 10%;
  }
  .goLogin {
    text-align: center;
    font-size: 14px;
    color: #223469;
  }
}
</style>
