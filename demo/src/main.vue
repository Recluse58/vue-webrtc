<template>
  <div class="container">
    <div class="row">
      <div class="col-md-12 my-3">
        <h2>Room</h2>
        <input v-model="roomId">
      </div>
    </div>
    <div class="row">
      <div class="col-md-12">
        <div class="">
          <vue-webrtc ref="webrtc"
                      width="100%"
                      :roomId="roomId"
                      v-on:joined-room="joinedRoomEvent"
                      v-on:left-room="leftRoomEvent"
                      v-on:opened-room="openedRoomEvent"
                      v-on:share-started="shareStartedEvent"
                      v-on:share-stopped="shareStoppedEvent"
                      v-on:new-remote-stream="newRemoteStreamEvent"
                      @error="onError" />
        </div>
        <div class="row">
          <div class="col-md-12 my-3">
            <button type="button" class="btn btn-primary" @click="onJoin">Join</button>
            <button type="button" class="btn btn-primary" @click="onLeave">Leave</button>
            <button type="button" class="btn btn-primary" @click="onCapture">Capture Photo</button>
            <button type="button" class="btn btn-primary" @click="onShareScreen">Share Screen</button>
          </div>
        </div>
      </div>
    </div>
    <div class="row">
      <div class="col-md-12">
        <h2>Captured Image</h2>
        <figure class="figure">
          <img :src="img" class="img-responsive" />
        </figure>
      </div>
    </div>
  </div>
</template>

<script>
  import Vue from 'vue'
  import { WebRTC } from 'plugin';
  import { find, head } from 'lodash';

  import * as io from 'socket.io-client'
  window.io = io

  Vue.component(WebRTC.name, WebRTC);

  export default {
    name: 'app',
    components: {
    },
    data() {
      return {
        img: null,
        roomId: "kazoo-room-01"
      };
    },
    computed: {
    },
    watch: {
    },
    methods: {
      onCapture() {
        this.img = this.$refs.webrtc.capture();
      },
      onJoin() {
        this.$refs.webrtc.join();
      },
      onLeave() {
        this.$refs.webrtc.leave();
      },
      onShareScreen() {
        this.img = this.$refs.webrtc.shareScreen();
      },
      onError(error, stream) {
        console.log('On Error Event', error, stream);
      },
      leftRoomEvent(event){
        console.log('leftRoomEvent : ', event);
      },
      joinedRoomEvent(event){
        this.joinedRoom = true;
        console.log('joinedRoomEvent : ', event);
      },
      openedRoomEvent(event){
        this.joinedRoom = true;
        console.log('openedRoomEvent : ', event);
      },
      shareStartedEvent(event){
        console.log('shareStartedEvent : ', event);
      },
      shareStoppedEvent(event){
        console.log('shareStoppedEvent : ', event);
      },
      logEvent(event) {
        console.log('Event : ', event);
      },
      newRemoteStreamEvent(videoList) {
        console.log('newRemoteStream : ', videoList);
        //this.videoList = videoList.filter( video => video.type != 'local')
      }
    }
  };
</script>
