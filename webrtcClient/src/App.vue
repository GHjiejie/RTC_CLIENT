<template>
  <div class="main">
    <input @keyup.enter="joinRoom" v-model="roomId" placeholder="输入房间号" />
  </div>
  <div class="video">
    <video ref="localVideo" autoplay></video>
    <video
      v-for="(stream, index) in remoteStreams"
      :key="index"
      :ref="'remoteVideo' + index"
      autoplay
    ></video>
  </div>
</template>

<script setup>
import { ref } from "vue";
import { io } from "socket.io-client";

const socket = io("wss://ahjie.com:3000");
const roomId = ref("");
const localVideo = ref(null);
const remoteStreams = ref([]);
const peerConnections = ref({});

// 获取本地视频流
const getLocalStream = async () => {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true,
  });
  // 将本地视频流添加到video标签
  localVideo.value.srcObject = stream;
  localVideo.value.play();
  return stream;
};

const createPeerConnection = (socketId) => {
  const pc = new RTCPeerConnection({
    iceServers: [
      {
        urls: "stun:stun.l.google.com:19302",
      },
    ],
  });

  pc.onicecandidate = (event) => {
    if (event.candidate) {
      socket.emit("iceCandidate", {
        candidate: event.candidate,
        roomId: roomId.value,
        socketId: socketId,
      });
    }
  };

  pc.ontrack = (event) => {
    if (
      !remoteStreams.value.some((stream) => stream.id === event.streams[0].id)
    ) {
      remoteStreams.value.push(event.streams[0]);
    }
  };

  return pc;
};

const joinRoom = async () => {
  console.log("joinRoom");
  socket.emit("joinRoom", roomId.value);

  const localStream = await getLocalStream();

  socket.on("allUsers", (users) => {
    users.forEach((socketId) => {
      const pc = createPeerConnection(socketId);
      localStream
        .getTracks()
        .forEach((track) => pc.addTrack(track, localStream));
      peerConnections.value[socketId] = pc;
    });
  });

  socket.on("userJoined", (socketId) => {
    const pc = createPeerConnection(socketId);
    localStream.getTracks().forEach((track) => pc.addTrack(track, localStream));
    peerConnections.value[socketId] = pc;
  });

  socket.on("offer", async (data) => {
    const { offer, socketId } = data;
    const pc =
      peerConnections.value[socketId] || createPeerConnection(socketId);
    await pc.setRemoteDescription(offer);
    const answer = await pc.createAnswer();
    await pc.setLocalDescription(answer);
    socket.emit("answer", { answer, socketId });
    peerConnections.value[socketId] = pc;
  });

  socket.on("answer", async (data) => {
    const { answer, socketId } = data;
    const pc = peerConnections.value[socketId];
    if (pc) {
      await pc.setRemoteDescription(answer);
    }
  });

  socket.on("iceCandidate", async (data) => {
    const { candidate, socketId } = data;
    const pc = peerConnections.value[socketId];
    if (pc) {
      try {
        await pc.addIceCandidate(candidate);
      } catch (e) {
        console.error("Error adding received ice candidate", e);
      }
    }
  });

  socket.on("disconnect", (socketId) => {
    console.log("Disconnected from server");
    const pc = peerConnections.value[socketId];
    if (pc) {
      pc.close();
      delete peerConnections.value[socketId];
    }
    remoteStreams.value = remoteStreams.value.filter(
      (stream) => stream.id !== socketId
    );
  });
};
</script>

<style>
video {
  width: 200px;
  height: 150px;
  margin: 10px;
}
</style>
