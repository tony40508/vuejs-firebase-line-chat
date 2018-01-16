<template lang="pug">
  .container
    //- nameArea
    .name
      h3 Name：{{ userName }}
      .reset(@click="setName()") Reset Name
    //- chatRoom
    .chatRoom
      //- head
      .roomHead
        .roomHead__topButtons
          .roomHead__button.close
          .roomHead__button.minimize
          .roomHead__button.zoom
        img.roomHead__img(src="https://lorempixel.com/50/50" draggable="false")
        .roomHead__title NCKU IM
        img.roomHead__img_users(src="static/img/users-icon.svg")
        .users-text {{userNumber}}

      //- roomBody
      #js-roomBody.roomBody
        //- 使用 template 來當迴圈容器，或是判斷用的容器，當條件達成時產出 template 內容
        <template v-for="item in messages">
          //- other people
          <template v-if="item.userName != userName">
            .messageBox
              img.messageBox__user(src="https://lorempixel.com/40/40/" draggable="false")
              .messageBox__content
                .messageBox__name {{item.userName}}
                .messageBox__message(v-if="item.type == 'text'")
                  .messageBox__text {{item.message}}
                  .messageBox__readMore(@click="readMore($event)") 顯示更多
                .messageBox__image(v-if="item.type == 'image'")
                  img(:src="item.message")
              .messageBox__time {{item.timeStamp}}
          </template>
          //- self
          <template v-if="item.userName == userName">
            .messageBox.messageBox--self
              .messageBox__time {{item.timeStamp}}
              .messageBox__content
                .messageBox__message(v-if="item.type == 'text'")
                  .messageBox__text {{item.message}}
                .messageBox__image(v-if="item.type == 'image'")
                  img(:src="item.message")
          </template>
        </template>
        //- progressBar
        .messageBox.messageBox--self(v-show="upload")
          .messageBox__progress
            #js-progressBar.messageBox__progress--state
            .messageBox__progress--number {{progress}}
      //- bottom
      .roomBottom(:class="{ disable: !userName }")
        .roomBottom__tools
          .roomBottom__tools_upload
            input(type="file" accept="image/*" @change="sendImage($event)")
            img(src="@/assets/tools_file.png")
          .roomBottom__tools_emoji
            img(src="@/assets/tools_emoji.png")
            .emoji-list
              span(@click="sendEmoji($event)").emoji 😄
              span(@click="sendEmoji($event)").emoji 😃
              span(@click="sendEmoji($event)").emoji 😀
              span(@click="sendEmoji($event)").emoji 😊
              span(@click="sendEmoji($event)").emoji 😉

        .roomBottom__input
          //- 若要再帶入原生 js 的 event(e) 到 function 中，必須使用 $event 當參數傳入
          textarea#js-message.roomBottom__input__textarea(:class="{ disable: !userName }" @keydown.enter="sendMessage($event)")

    //- modal
    <transition name="fade">
      .modal(v-show="userNameSet || userName == ''")
        .modal__container
          header.modal__header
            h2.view-title Your Name
          .modal__body
            //- 使用 @keydown.enter 來偵測 keydown enter，觸發時執行 saveName()
            input#js-userName.userName(type="text" maxlength="6" @keydown.enter="saveName()" :value="userName")
            .button(@click="saveName()") OK
          footer.modal__footer
    </transition>
</template>

<script>
// msgRef = firebase中的資料表/messages/，若沒有的會自動建立
const msgRef = firebase.database().ref('/messages/');
const storageRef = firebase.storage().ref('/images/');
export default {
  // 指定此頁使用的 name
  name: 'ChatRoom',
  // 資料位置，於html中可用{{}}渲染出來
  data() {
    return {
      userNameSet: false, // 姓名輸入框
      userName: '', // 名稱
      userNumber: 0,
      messages: [], // 訊息內容
      upload: false, // 上傳進度框
      progress: '' // 上傳進度 % 數
    }
  },
  watch: {
    messages: {
      handler(val, oldVal) {
        const roomBody = document.querySelector('#js-roomBody');
        roomBody.scrollTop = roomBody.scrollHeight;
      },
      deep: true
    }
  },
  // 這個頁面的functions
  methods: {
    /** 彈出設定視窗 */
    setName() {
      const vm = this;
      vm.userNameSet = true;
    },
    /** 儲存設定名稱 */
    saveName() {
      // vue 的 method 中 this 是指 export 中這整塊的資料
      const vm = this;
      const userName = document.querySelector('#js-userName').value;
      if (userName.trim() == '') { return; }
      // 這裡的vm.userName(this.userName)就是data()裡面的userName
      vm.userName = userName;
      vm.userNameSet = false;
    },
    /** 取得時間 */
    getTime() {
      const now = new Date();
      const hours = now.getHours();
      const minutes = now.getMinutes();
      return `${(hours >= 12) ? "下午" : "上午"} ${hours}:${(minutes < 10) ? '0' + minutes : minutes}`;
    },
    /** 傳送訊息 */
    sendMessage(e) {
      const vm = this;
      const userName = document.querySelector('#js-userName');
      let message = document.querySelector('#js-message');
      // 如果是按住 shift 則不傳送訊息(多行輸入)
      if (e.shiftKey) {
        return false;
      }
      // 如果輸入是空則不傳送訊息
      if (message.value.length <= 1 && message.value.trim() == '') {
        // 避免 enter 產生的空白換行
        e.preventDefault();
        return false;
      }
      // 對 firebase 的 db 做 push，db 只能接受 json 物件格式，若要用陣列要先轉字串來存
      msgRef.push({
        userName: userName.value,
        type: 'text',
        message: message.value,
        // 取得時間，這裡的 vm.getTime() 就是method中的getTime
        timeStamp: vm.getTime()
      })
      // 清空輸入欄位並避免 enter 產生的空白換行
      message.value = '';
      e.preventDefault();
    },
    /** 傳送圖片 */
    sendImage(e) {
  		const vm = this;
      const userName = document.querySelector('#js-userName');
      // 取得上傳檔案的資料
      const file = e.target.files[0];
      const fileName = Math.floor(Date.now() / 1000) + `_${file.name}`;
      const metadata = {
        contentType: 'image/*'
      };
      // 取得HTML進度條元素
      let progressBar = document.querySelector('#js-progressBar');
      // 上傳資訊設定
      const uploadTask = storageRef.child(fileName).put(file, metadata);
      // 上傳狀態處理
      uploadTask.on(firebase.storage.TaskEvent.STATE_CHANGED,
        /* 上傳進度 */
        function(snapshot) {
          let progress = Math.floor((snapshot.bytesTransferred / snapshot.totalBytes) * 100);
          if (progress < 100) {
            // 開啟進度條
            vm.upload = true;
            vm.progress = `${progress}%`;
            progressBar.setAttribute('style', `width:${progress}%`);
          }
        },
        /* 錯誤處理 */
        function(error) {
          msgRef.child('bug/').push({
            userName: userName.value,
            type: 'image',
            message: error.code,
            timeStamp: vm.getTime()
          })
        },
        /* 上傳結束處理 */
        function() {
          var downloadURL = uploadTask.snapshot.downloadURL;
          msgRef.push({
            userName: userName.value,
            type: 'image',
            message: downloadURL,
            timeStamp: vm.getTime()
          })
          // 關閉進度條
          vm.upload = false;
        });
    },
    /** 顯示更多 */
    readMore(e) {
      // 把內容高度限制取消
      e.target.previousElementSibling.setAttribute('style', 'max-height: 100%;')
      // 隱藏 "顯示更多" 按紐
      e.target.setAttribute('style', 'display: none;');
    },
    /** 傳送 Emoji */
    sendEmoji(e) {
      const textarea = document.getElementById('js-message');
      // 取得當前的 target
      let currentTarget = e.currentTarget;
      textarea.value = textarea.value + e.currentTarget.innerHTML;
    }
  },
  // mounted 是 vue 的生命週期之一，代表模板已編譯完成，已經取值準備渲染元件了
  mounted() {
    const vm = this;
    msgRef.on('value', function(snapshot) {
      const val = snapshot.val();
      vm.messages = val;

      // Convert A Firebase Database Snapshot To An Array
      let returnArr = [];
      snapshot.forEach(childSnapshot => {
          let item = childSnapshot.val();
          item.key = childSnapshot.key;
          returnArr.push(item.userName);
      });

      // 取出陣列不重複值
      let resultArr = Array.from(new Set(returnArr));
      vm.userNumber = resultArr.length;
    })
  },
  // update 是 vue的生命週期之一，在元件渲染完成後執行
  updated() {
    // 判斷內容高度超過 300 就隱藏起來，把 "顯示更多" 按紐打開
    const messages = document.querySelectorAll('.messageBox__message');
    messages.forEach((message) => {
      if (message.offsetHeight > 300) {
        message.querySelector('.messageBox__readMore').setAttribute('style', 'display: block');
      }
    })
    // 當畫面渲染完成，把聊天視窗滾到最底部(讀取最新消息)
    const roomBody = document.querySelector('#js-roomBody');
    roomBody.scrollTop = roomBody.scrollHeight;
  }
}
</script>

<style scoped>
* {
  font-family: '微軟正黑體', sans-serif;
  letter-spacing: 1.5px;
  margin: auto;
}
.container {
  padding: 0px;
}
.name {
  text-align: center;
  margin: -10px 0px 10px 0px;
}
.name h3{
  text-align: center;
  color: #333333;
  display: inline;
  margin-right: 30px;
}
.reset {
  margin-top: 10px;
  padding: 5px 10px;
  border-radius: 10px;
  font-weight: 600;
  color: #333333;
  background-color: #CCCCCC;
  display: inline-block;
  cursor: pointer;
}
.chatRoom {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  border-radius: 5px;
  min-width: 500px;
  max-width: 500px;
  box-shadow: 0 14px 28px rgba(0, 0, 0, 0.25), 0 10px 10px rgba(0, 0, 0, 0.22);
}

/* Head */
.roomHead {
  width: 100%;
  height: 85px;
  border-radius: 5px 5px 0px 0px;
  background-color: #2B364B;
  position: relative;
}
.roomHead__topButtons {
  padding: 2px 0px 5px 8px;
  text-align: left;
}
.roomHead__button {
  height: 12px;
  width: 12px;
  border-radius: 50%;
  margin: 0px 3px;
  display: inline-block;
  cursor: pointer;
}
.close {
  background-color: #FF625A;
}
.minimize {
  background-color: #FFC02F;
}
.zoom {
  background-color: #28CB40;
}
.roomHead__img {
  width: 50px;
  height: 50px;
  margin: 0px 10px 10px 12px;
  border-radius: 50%;
  position: absolute;
  cursor: pointer;
}
.roomHead__title {
  font-size: 16px;
  font-weight: 600;
  color: #FFFFFF;
  height: 80px;
  margin: 0px 0px 0px 80px;
  position: absolute;
  cursor: pointer;
}

.roomHead__img_users {
  position: absolute;
  top: 52px;
  left: 75px;
  width: 25px;
  height: 25px;
  opacity: 0.4;
}

.users-text {
  color: white;
  font-size: 16px;
  position: absolute;
  top: 54px;
  left: 105px;
  opacity: 0.4;
}

/* Body */
.roomBody {
  padding: 10px 0px;
  background-color: #fff;
  height: 60vh;
  overflow-y: auto;
  overflow-x: hidden;
}
.messageBox {
  padding: 5px 10px;
  position: relative;
}
.messageBox__user {
  height: 40px;
  width: 40px;
  border-radius: 50%;
  vertical-align: top;
  display: inline-block;
  cursor: pointer;
}
.messageBox__content {
  max-width: 70%;
  display: inline-block;
}
.messageBox__name {
  margin: 5px 0px 5px 5px;
  font-size: 13px;
  color: #727C8A;
  vertical-align: top;
  cursor: pointer;
}
.messageBox__message {
  margin: 5px 0px 5px 5px;
  font-size: 12px;
  color: #35393D;
  letter-spacing: 0.6px;
  background-color: #E3E8EB;
  border-radius: 12px;
  line-height: 1.5;
  text-align: left;
  word-break: break-all;
  /*：與html的<pre></pre>同效果，可以使textarea的換行元素正常顯示 */
  white-space: pre-line;
}
.messageBox__text {
  padding: 8px 10px 9px 11px;
  max-height: 300px;
  overflow: hidden;
}
.messageBox__readMore {
  border-top: 1px solid #D9DBDD;
  margin-top: 6px;
  padding: 6px 13px 10px 13px;
  left: 0;
  right: 0;
  cursor: pointer;
  display: none;
}
.messageBox__image {
  margin: 5px 25px 5px 5px;
}
.messageBox__image img {
  border-radius: 5px;
  max-width: 100%;
  max-height: 335px;
}
.messageBox__time {
  transform: scale(0.7);
  color: #ACB0B8;
  vertical-align: bottom;
  margin: 0px 0px 5px -12px;
  display: inline-block;
}
.messageBox__progress {
  width: 25%;
  margin-right: 60px;
  border-radius: 5px;
  background-color: #D4D9E1;
  height: 6px;
}
.messageBox__progress--state {
  background-color: #00CE00;
  height: 6px;
  width: 10%;
  border-radius: 5px;
  float: right;
}
.messageBox__progress--number {
  position: absolute;
  top: 0;
  right: 0;
  font-size: 12px;
  color: #B7B7B7;
  margin-right: 35px;
}
.messageBox--self {
  text-align: right;
}
.messageBox--self .messageBox__message {
  background-color: #AFF47E;
  margin-right: 25px;
}
.messageBox--self .messageBox__message__text {
  padding: 8px 10px 9px 11px;
  max-height: 100%;
  overflow: unset;
}
.messageBox--self .messageBox__time {
  margin: 0px -16px 5px 0px;
}

/* Bottom */
.roomBottom {
  bottom: 0px;
  border-radius: 0px 0px 5px 5px;
  background-color: #FFFFFF;
}
.roomBottom__tools {
  border-top: solid 1px #E7E7E7;
  border-bottom: solid 2px #E7E7E7;
  background-color: #F6F6F6;
  height: 30px;
  padding: 0px 5px;
}
.roomBottom__tools_upload {
  height: 20px;
  margin: 5px;
  padding: 1px;
  display: inline-block;
  position: relative;
  overflow: hidden;
  transition: all 0.3s;
}
.roomBottom__tools_upload:hover {
  background-color: #e3e8eb;
}
.roomBottom__tools_upload input {
  width: 100%;
  height: 100%;
  left: 0;
  right: 0;
  opacity: 0;
  position: absolute;
  cursor: pointer;
  /* 讓input file可以支援pointer要加 pl100% */
  padding-left: 100%;
}
.roomBottom__tools_upload img {
  height: 100%;
}
.roomBottom__input {
  padding: 10px 10px 5px 10px;
}
.roomBottom__input__textarea {
  width: 100%;
  height: 60px;
  border: none;
  resize: none;
  outline: none;
}
/* emoji */
.roomBottom__tools_emoji {

}
.roomBottom__tools_emoji img {
  position: absolute;
  bottom: 90px;
  left: 35px;
  width: 20px;
  height: 20px;
  cursor: pointer;
  transition: all 0.3s;
}
.roomBottom__tools_emoji img:hover {
  background-color: #e3e8eb;
}
.emoji-list {
  position: absolute;
  bottom: 120px;
  width: 200px;
  height: 100px;
  background-color: #ebebeb;
  border-radius: 5px;
  overflow-y: scroll;
  padding: 0 5px 0 5px;
}
.emoji {
  font-size: 20px;
  cursor: pointer;
  margin: 2px;
}
/* status */
.disable {
  pointer-events: none;
  background-color: #ebebeb;
}

/* modal */
.modal {
  z-index: 3;
  padding: 50px 0px;
  position: fixed;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  overflow: auto;
  background-color: rgba(0, 0, 0, 0.4);
  animation: opac 0.8s;
  letter-spacing: 2px;
  text-align: unset;
}
.modal__container {
  margin: auto;
  position: relative;
  padding: 10px;
  outline: 0;
  max-width: 300px;
}
.modal__header {
  color: #fff;
  background-color: #2B364B;
  padding: 10px;
  text-align: center;
  border-radius: 5px 5px 0px 0px;
}
.modal__body {
  background-color: #fff;
  padding: 20px 50px;
  text-align: center;
}
.modal__body p {
  text-align: left;
  line-height: 24px;
}

.modal__body input {
  outline: 0;
  box-shadow: 0 0 0px 1000px #fefefe inset;
  /* Remove the yellow background on input autofill */
}
.modal__body input:focus{
  border: solid 1px #2B364B;
}

.modal__body .userName{
  background-color: #fefefe;
}

.modal__img {
  max-width: 100%;
}
.modal__footer {
  color: #fff;
  background-color: #2B364B;
  height: 8px;
  border-radius: 0px 0px 5px 5px;
}
/* transition */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.6s
}
.fade-enter, .fade-leave-to  {
  opacity: 0
}

/* name set */
.userName {
  height: 30px;
  font-size: 16px;
  margin-bottom: 10px;
  border: solid 1px #cbcbcb;
  width: 70%;
  text-align: center;
  display: inline-block;
}
.button {
  font-size: 14px;
  color: #FFFFFF;
  background-color: #2B364B;
  padding: 10px 20px;
  display: inline-block;
  cursor: pointer;
}

/* media */
@media screen and (max-width: 425px) {
  .messageBox__content {
    max-width: 60%;
  }
}

@media screen and (max-width: 385px) {
  .messageBox__content {
    max-width: 50%;
  }
}
</style>
