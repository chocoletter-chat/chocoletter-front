<template>
  <div class="chat-container">
    <h2>실시간 채팅</h2>

    <!-- 사용자 정보 입력 UI -->
    <div v-if="!username || !nickname || !roomId">
      <input v-model="tempUsername" placeholder="사용자 ID를 입력하세요..." />
      <input v-model="tempNickname" placeholder="닉네임을 입력하세요..." />
      <input v-model="tempToken" placeholder="토큰을 입력하세요..." />
      <input v-model="tempRoomId" placeholder="채팅방 ID를 입력하세요..." />
      <button @click="setUserInfo">확인</button>
    </div>

    <div v-else>
      <p class="user-info">
        현재 사용자: <strong>{{ nickname }} ({{ username }})</strong> | 채팅방
        ID: <strong>{{ roomId }}</strong>
      </p>

      <button class="disconnect-btn" @click="disconnectStomp">연결 종료</button>

      <div class="chat-box">
        <div
          v-for="(msg, index) in messages.filter(
            (m) => m.messageType !== 'READ_STATUS'
          )"
          :key="index"
          class="message"
        >
          <span class="sender">{{ msg.senderName }} ({{ msg.senderId }}):</span>
          {{ msg.content }}
          <span class="timestamp">
            {{ formatTimestamp(msg.createdAt) }}
            <span v-if="msg.senderId == username">
              <span v-if="msg.isRead === false" class="unread-indicator"
                >안읽음</span
              >
              <span v-else class="unread-indicator">읽음</span>
            </span>
          </span>
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
import axios from "axios";

const messages = ref([]); // 메시지 목록
const message = ref(""); // 입력된 메시지
const stompClient = ref(null); // STOMP 클라이언트
const tempUsername = ref(""); // 임시 사용자 ID
const tempNickname = ref(""); // 임시 닉네임
const tempRoomId = ref(""); // 임시 채팅방 ID
const tempToken = ref(""); // 임시 토큰
const username = ref(localStorage.getItem("username") || ""); // 사용자 ID
const nickname = ref(localStorage.getItem("nickname") || ""); // 닉네임
const roomId = ref(localStorage.getItem("roomId") || ""); // 채팅방 ID
const token = ref(localStorage.getItem("token") || ""); // 채팅방 ID

// 기존 채팅 기록 불러오기
const fetchChatHistory = async () => {
  if (!roomId.value) return;
  try {
    const response = await axios.get(
      `http://localhost:8080/api/v1/chat/${roomId.value}/all`
    );
    messages.value = response.data.chatMessages.reverse(); // chatMessages 배열을 직접 할당
    console.log("📜 채팅 기록 불러오기 성공:", response.data);
  } catch (error) {
    console.error("❌ 채팅 기록 불러오기 실패:", error);
  }
};

// 사용자 정보 설정
const setUserInfo = () => {
  if (
    tempUsername.value.trim() === "" ||
    tempNickname.value.trim() === "" ||
    tempRoomId.value.trim() === "" ||
    tempToken.value.trim() === ""
  )
    return;
  username.value = tempUsername.value.trim();
  nickname.value = tempNickname.value.trim();
  roomId.value = tempRoomId.value.trim();
  token.value = tempToken.value.trim();
  localStorage.setItem("username", username.value);
  localStorage.setItem("nickname", nickname.value);
  localStorage.setItem("roomId", roomId.value);
  localStorage.setItem("token", token.value);
  fetchChatHistory();
  connectStomp();
};

// STOMP 연결 설정
const connectStomp = () => {
  stompClient.value = new Client({
    brokerURL: "ws://localhost:8080/chat", // STOMP 서버 주소
    reconnectDelay: 5000, // 재연결 대기 시간 (5초)
    connectHeaders: {
      Authorization: "Bearer " + token.value,
    },
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

  const destination = `/topic/${roomId.value}`; // 동적 roomId 적용
  const headers = {
    Authorization: `Bearer ${token.value}`, // 인증 토큰 추가
  };

  stompClient.value.subscribe(
    destination,
    (message) => {
      console.log("📩 새 메시지 수신:", message.body);
      try {
        const receivedMessage = JSON.parse(message.body);

        if (receivedMessage.messageType) {
          if (receivedMessage.messageType === "CHAT") {
            messages.value = [...messages.value, receivedMessage];
          } else if (receivedMessage.messageType === "READ_STATUS") {
            fetchChatHistory();
          }
        }
      } catch (error) {
        console.error("❌ 메시지 JSON 파싱 오류:", error);
      }
    },
    headers
  ); // ✅ 헤더 추가

  console.log(`✅ 채팅방 구독 완료: ${destination}`);
};

// 메시지 전송
const sendMessage = () => {
  if (message.value.trim() === "" || !stompClient.value.connected) return;
  const msgObject = {
    messageType: "CHAT",
    roomId: roomId.value,
    senderId: username.value,
    senderName: nickname.value,
    content: message.value,
  };
  console.log("📤 메시지 전송:", msgObject);
  stompClient.value.publish({
    destination: `/app/send`,
    body: JSON.stringify(msgObject),
    headers: {
      Authorization: `Bearer ${token.value}`, // 🔥 여기에서 헤더 추가
    },
  });
  message.value = "";
};

const disconnectStomp = async () => {
  try {
    // REST API 호출로 채팅방 연결 끊기
    await axios.post(
      `http://localhost:8080/api/v1/chat/${roomId.value}/${username.value}/disconnect`
    );

    // STOMP 클라이언트 연결 종료
    if (stompClient.value) {
      stompClient.value.deactivate();
      console.log("🔴 STOMP 연결 종료됨");
    }

    // 상태 및 로컬 스토리지 초기화
    username.value = "";
    nickname.value = "";
    roomId.value = "";
    messages.value = [];
    localStorage.removeItem("username");
    localStorage.removeItem("nickname");
    localStorage.removeItem("roomId");
  } catch (error) {
    // API 호출 실패 시 처리
    console.error("채팅방 연결 끊기 실패:", error);

    // 옵션: 에러 발생해도 STOMP 연결은 종료
    if (stompClient.value) {
      stompClient.value.deactivate();
    }
  }
};

// 타임스탬프 포맷
const formatTimestamp = (timestamp) => {
  const date = new Date(timestamp);
  return date.toLocaleString();
};

onMounted(() => {
  if (username.value && nickname.value && roomId.value) {
    fetchChatHistory();
    connectStomp();
  }
});

onUnmounted(() => {
  disconnectStomp();
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
.disconnect-btn {
  display: block;
  width: 100%;
  margin: 10px 0;
  padding: 5px;
  background: #ff4d4d;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}
.disconnect-btn:hover {
  background: #cc0000;
}
</style>
