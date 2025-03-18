<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from "vue";

const wsUrl = "ws://localhost:53241/ws";
// const apiUrl = "http://223.194.20.119:53241";

// WebSocket connection
const socket = ref<WebSocket | null>(null);
const messages = ref<string[]>([]);
const inputUrl = ref("");
const selectedFormat = ref("mp3");
// 파일 다운로드를 위한 상태 추가
const fileChunks = ref<Uint8Array[]>([]);
const currentFileMetadata = ref<{
  type: string;
  filename: string;
  format: string;
} | null>(null);
const isDownloading = ref(false);

// Connect to WebSocket
const connectWebSocket = () => {
  socket.value = new WebSocket(`${wsUrl}`);

  socket.value.onopen = () => {
    console.log("WebSocket connected");
  };

  socket.value.onmessage = (event: MessageEvent) => {
    try {
      // 바이너리 데이터인 경우
      if (event.data instanceof Blob) {
        // blob을 arrayBuffer로 변환
        event.data.arrayBuffer().then((buffer) => {
          const chunk = new Uint8Array(buffer);
          fileChunks.value.push(chunk);
          console.log(`Received binary chunk: ${chunk.length} bytes`);
        });
        return;
      }

      // 텍스트 데이터인 경우
      let data;
      try {
        data = JSON.parse(event.data);
      } catch (e) {
        console.error("Invalid JSON data:", event.data);
        messages.value.push(event.data);
        return;
      }

      console.log("Received data:", data);

      if (typeof data === "string") {
        messages.value.push(data);
        return;
      }

      // 메타데이터 수신
      if (data.type === "metadata") {
        console.log("Received metadata:", data);
        currentFileMetadata.value = data;
        fileChunks.value = []; // 새로운 파일 수신 시작
        isDownloading.value = true;
        return;
      }

      // 전송 완료
      if (data.type === "complete" && currentFileMetadata.value) {
        console.log("File transfer complete");

        // 모든 청크 합치기
        let totalLength = 0;
        fileChunks.value.forEach((chunk: Uint8Array) => {
          totalLength += chunk.length;
        });

        const combinedChunks = new Uint8Array(totalLength);
        let offset = 0;
        fileChunks.value.forEach((chunk: Uint8Array) => {
          combinedChunks.set(chunk, offset);
          offset += chunk.length;
        });

        // Blob 생성 및 다운로드
        const blob = new Blob([combinedChunks], {
          type:
            currentFileMetadata.value.format === "mp3"
              ? "audio/mpeg"
              : "video/mp4",
        });

        const url = window.URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = currentFileMetadata.value.filename;
        document.body.appendChild(a);
        a.click();
        window.URL.revokeObjectURL(url);
        document.body.removeChild(a);

        // 상태 초기화
        const filename = currentFileMetadata.value.filename;
        fileChunks.value = [];
        currentFileMetadata.value = null;
        isDownloading.value = false;
        messages.value.push(`다운로드 완료: ${filename}`);
      }

      // 에러 처리
      if (data.type === "error") {
        console.error("Error from server:", data.message);
        messages.value.push(`Error: ${data.message}`);
        isDownloading.value = false;
      }
    } catch (error) {
      console.error("Error in message handler:", error);
    }
  };

  socket.value.onclose = () => {
    console.log("WebSocket disconnected");
  };

  socket.value.onerror = (error: Event) => {
    console.error("WebSocket error:", error);
  };
};

// Send message through WebSocket
const sendMessage = () => {
  if (
    socket.value &&
    socket.value.readyState === WebSocket.OPEN &&
    inputUrl.value
  ) {
    const message = {
      url: inputUrl.value,
      format: selectedFormat.value,
    };
    socket.value.send(JSON.stringify(message));
    messages.value.push(
      `다운로드 요청 전송: ${inputUrl.value} (${selectedFormat.value})`
    );
    inputUrl.value = "";
  }
};

// Connect when component is mounted
onMounted(() => {
  connectWebSocket();
});

// Disconnect when component is unmounted
onBeforeUnmount(() => {
  if (socket.value) {
    socket.value.close();
  }
});
</script>

<template>
  <div class="app">
    <h1>YouTube Downloader</h1>

    <div class="input-area">
      <input
        v-model="inputUrl"
        @keyup.enter="sendMessage"
        placeholder="YouTube URL을 입력하세요..."
      />
      <select v-model="selectedFormat">
        <option value="mp3">MP3</option>
        <option value="mp4">MP4</option>
      </select>
      <button @click="sendMessage" :disabled="isDownloading">다운로드</button>
    </div>

    <div v-if="messages.length > 0" class="messages">
      <div v-for="(message, index) in messages" :key="index" class="message">
        {{ message }}
      </div>
    </div>

    <div v-if="isDownloading" class="downloading">
      <p>다운로드 중... 잠시만 기다려주세요</p>
    </div>
  </div>
</template>

<style scoped>
.app {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
}

.messages {
  height: 300px;
  overflow-y: auto;
  border: 1px solid #ccc;
  margin-bottom: 20px;
  padding: 10px;
  margin-top: 20px;
}

.message {
  margin-bottom: 10px;
  padding: 8px;
  background-color: #f1f1f1;
  border-radius: 4px;
}

.input-area {
  display: flex;
}

input {
  flex: 1;
  padding: 8px;
  margin-right: 10px;
}

button {
  padding: 8px 16px;
}

select {
  padding: 8px;
  margin-right: 10px;
}

.downloading {
  margin-top: 20px;
  padding: 10px;
  background-color: #e8f4f8;
  border-radius: 4px;
  text-align: center;
}
</style>
