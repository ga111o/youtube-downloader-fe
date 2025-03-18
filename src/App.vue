<script setup lang="ts">
import { ref, reactive, watch } from "vue";
import axios from "axios";

const apiUrl = "http://localhost:8000";
// const apiUrl = "http://223.194.20.119:50241";

// 다운로드 요청 상태
const youtubeUrl = ref("");
const format = ref("mp3");
const formats = ["mp3", "mp4"];
const isLoading = ref(false);
const error = ref("");
const downloadState = reactive({
  downloadId: "",
  status: "",
  message: "",
  isReady: false,
});

// 유튭 비디오 정보 상태
const videoInfo = reactive({
  title: "",
  thumbnail: "",
  isLoading: false,
  error: "",
});

const downloadYouTube = async () => {
  if (!youtubeUrl.value) {
    error.value = "YouTube URL을 입력해주세요";
    return;
  }

  try {
    isLoading.value = true;
    error.value = "";

    const response = await axios.post(`${apiUrl}/download`, {
      url: youtubeUrl.value,
      format: format.value,
    });

    downloadState.downloadId = response.data.download_id;
    downloadState.status = response.data.status;
    downloadState.message = response.data.message;
    downloadState.isReady = false;

    checkDownloadStatus();
  } catch (err: any) {
    error.value = `다운로드 요청 오류: ${
      err.response?.data?.detail || err.message
    }`;
  } finally {
    isLoading.value = false;
  }
};

const checkDownloadStatus = async () => {
  if (!downloadState.downloadId) return;

  try {
    const response = await axios.get(
      `${apiUrl}/status/${downloadState.downloadId}`
    );
    downloadState.status = response.data.status;
    downloadState.message = response.data.message;

    if (response.data.status === "completed") {
      downloadState.isReady = true;
    } else if (response.data.status === "processing") {
      // 아직 처리 중이면 3초 후 다시 확인
      setTimeout(checkDownloadStatus, 1000);
    }
  } catch (err: any) {
    error.value = `상태 확인 오류: ${
      err.response?.data?.detail || err.message
    }`;
  }
};

const downloadFile = () => {
  if (!downloadState.downloadId || !downloadState.isReady) return;

  window.location.href = `${apiUrl}/download/${downloadState.downloadId}`;
};

const fetchVideoInfo = async () => {
  if (!youtubeUrl.value) {
    videoInfo.title = "";
    videoInfo.thumbnail = "";
    return;
  }

  try {
    videoInfo.isLoading = true;
    videoInfo.error = "";

    const videoId = extractVideoId(youtubeUrl.value);

    if (!videoId) {
      videoInfo.error = "!videoId" + youtubeUrl.value;
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

watch(youtubeUrl, (newUrl) => {
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
          v-model="youtubeUrl"
          placeholder="YouTube URL을 입력하세요"
          :disabled="isLoading"
        />

        <select v-model="format" :disabled="isLoading">
          <option v-for="fmt in formats" :key="fmt" :value="fmt">
            {{ fmt }}
          </option>
        </select>

        <button @click="downloadYouTube" :disabled="isLoading || !youtubeUrl">
          {{ isLoading ? "처리중..." : "다운로드" }}
        </button>
      </div>

      <!-- 비디오 정보 표시 영역 -->
      <div v-if="videoInfo.isLoading" class="video-info-loading">
        비디오 정보를 불러오는 중...
      </div>
      <div
        v-else-if="videoInfo.title && videoInfo.thumbnail"
        class="video-info"
      >
        <img
          :src="videoInfo.thumbnail"
          alt="비디오 썸네일"
          class="video-thumbnail"
        />
        <h3 class="video-title">{{ videoInfo.title }}</h3>
      </div>
      <div v-if="videoInfo.error" class="error-message">
        {{ videoInfo.error }}
      </div>

      <div v-if="error" class="error-message">{{ error }}</div>
    </div>

    <div v-if="downloadState.downloadId" class="download-status">
      <h3>다운로드 상태</h3>
      <p><strong>상태:</strong> {{ downloadState.status }}</p>
      <p><strong>메시지:</strong> {{ downloadState.message }}</p>

      <button
        v-if="downloadState.isReady"
        @click="downloadFile"
        class="download-button"
      >
        파일 다운로드
      </button>
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

.download-status {
  margin-top: 2rem;
  padding: 1rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: #f9f9f9;
  text-align: left;
}

.download-button {
  background-color: #42b883;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
  margin-top: 1rem;
}

.download-button:hover {
  background-color: #3a9d74;
}

.video-info {
  margin: 1rem 0;
  padding: 1rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: #f9f9f9;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.video-thumbnail {
  max-width: 100%;
  height: auto;
  border-radius: 4px;
  margin-bottom: 1rem;
}

.video-title {
  margin: 0;
  font-size: 1.2rem;
  text-align: center;
}

.video-info-loading {
  margin: 1rem 0;
  font-style: italic;
  color: #666;
}
</style>
