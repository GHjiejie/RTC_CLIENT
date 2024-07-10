<template>
    <div class="main">
      <input @keyup.enter="joinRoom" v-model="roomId" placeholder="输入房间号">
    </div>
    <div class="video">
      <video ref="localVideo" autoplay muted></video>
      <video v-for="stream in remoteStreams" :key="stream.id" :srcObject="stream" autoplay></video>
    </div>
  </template>
  
  <script setup>
  import { ref, reactive } from 'vue';
  import io from 'socket.io-client';
  
  const socket = io('http://localhost:3000');
  const roomId = ref('');
  const localVideo = ref(null);
  const remoteStreams = reactive([]);
  const peerConnection = ref(null);
  
  const getLocalStream = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: true,
    });
    localVideo.value.srcObject = stream;
  };
  
  const joinRoom = async () => {
    console.log('加入房间', roomId.value);
    socket.emit('joinRoom', roomId.value);
  
    await getLocalStream();
  
    peerConnection.value = new RTCPeerConnection({
      iceServers: [
        {
          urls: 'stun:stun.l.google.com:19302',
        },
      ],
    });
  
    localVideo.value.srcObject.getTracks().forEach((track) => {
      peerConnection.value.addTrack(track, localVideo.value.srcObject);
    });
  
    // 处理 ICE Candidate
    peerConnection.value.onicecandidate = (event) => {
      if (event.candidate) {
        socket.emit('candidate', event.candidate);
      }
    };
  
    // 处理远程媒体流
    peerConnection.value.ontrack = (event) => {
      const stream = event.streams[0];
      if (!remoteStreams.includes(stream)) {
        remoteStreams.push(stream);
      }
    };
  
    // 处理对方发送的 offer
    socket.on('offer', async (offer) => {
      await peerConnection.value.setRemoteDescription(offer);
      const answer = await peerConnection.value.createAnswer();
      await peerConnection.value.setLocalDescription(answer);
      socket.emit('answer', answer);
    });
  
    // 处理对方发送的 answer
    socket.on('answer', async (answer) => {
      await peerConnection.value.setRemoteDescription(answer);
    });
  
    // 处理对方发送的 ICE Candidate
    socket.on('candidate', async (candidate) => {
      await peerConnection.value.addIceCandidate(candidate);
    });
  
    // 处理用户断开连接
    socket.on('userDisconnected', () => {
      remoteStreams.length = 0; // 清空远程媒体流
      peerConnection.value.close();
      peerConnection.value = null;
    });
  
    // 创建 offer（如果是第一个加入房间的用户）
    const roomSockets = io.sockets.adapter.rooms.get(roomId.value);
    const numClients = roomSockets ? roomSockets.size : 0;
    if (numClients === 1) {
      const offer = await peerConnection.value.createOffer();
      await peerConnection.value.setLocalDescription(offer);
      socket.emit('offer', offer);
    }
  };
  </script>
  
  <style>
  video {
    width: 200px;
    height: 150px;
    margin: 10px;
  }
  </style>
  
  