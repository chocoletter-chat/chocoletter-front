<template>
  <div class="chat-container">
    <h2>실시간 채팅</h2>

    <!-- 사용자 정보 입력 UI -->
    <div v-if="!username || !nickname || !roomId">
      <input v-model="tempUsername" placeholder="사용자 ID를 입력하세요..." />
      <input v-model="tempNickname" placeholder="닉네임을 입력하세요..." />
      <input v-model="tempRoomId" placeholder="채팅방 ID를 입력하세요..." />
      <button @click="setUserInfo">확인</button>
    </div>

    <div v-else>
      <p class="user-info">
        현재 사용자: <strong>{{ nickname }} ({{ username }})</strong> | 채팅방
        ID: <strong>{{ roomId }}</strong>
      </p>

      <div class="chat-box">
        <div v-for="(msg, index) in messages" :key="index" class="message">
          <span class="sender">{{ msg.senderName }} ({{ msg.senderId }}):</span>
          {{ msg.content }}
          <span class="timestamp">{{ formatTimestamp(msg.createdAt) }}</span>
        </div>
      </div>

      <div class="input-box">
        <input
          v-model="message"
          @keyup.enter="sendMessage"
          placeholder="메시지를 입력하세요..."
        />
        <button @click="sendMessage">전송</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import { Client } from "@stomp/stompjs";

const messages = ref([]); // 메시지 목록
const message = ref(""); // 입력된 메시지
const stompClient = ref(null); // STOMP 클라이언트
const tempUsername = ref(""); // 임시 사용자 ID
const tempNickname = ref(""); // 임시 닉네임
const tempRoomId = ref(""); // 임시 채팅방 ID
const username = ref(localStorage.getItem("username") || ""); // 사용자 ID
const nickname = ref(localStorage.getItem("nickname") || ""); // 닉네임
const roomId = ref(localStorage.getItem("roomId") || ""); // 채팅방 ID

// 사용자 정보 설정
const setUserInfo = () => {
  if (
    tempUsername.value.trim() === "" ||
    tempNickname.value.trim() === "" ||
    tempRoomId.value.trim() === ""
  )
    return;
  username.value = tempUsername.value.trim();
  nickname.value = tempNickname.value.trim();
  roomId.value = tempRoomId.value.trim();
  localStorage.setItem("username", username.value);
  localStorage.setItem("nickname", nickname.value);
  localStorage.setItem("roomId", roomId.value);
  connectStomp();
};

const connectStomp = () => {
  stompClient.value = new Client({
    brokerURL: "ws://localhost:8080/chat", // STOMP 서버 주소
    reconnectDelay: 5000, // 재연결 대기 시간 (5초)
    onConnect: () => {
      console.log("✅ STOMP 연결 성공");
      subscribeToRoom();
    },
    onDisconnect: () => {
      console.log("🔴 STOMP 연결 종료");
    },
    onStompError: (frame) => {
      console.error("🚨 STOMP 오류:", frame);
    },
  });
  stompClient.value.activate();
};

const subscribeToRoom = () => {
  if (!stompClient.value || !stompClient.value.connected) {
    console.error("🚨 STOMP 연결되지 않음. 구독 불가능.");
    return;
  }
  stompClient.value.subscribe(`/topic/${roomId.value}`, (message) => {
    console.log("📩 새 메시지 수신:", message.body);
    try {
      const receivedMessage = JSON.parse(message.body);
      if (
        receivedMessage.senderId &&
        receivedMessage.senderName &&
        receivedMessage.content &&
        receivedMessage.createdAt
      ) {
        messages.value.push(receivedMessage);
      } else {
        console.error("🚨 메시지 형식이 올바르지 않음:", receivedMessage);
      }
    } catch (error) {
      console.error("❌ 메시지 JSON 파싱 오류:", error);
    }
  });
  console.log(`✅ 채팅방 구독 완료: /topic/${roomId.value}`);
};

const sendMessage = () => {
  if (message.value.trim() === "" || !stompClient.value.connected) return;
  const msgObject = {
    roomId: roomId.value,
    senderId: username.value,
    senderName: nickname.value,
    content: message.value,
  };
  console.log("📤 메시지 전송:", msgObject);
  stompClient.value.publish({
    destination: `/app/send`,
    body: JSON.stringify(msgObject),
  });
  message.value = "";
};

const formatTimestamp = (timestamp) => {
  const date = new Date(timestamp);
  return date.toLocaleString();
};

onMounted(() => {
  if (username.value && nickname.value && roomId.value) {
    connectStomp();
  }
});

onUnmounted(() => {
  if (stompClient.value) {
    stompClient.value.deactivate();
  }
});
</script>

<style scoped>
.chat-container {
  width: 400px;
  margin: 20px auto;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 5px;
  background: #f9f9f9;
}
h2 {
  text-align: center;
}
.chat-box {
  height: 300px;
  overflow-y: auto;
  border: 1px solid #ccc;
  padding: 10px;
  background: white;
}
.message {
  margin-bottom: 5px;
}
.sender {
  font-weight: bold;
  color: #007bff;
}
.timestamp {
  font-size: 0.8em;
  color: gray;
  margin-left: 10px;
}
.input-box {
  display: flex;
  margin-top: 10px;
}
.user-info {
  text-align: center;
  margin-bottom: 10px;
}
input {
  flex: 1;
  padding: 5px;
}
button {
  padding: 5px 10px;
  margin-left: 5px;
  cursor: pointer;
}
</style>
