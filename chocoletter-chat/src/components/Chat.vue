<template>
  <div class="chat-container">
    <h2>실시간 채팅</h2>

    <!-- 사용자 ID 입력 UI -->
    <div v-if="!username">
      <input
        v-model="tempUsername"
        placeholder="사용자 ID를 입력하세요..."
        @keyup.enter="setUsername"
      />
      <button @click="setUsername">확인</button>
    </div>

    <div v-else>
      <p class="user-info">
        현재 사용자: <strong>{{ username }}</strong>
      </p>

      <div class="chat-box">
        <div v-for="(msg, index) in messages" :key="index" class="message">
          <span class="sender">{{ msg.senderId }}:</span> {{ msg.content }}
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
const roomId = "abc123"; // 특정 채팅방 ID (변경 가능)
const tempUsername = ref(""); // 임시 입력값
const username = ref(localStorage.getItem("username") || ""); // 사용자 ID

// 사용자 ID 설정
const setUsername = () => {
  if (tempUsername.value.trim() === "") return;
  username.value = tempUsername.value.trim();
  localStorage.setItem("username", username.value);
  connectStomp();
};

const connectStomp = () => {
  stompClient.value = new Client({
    brokerURL: "ws://3.34.5.195:8080/chat", // STOMP 서버 주소
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

  stompClient.value.subscribe(`/topic/${roomId}`, (message) => {
    console.log("📩 새 메시지 수신:", message.body);

    try {
      const receivedMessage = JSON.parse(message.body);
      if (receivedMessage.senderId && receivedMessage.content) {
        messages.value.push(receivedMessage);
      } else {
        console.error("🚨 메시지 형식이 올바르지 않음:", receivedMessage);
      }
    } catch (error) {
      console.error("❌ 메시지 JSON 파싱 오류:", error);
    }
  });

  console.log(`✅ 채팅방 구독 완료: /topic/${roomId}`);
};

const sendMessage = () => {
  if (message.value.trim() === "" || !stompClient.value.connected) return;

  const msgObject = {
    roomId: roomId,
    senderId: username.value, // 입력받은 사용자 ID 사용
    content: message.value,
  };

  console.log("📤 메시지 전송:", msgObject);

  // 서버에 메시지 전송
  stompClient.value.publish({
    destination: `/app/send`,
    body: JSON.stringify(msgObject),
  });

  // 내가 보낸 메시지를 즉시 화면에 추가
  // messages.value.push(msgObject);

  // 입력창 비우기
  message.value = "";
};

onMounted(() => {
  if (username.value) {
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
