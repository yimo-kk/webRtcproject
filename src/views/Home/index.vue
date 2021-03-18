<template>
  <div class="home">
    <div class="body_content">
      <ul>
        <li v-for="(item, index) in chatList" :key="index">
          <div
            v-if="item.from || item.to"
            :class="[item.from && item.from == formInline.userId ? 'msg_left' : 'msg_right']"
          >
            <div>
              <span class="name" v-if="item.from !== formInline.userId">{{ item.from }}:</span>
              <span class="msg_content">{{ item.body }}</span>
              <span class="name" v-if="item.from == formInline.userId">:{{ item.from }}</span>
            </div>
          </div>
          <div v-else class="prompt">
            <p>{{ item.body }}</p>
          </div>
        </li>
      </ul>
    </div>
    <div class="inputBox">
      <el-form :inline="true" :model="formInline" class="demo-form-inline">
        <el-row>
          <el-col :xs="6" :sm="6" :md="6" :lg="6" :xl="6">
            <el-form-item label="自己Id">
              <el-input
                size="small"
                v-model="formInline.userId"
                placeholder="连接Id"
              ></el-input> </el-form-item
          ></el-col>
          <el-col :xs="4" :sm="3" :md="2" :lg="2" :xl="2">
            <el-form-item>
              <el-button type="primary" @click="createConnect" size="small">连接</el-button>
            </el-form-item>
          </el-col>
          <el-col :xs="11" :sm="11" :md="11" :lg="11" :xl="11">
            <el-form-item label="对方Id">
              <el-input
                size="small"
                v-model="formInline.oppositeId"
                placeholder="连接Id"
              ></el-input>
              <el-input
                size="small"
                v-model="formInline.txtMsg"
                placeholder="内容"
              ></el-input> </el-form-item
          ></el-col>
          <el-col :xs="2" :sm="2" :md="2" :lg="2" :xl="2">
            <el-form-item>
              <el-button type="primary" @click="onSubmit" size="small">发送</el-button>
              <el-button type="primary" @click="btnCall" size="small">call</el-button>
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
    </div>
    <!-- 视频 -->
    <div class="video" v-show="true">
      <video ref="localVideo" id="localVideo" autoplay controls></video>
      <video ref="remoteVideo" id="remoteVideo" autoplay controls></video>
    </div>
  </div>
</template>

<script>
// import Peer from "peerjs";
export default {
  name: "Home",
  components: {},
  data() {
    return {
      peer: null,
      conn: null,
      formInline: {
        userId: "",
        oppositeId: "",
        txtMsg: ""
      },
      connOption: {
        host: "localhost",
        port: 9000,
        path: "/",
        debug: 3
      },
      chatList: [],
      isCall: false,
      localStream: null
    };
  },
  computed: {},
  watch: {},
  methods: {
    // 创建连接
    createConnect() {
      let _this = this;
      if (!_this.formInline.userId) {
        _this.$message.error("请输入连接自己Id");
        return;
      }
      if (!_this.peer) {
        _this.peer = new Peer(_this.hashCode(_this.formInline.userId), _this.connOption);
        _this.peer.on("open", id => {
          _this.chatList.push({ body: `${id} 连接成功` });
        });
        _this.peer.on("call", function(call) {
          call.answer(_this.localStream);
        });
        _this.peer.on("connection", conn => {
          //收到对方消息的回调
          conn.on("data", data => {
            var msg = JSON.parse(data);
            if (msg.hasOwnProperty("body")) {
              _this.chatList.push({ from: msg.from, body: msg.body, to: msg.to });
              _this.formInline.oppositeId = msg.from;
            }
            if (msg.hasOwnProperty("action")) {
              //收到视频邀请时，弹出询问对话框
              if (msg.action === "call") {
                _this.formInline.oppositeId === msg.from
                  ? ""
                  : (_this.formInline.oppositeId = msg.from);
                // lblFrom.innerText = msg.from;
                // txtTargetId.value = msg.from;
                // $("#dialog-confirm").dialog({
                //   resizable: false,
                //   height: "auto",
                //   width: 400,
                //   modal: true,
                //   buttons: {
                //     Accept: function() {
                //       $(this).dialog("close");
                //       sendMessage(msg.to, msg.from, "accept");
                //     },
                //     Cancel: function() {
                //       $(this).dialog("close");
                //     }
                //   }
                // });
                _this
                  .$confirm(`${_this.formInline.oppositeId}给你视频通话!`, "收到视频通话", {
                    confirmButtonText: "确定",
                    cancelButtonText: "拒绝",
                    type: "warning"
                  })
                  .then(() => {
                    _this.sendMessage({ from: msg.to, to: msg.from, action: "accept" }, "call");
                  })
                  .catch(() => {
                    _this.$message({
                      type: "info",
                      message: "已拒绝"
                    });
                  });
              }

              //接受视频通话邀请
              if (msg.action === "accept") {
                console.log("accept call => " + JSON.stringify(msg));
                console.log(_this.peer);
                debugger;
                var call = _this.peer.call(_this.hashCode(msg.from), _this.localStream);
                call.on("stream", function(stream) {
                  console.log(stream, "收到的流");
                  console.log("received remote stream");
                  _this.$refs.remoteVideo.srcObject = stream;
                  _this.sendMessage({ from: msg.to, to: msg.from, action: "accept-ok" }, "call");
                });
              }

              //接受视频通话邀请后，通知另一端
              if (msg.action === "accept-ok") {
                console.log("accept-ok call => " + JSON.stringify(msg));
                console.log(_this.peer);
                debugger;
                var call = _this.peer.call(_this.hashCode(msg.from), _this.localStream);
                call.on("stream", function(stream) {
                  console.log(stream, "收到的流");
                  console.log("received remote stream");
                  _this.$refs.remoteVideo.srcObject = stream;
                  _this.isCall = true;
                });
              }
            }
          });
        });
      }
    },
    // 发送消息
    onSubmit() {
      //消息体
      var message = {
        from: this.formInline.userId,
        to: this.formInline.oppositeId,
        body: this.formInline.txtMsg
      };
      if (!this.conn) {
        if (!this.formInline.oppositeId) {
          this.$message.error("输入对方Id");
          return;
        }
        if (!this.formInline.txtMsg) {
          this.$message.error("输入发送内容！");
          return;
        }
        //创建到对方的连接
        this.conn = this.peer.connect(this.hashCode(this.formInline.oppositeId));
        this.conn.on("open", () => {
          //首次发送消息
          this.sendMessage(message, "message");
        });
      }
      //发送消息
      if (this.conn.open) {
        this.sendMessage(message, "message");
      }
    },
    hashCode(str) {
      var hash = 0;
      if (str.length == 0) return hash;
      for (let i = 0; i < str.length; i++) {
        let char = str.charCodeAt(i);
        hash = (hash << 5) - hash + char;
        hash = hash & hash;
      }
      return hash;
    },
    sendMessage({ from, to, body, action }, handle) {
      let _this = this;
      let message = { from: from, to: to };
      if (handle === "message") {
        message.body = body;
        _this.conn.send(JSON.stringify(message));
        _this.formInline.txtMsg = "";
        _this.chatList.push(message);
      } else if (handle === "call") {
        message.action = action;
        // let message = { from: from, to: to, action: action };
        if (!_this.conn) {
          _this.conn = _this.peer.connect(_this.hashCode(to));
          _this.conn.on("open", () => {
            _this.conn.send(JSON.stringify(message));
            console.log(message);
          });
        }
        if (_this.conn.open) {
          _this.conn.send(JSON.stringify(message));
          console.log(message);
        }
      }
    },
    // //获取流
    gotStream(stream) {
      console.log(stream, 234);
      debugger;
      console.log("received local stream");
      this.localStream = stream;
      this.$refs.localVideo.srcObject = stream;
      this.isCall = true;
    },
    // //开启本地摄像头
    start() {
      console.log(this.localStream);
      debugger;
      if (this.localStream) {
        // this.localStream.getTracks().forEach(track => {
        //   // track.stop();
        // });
      }
      const videoSource = false;
      const constraints = {
        audio: true,
        video: { width: 320, deviceId: videoSource ? { exact: videoSource } : undefined }
      };
      //facingMode: "user"
      // const constraints = {
      //   audio: true,
      //   video: { width: 320, facingMode: "user" }
      // };

      navigator.mediaDevices
        .getUserMedia(constraints)
        .then(this.gotStream)
        //
        // .then(()=>{
        //   this.$message.error('')
        // })
        .catch(err => {
          console.log(err, "34343");
          this.$message.error("获取流失败");
        });
    },
    // 点击 call
    btnCall() {
      if (!this.formInline.oppositeId) {
        this.$message("输入连接方Id");
        return;
      }
      this.sendMessage(
        { from: this.formInline.userId, to: this.formInline.oppositeId, action: "call" },
        "call"
      );
    }
  },
  created() {},
  mounted() {
    // 判断是否支持 流
    if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
      // alert("webrtc is not supported!");
      this.$message("不支持获取流操作");
      return;
    }
    //获取摄像头列表
    navigator.mediaDevices
      .enumerateDevices()
      .then(deviceInfos => {
        console.log(deviceInfos);
        if (deviceInfos === undefined) {
          return;
        }
        // for (let i = 0; i !== deviceInfos.length; ++i) {
        //   const deviceInfo = deviceInfos[i];
        //   const option = document.createElement("option");
        //   option.value = deviceInfo.deviceId;
        //   if (deviceInfo.kind === "videoinput") {
        //     option.text = deviceInfo.label || `camera ${videoSelect.length + 1}`;
        //     videoSelect.appendChild(option);
        //   }
        // }
      })
      .catch(error => {
        this.$message.error(error.message);
      });
    this.start();
  }
};
</script>
<style lang="less" scoped>
.body_content {
  width: 100%;
  min-height: 400px;
  background: #eee;
  padding: 10px 20px;
  margin: 10px 0;

  .msg_left {
    display: flex;
    justify-content: flex-end;
    color: cornflowerblue;
  }
  .msg_right {
    display: flex;
    justify-content: flex-start;
    color: coral;
  }
  .prompt {
    text-align: center;
    color: #ccc;
  }
}
.video {
  position: absolute;
  top: 0;
  left: 0;
  #localVideo {
    width: 200px;
    height: 120px;
    position: absolute;
    bottom: -115px;
    right: 0;
  }
  #remoteVideo {
    width: 500px;
    height: 300px;
  }
}
</style>
