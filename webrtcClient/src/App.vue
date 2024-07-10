<template>
  <div class="main">
    <input @keyup.enter="joinRoom" v-model="roomId">输入房间号</input>
  </div>
  <div class="video">
    <video ref="localVideo" autoplay></video>
    <video v-for="stream in remoteStreams" :srcObject="stream" autoplay></video>
  </div>
</template>

<script setup>
import { ref, reactive } from "vue";
import io from "socket.io-client";
const socket = io("http://localhost:3000");
const roomId = ref("");
const localVideo = ref(null);
const remoteStreams = reactive({});
const peerConnection=ref(null);


// 获取本地视频流
const getLoacalStream = async () => {
const stream =await navigator.mediaDevices.getUserMedia({
  video: true,
  audio: true,
})
// 将本地视频流添加到video标签
localVideo.value.srcObject = stream;
localVideo.value.play();
};
const joinRoom=async()=>{
  console.log("joinRoom");
  // socket.emit("joinRoom", "123");
  socket.emit("joinRoom", roomId.value);

  // console.log("添加本地流到peerConnection前",peerConnection.value);
  await getLoacalStream();

  // 创建RTCPeerConnection 对象,用于建立点对点连接
 peerConnection.value = new RTCPeerConnection({
  iceServers: [
    {
      urls: "stun:stun.l.google.com:19302",
    },
  ],
});


// 将本地流添加到RTCPeerConnection对象
localVideo.value.srcObject.getTracks().forEach((track) => {
  peerConnection.value.addTrack(track, localVideo.value.srcObject);
});

// console.log("添加本地流到peerConnection后",peerConnection.value);

// 创建offer
const offer = await peerConnection.value.createOffer();
// console.log("offer", offer);
// 将offer设置为本地描述
await peerConnection.value.setLocalDescription(offer);
// 将offer发送给服务器
socket.emit("offer", offer);

// 监听对方返回的offer
socket.on('offer',async(offer)=>{
  console.log("收到offer",offer);
  // 设置远端描述
  await peerConnection.value.setRemoteDescription(offer);
  // 创建answer
  const answer=await peerConnection.value.createAnswer();
  // console.log("answer",answer);
  // 设置本地描述
  await peerConnection.value.setLocalDescription(answer);
  // 发送answer
  socket.emit("answer",answer);
})

// 监听对方返回的answer
socket.on('answer',async(answer)=>{
  console.log("收到answer",answer);
  // 设置远端描述
  await peerConnection.value.setRemoteDescription(answer);
})




}
</script>

<style>
video {
  width: 200px;
  height: 150px;
  margin: 10px;
}
</style>
