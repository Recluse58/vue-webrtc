<template>
  <div class="video-list" >
      <div v-for="item in videoList"
          v-bind:video="item"
          v-bind:key="item.id"
          class="video-item">
        <video controls autoplay playsinline ref="videos" :height="cameraHeight" :muted="item.muted" :id="item.id"></video>
      </div>
  </div>
</template>

<script>
import RTCMultiConnection from 'rtcmulticonnection';
require('adapterjs');
export default {
  name: 'vue-webrtc',
  components: {
    //RTCMultiConnection
  },
  data() {
    return {
      rtcmConnection: null,
      localVideo: null,
      videoList: [],
      endedVideoList: [], // Kendall
      canvas: null,
    };
  },
  props: {
    roomId: {
      type: String,
      default: 'kazoo-room'
    },
    socketURL: {
      type: String,
      default: "http://localhost:9001/"// 'https://rtcmulticonnection.herokuapp.com:443/'
      //TODO: Implement our own signaling server
    },
    cameraHeight: {
      type: [Number, String],
      default: 160
    },
    autoplay: {
      type: Boolean,
      default: true
    },
    screenshotFormat: {
      type: String,
      default: 'image/jpeg'
    },
    enableAudio: {
      type: Boolean,
      default: true
    },
    enableVideo: {
      type: Boolean,
      default: true
    },
    enableLogs: {
      type: Boolean,
      default: false
    },
    stunServer: {
      type: String,
      default: null
    },
    turnServer: {
      type: String,
      default: null
    }
  },
  watch: {
  },
  mounted() {
    var that = this;

    that.rtcmConnection = new RTCMultiConnection();
    that.rtcmConnection.socketURL = that.socketURL;
    that.rtcmConnection.autoCreateMediaElement = false;
    that.rtcmConnection.enableLogs = that.enableLogs;
    that.rtcmConnection.session = {
      audio: that.enableAudio,
      video: that.enableVideo,
    };
    that.rtcmConnection.sdpConstraints.mandatory = {
      OfferToReceiveAudio: that.enableAudio,
      OfferToReceiveVideo: that.enableVideo
    };
    if ((that.stunServer !== null) || (that.turnServer !== null)) {
      that.rtcmConnection.iceServers = []; // clear all defaults
    }
    if (that.stunServer !== null) {
      that.rtcmConnection.iceServers.push({
        urls: that.stunServer
      });
    }
    if (that.turnServer !== null) {
      var parse = that.turnServer.split('%');
      var username = parse[0].split('@')[0];
      var password = parse[0].split('@')[1];
      var turn = parse[1];
      this.rtcmConnection.iceServers.push({
        urls: turn,
        credential: password,
        username: username
      });
    }
    that.rtcmConnection.onstream = function (stream) {
      console.log('onstream -> incoming streamId', stream.streamid)
      that.videoList.forEach( elem => {
        console.log('onstream -> current video list id: ', elem.id)

      })

      // Kendall
      if(that.endedVideoList && that.endedVideoList.length > 0){
        let bugFound = that.endedVideoList.find(videoId => {
          return videoId === stream.streamid
        });
        if (bugFound != undefined) {
          console.log(stream.streamid, " is supposed to have ended. Skipping this new stream add.")
          return
        }
      }
    // Kendall


      let found = that.videoList.find(video => {
        return video.id === stream.streamid
      })
      if (found === undefined) {
        let video = {
          id: stream.streamid,
          muted: stream.type === 'local',
          type: stream.type
        };
        that.videoList.push(video);
        console.log('onstream -> new-remote-stream')
        that.$emit('new-remote-stream', that.videoList)
        if (stream.type === 'local') {
          that.localVideo = video;
        }
      }

      setTimeout(function(){
        for (var i = 0, len = that.$refs.videos.length; i < len; i++) {
          if (that.$refs.videos[i].id === stream.streamid)
          {
            that.$refs.videos[i].srcObject = stream.stream;
            break;
          }
        }
      }, 1000);

      that.$emit('joined-room', 'joined-room ' + stream.streamid);
    };
    that.rtcmConnection.onstreamended = function (stream) {
      console.log('## onstreamended called ##', stream.streamid);
      var newList = [];
      that.videoList.forEach(function (item) {
        if (item.id !== stream.streamid) {
          newList.push(item);
        }
        else {
          that.endedVideoList.push(stream.streamid);
        }
      });
      that.videoList = newList;
      console.log('onstreamended -> new-remote-stream')
      that.$emit('onstreamended -> left-room', 'left-room ' + stream.streamid);
    };
  },
  methods: {
    join() {
      var that = this;
      this.rtcmConnection.openOrJoin(this.roomId, function (isRoomExist, roomid) {
        if (isRoomExist === false && that.rtcmConnection.isInitiator === true) {
          that.$emit('opened-room', 'opened-room ' + roomid);
        }
      });
    },
    leave() {
      this.rtcmConnection.attachStreams.forEach(function (localStream) {
        localStream.stop();
      });
      this.videoList = [];
      console.log('leave() -> new-remote-stream')
      this.$emit('new-remote-stream', this.videoList)
    },
    capture() {
      return this.getCanvas().toDataURL(this.screenshotFormat);
    },
    getCanvas() {
      let video = this.getCurrentVideo();
      if (video !== null && !this.ctx) {
        let canvas = document.createElement('canvas');
        canvas.height = video.clientHeight;
        canvas.width = video.clientWidth;
        this.canvas = canvas;
        this.ctx = canvas.getContext('2d');
      }
      const { ctx, canvas } = this;
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      return canvas;
    },
    getCurrentVideo() {
      if (this.localVideo === null) {
        return null;
      }
      for (var i = 0, len = this.$refs.videos.length; i < len; i++) {
        if (this.$refs.videos[i].id === this.localVideo.id)
          return this.$refs.videos[i];
      }
      return null;
    },

    shareScreen() {
      var that = this;
      if (navigator.getDisplayMedia || navigator.mediaDevices.getDisplayMedia) {
        // eslint-disable-next-line no-inner-declarations
        function addStreamStopListener(stream, callback) {
          var streamEndedEvent = 'ended';
          if ('oninactive' in stream) {
            streamEndedEvent = 'inactive';
          }
          stream.addEventListener(streamEndedEvent, function() {
            callback();
            callback = function() {};
          }, false);
        }

        // eslint-disable-next-line no-inner-declarations
        function onGettingSteam(stream) {
          that.rtcmConnection.addStream(stream);
          that.$emit('share-started', 'share-started ' + stream.streamid);

          addStreamStopListener(stream, function() {
            that.rtcmConnection.removeStream(stream.streamid);
            that.$emit('share-stopped', 'share-stopped ' + stream.streamid);
          });
        }

        // eslint-disable-next-line no-inner-declarations
        function getDisplayMediaError(error) {
          console.log('Media error: ' + JSON.stringify(error));
        }

        if (navigator.mediaDevices.getDisplayMedia) {
          navigator.mediaDevices.getDisplayMedia({video: true, audio: false}).then(stream => {
            onGettingSteam(stream);
          }, getDisplayMediaError).catch(getDisplayMediaError);
        }
        else if (navigator.getDisplayMedia) {
          navigator.getDisplayMedia({video: true}).then(stream => {
            onGettingSteam(stream);
          }, getDisplayMediaError).catch(getDisplayMediaError);
        }
      }
    }
  }
};
</script>
<style scoped>
.video-list {
  background: whitesmoke;
  height: auto;
}

.video-list div {
  padding: 0px;
}

.video-item {
  background: #c5c4c4;
  display: inline-block;
}
</style>
