<script setup lang="ts">
import { ref, reactive, watch } from "vue";
import axios from "axios";

const apiUrl = "http://localhost:53241";
// const apiUrl = "http://223.194.20.119:53241";

// 다운로드 요청 상태 - 단일 URL 대신 여러 URL 관리
const newUrl = ref("");
const format = ref("mp3");
const formats = ["mp3", "mp4"];
const isLoading = ref(false);
const error = ref("");

// URL 입력이 완료되면 자동으로 큐에 추가하는 함수
const handleUrlInput = (event: Event) => {
  // Enter 키를 누르거나 붙여넣기 후 포커스가 떠날 때 실행
  if (
    newUrl.value &&
    ((event instanceof KeyboardEvent && event.key === "Enter") ||
      event.type === "blur")
  ) {
    addToDownloadQueue();
  }
};

// 유튭 비디오 정보 상태
const videoInfo = reactive({
  title: "",
  thumbnail: "",
  isLoading: false,
  error: "",
});

// 기존 상태 객체들을 제거하고 큐 기반 처리로 변경
const downloadQueue = reactive<
  {
    url: string;
    format: string;
    id?: string;
    status: string;
    message: string;
    isReady: boolean;
    title: string;
    thumbnail: string;
    error?: string;
  }[]
>([]);

const addToDownloadQueue = async () => {
  if (!newUrl.value) {
    error.value = "YouTube URL을 입력해주세요";
    return;
  }

  try {
    error.value = "";

    // 비디오 정보 가져오기
    const videoId = extractVideoId(newUrl.value);
    if (!videoId) {
      error.value = "유효한 YouTube URL이 아닙니다";
      return;
    }

    const thumbnail = `https://img.youtube.com/vi/${videoId}/maxresdefault.jpg`;

    // 큐에 추가
    downloadQueue.push({
      url: newUrl.value,
      format: format.value,
      status: "pending",
      message: "대기 중",
      isReady: false,
      title: `Video ID: ${videoId}`,
      thumbnail: thumbnail,
    });

    // 입력창 초기화
    newUrl.value = "";
  } catch (err: any) {
    error.value = `URL 추가 오류: ${err.message}`;
  }
};

const startDownload = async (index: number) => {
  const item = downloadQueue[index];
  if (item.status === "processing" || item.status === "completed") return;

  try {
    item.status = "processing";
    item.message = "처리 중...";

    const response = await axios.post(`${apiUrl}/download`, {
      url: item.url,
      format: item.format,
    });

    item.id = response.data.download_id;
    item.status = response.data.status;
    item.message = response.data.message;

    // 상태 확인 시작
    checkItemStatus(index);
  } catch (err: any) {
    item.status = "error";
    item.error = `다운로드 요청 오류: ${
      err.response?.data?.detail || err.message
    }`;
    item.message = "오류 발생";
  }
};

const startAllDownloads = () => {
  downloadQueue.forEach((item, index) => {
    if (item.status === "pending") {
      startDownload(index);
    }
  });
};

const checkItemStatus = async (index: number) => {
  const item = downloadQueue[index];
  if (!item || !item.id) return;

  try {
    const response = await axios.get(`${apiUrl}/status/${item.id}`);
    item.status = response.data.status;
    item.message = response.data.message;

    if (response.data.status === "completed") {
      item.isReady = true;
    } else if (response.data.status === "processing") {
      // 아직 처리 중이면 1초 후 다시 확인
      setTimeout(() => checkItemStatus(index), 1000);
    }
  } catch (err: any) {
    item.status = "error";
    item.error = `상태 확인 오류: ${err.response?.data?.detail || err.message}`;
  }
};

const downloadFile = (index: number) => {
  const item = downloadQueue[index];
  if (!item || !item.id || !item.isReady) return;

  window.location.href = `${apiUrl}/download/${item.id}`;
};

const removeFromQueue = (index: number) => {
  downloadQueue.splice(index, 1);
};

const fetchVideoInfo = async () => {
  if (!newUrl.value) {
    videoInfo.title = "";
    videoInfo.thumbnail = "";
    return;
  }

  try {
    videoInfo.isLoading = true;
    videoInfo.error = "";

    const videoId = extractVideoId(newUrl.value);

    if (!videoId) {
      videoInfo.error = "!videoId" + newUrl.value;
      videoInfo.title = "";
      videoInfo.thumbnail = "";
      return;
    }

    videoInfo.thumbnail = `https://img.youtube.com/vi/${videoId}/maxresdefault.jpg`;

    videoInfo.title = `Video ID: ${videoId}`;

    const img = new Image();
    img.onload = () => {};
    img.onerror = () => {
      videoInfo.thumbnail = `https://img.youtube.com/vi/${videoId}/hqdefault.jpg`;
    };
    img.src = videoInfo.thumbnail;
  } catch (err: any) {
    videoInfo.error = err.message;
    videoInfo.title = "";
    videoInfo.thumbnail = "";
  } finally {
    videoInfo.isLoading = false;
  }
};

const extractVideoId = (url: string): string | null => {
  // https://www.youtube.com/watch?v={VIDEO_ID}
  const regExp =
    /^.*((youtu.be\/)|(v\/)|(\/u\/\w\/)|(embed\/)|(watch\?))\??v?=?([^#&?]*).*/;
  const match = url.match(regExp);

  return match && match[7].length === 11 ? match[7] : null;
};

watch(newUrl, (newUrl) => {
  if (newUrl) {
    fetchVideoInfo();
  } else {
    videoInfo.title = "";
    videoInfo.thumbnail = "";
  }
});
</script>

<template>
  <div class="app-container">
    <div class="download-form">
      <div class="input-group">
        <input
          type="text"
          v-model="newUrl"
          placeholder="YouTube URL을 입력하세요"
          :disabled="isLoading"
          @keyup.enter="handleUrlInput"
          @blur="handleUrlInput"
        />

        <select v-model="format" :disabled="isLoading">
          <option v-for="fmt in formats" :key="fmt" :value="fmt">
            {{ fmt }}
          </option>
        </select>
      </div>

      <div v-if="error" class="error-message">{{ error }}</div>

      <!-- 다운로드 목록 -->
      <div v-if="downloadQueue.length > 0" class="download-queue">
        <div class="queue-header">
          <h3>다운로드 대기열 ({{ downloadQueue.length }}개)</h3>
          <button
            @click="startAllDownloads"
            class="start-all-button"
            v-if="downloadQueue.some((item) => item.status === 'pending')"
          >
            모두 다운로드 시작
          </button>
        </div>

        <div
          v-for="(item, index) in downloadQueue"
          :key="index"
          class="queue-item"
        >
          <div class="item-info">
            <img :src="item.thumbnail" alt="썸네일" class="item-thumbnail" />
            <div class="item-details">
              <h4>{{ item.title }}</h4>
              <p class="item-format">형식: {{ item.format }}</p>
              <p class="item-status">
                상태: {{ item.status }} - {{ item.message }}
              </p>
              <p v-if="item.error" class="error-message">{{ item.error }}</p>
            </div>
          </div>

          <div class="item-actions">
            <button
              v-if="item.status === 'pending'"
              @click="startDownload(index)"
              class="start-button"
            >
              시작
            </button>
            <button
              v-if="item.isReady"
              @click="downloadFile(index)"
              class="download-button"
            >
              다운로드
            </button>
            <button @click="removeFromQueue(index)" class="remove-button">
              삭제
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
  text-align: center;
}

.download-form {
  margin: 2rem 0;
}

.input-group {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

input,
select,
button {
  padding: 0.5rem;
  font-size: 1rem;
}

input {
  flex: 1;
}

.error-message {
  color: #ff0000;
  margin-top: 0.5rem;
}

.download-queue {
  margin-top: 2rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: #f9f9f9;
}

.queue-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 1rem;
  border-bottom: 1px solid #ddd;
}

.start-all-button {
  background-color: #42b883;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

.queue-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  border-bottom: 1px solid #ddd;
}

.queue-item:last-child {
  border-bottom: none;
}

.item-info {
  display: flex;
  align-items: center;
  text-align: left;
  flex: 1;
}

.item-thumbnail {
  width: 120px;
  height: 90px;
  object-fit: cover;
  border-radius: 4px;
  margin-right: 1rem;
}

.item-details {
  flex: 1;
}

.item-details h4 {
  margin: 0 0 0.5rem 0;
}

.item-format,
.item-status {
  margin: 0.2rem 0;
  font-size: 0.9rem;
}

.item-actions {
  display: flex;
  gap: 0.5rem;
}

.start-button,
.download-button,
.remove-button {
  padding: 0.3rem 0.8rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.start-button {
  background-color: #4299e1;
  color: white;
}

.download-button {
  background-color: #42b883;
  color: white;
}

.remove-button {
  background-color: #e53e3e;
  color: white;
}
</style>
