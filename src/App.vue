<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch } from "vue";
// import downloadIcon from "../public/download.svg"; // SVG 파일 임포트
// import deleteIcon from "../public/delete.svg"; // SVG 파일 임포트

// 두 개의 서버 URL 정의
const LOCAL_SERVER_URL = "ws://localhost:53241/ws";
const REMOTE_SERVER_URL = "ws://downloader.ga111o.com/ws";
const serverType = ref<"local" | "remote">("remote"); // 기본값은 원격 서버
const wsUrl = ref(REMOTE_SERVER_URL); // 기본 URL 설정

// 서버 타입이 변경될 때 WebSocket URL 업데이트
watch(serverType, (newType) => {
  wsUrl.value = newType === "local" ? LOCAL_SERVER_URL : REMOTE_SERVER_URL;

  // 이미 연결되어 있다면 재연결
  if (socket.value) {
    socket.value.close();
    connectWebSocket();
  }
});

// WebSocket connection
const socket = ref<WebSocket | null>(null);
const messages = ref<string[]>([]);
const messagesContainer = ref<HTMLElement | null>(null);
const newUrl = ref("");
const selectedFormat = ref("mp3");
const downloadMode = ref("single"); // 'single' 또는 'playlist' - 기본값은 '단일 영상' 모드
// URL 리스트 관리를 위한 상태 추가 - 썸네일과 제목 정보 추가
const urlList = ref<
  { url: string; selected: boolean; title: string; thumbnail: string }[]
>([]);
// 파일 다운로드를 위한 상태 추가
const fileChunks = ref<Uint8Array[]>([]);
const currentFileMetadata = ref<{
  type: string;
  filename: string;
  format: string;
} | null>(null);
const isDownloading = ref(false);
const isProcessingQueue = ref(false);
const shouldStopProcessing = ref(false); // 다운로드 중단 플래그 추가
const isShowingPlaylistModal = ref(false);
const pendingUrl = ref("");

watch(
  messages,
  () => {
    setTimeout(() => {
      if (messagesContainer.value) {
        messagesContainer.value.scrollTop =
          messagesContainer.value.scrollHeight;
      }
    }, 0);
  },
  { deep: true }
);

// Connect to WebSocket
const connectWebSocket = () => {
  socket.value = new WebSocket(`${wsUrl.value}`);

  socket.value.onopen = () => {
    console.log("WebSocket connected");
  };

  socket.value.onmessage = async (event: MessageEvent) => {
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

      if (data.type === "playlist_videos" && Array.isArray(data.videos)) {
        messages.value.push(
          `플레이리스트에서 ${data.videos.length}개의 영상을 발견했습니다.`
        );

        // 각 비디오에 대한 정보 가져오기
        for (const videoUrl of data.videos) {
          const videoInfo = await getVideoInfo(videoUrl);
          urlList.value.push({
            url: videoUrl,
            selected: true,
            title: videoInfo.title || "제목 없음",
            thumbnail: videoInfo.thumbnail || "",
          });
        }

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

        // 큐에 있는 다음 URL 처리
        processNextInQueue();
      }

      // 에러 처리
      if (data.type === "error") {
        console.error("Error from server:", data.message);
        messages.value.push(`Error: ${data.message}`);
        isDownloading.value = false;

        // 에러가 발생해도 다음 URL 처리
        processNextInQueue();
      }
    } catch (error) {
      console.error("Error in message handler:", error);
      processNextInQueue();
    }
  };

  socket.value.onclose = () => {
    console.log("WebSocket disconnected");
  };

  socket.value.onerror = (error: Event) => {
    console.error("WebSocket error:", error);
  };
};

// URL 추가
const addUrl = async () => {
  if (newUrl.value.trim()) {
    const url = newUrl.value.trim();

    // 비디오 ID와 재생목록 ID가 모두 포함된 URL인지 확인
    if (url.includes("watch?v=") && url.includes("&list=")) {
      // 선택된 다운로드 모드에 따라 처리
      if (downloadMode.value === "playlist") {
        // 재생목록 전체 다운로드
        if (socket.value && socket.value.readyState === WebSocket.OPEN) {
          const message = {
            url: url,
            format: selectedFormat.value,
            type: "playlist",
          };
          socket.value.send(JSON.stringify(message));
          messages.value.push(`플레이리스트 처리 요청: ${url}`);
          newUrl.value = "";
        }
      } else {
        // 단일 비디오만 다운로드 - 플레이리스트 파라미터 제거
        const videoUrl = url.split("&list=")[0];
        const videoInfo = await getVideoInfo(videoUrl);
        urlList.value.push({
          url: videoUrl,
          selected: true,
          title: videoInfo.title || "제목 없음",
          thumbnail: videoInfo.thumbnail || "",
        });
        newUrl.value = "";
      }
      return;
    }

    // 플레이리스트 URL인지 확인 (일반적인 YouTube 플레이리스트 URL 패턴)
    if (
      url.includes("playlist?list=") ||
      url.includes("&list=") ||
      url.includes("/sets/")
    ) {
      // 플레이리스트 처리 요청 전송
      if (socket.value && socket.value.readyState === WebSocket.OPEN) {
        const message = {
          url: url,
          format: selectedFormat.value,
          type: "playlist",
        };
        socket.value.send(JSON.stringify(message));
        messages.value.push(`플레이리스트 처리 요청: ${url}`);
        newUrl.value = "";
      }
    } else {
      // 일반 URL 추가 - 썸네일과 제목 가져오기
      const videoInfo = await getVideoInfo(url);
      urlList.value.push({
        url: url,
        selected: true,
        title: videoInfo.title || "제목 없음",
        thumbnail: videoInfo.thumbnail || "",
      });
      newUrl.value = "";
    }
  }
};

// 유튜브 비디오 정보 가져오기
const getVideoInfo = async (url: string) => {
  try {
    const videoId = extractVideoId(url);
    if (!videoId) {
      return { title: "알 수 없는 비디오", thumbnail: "" };
    }

    // 썸네일 URL 생성 (유튜브는 이 형식의 썸네일 URL을 항상 제공)
    const thumbnailUrl = `https://img.youtube.com/vi/${videoId}/mqdefault.jpg`;

    // oEmbed API를 사용하여 비디오 제목 가져오기
    const response = await fetch(
      `https://www.youtube.com/oembed?url=https://www.youtube.com/watch?v=${videoId}&format=json`
    );

    if (response.ok) {
      const data = await response.json();
      return {
        title: data.title,
        thumbnail: thumbnailUrl,
      };
    } else {
      // API 호출 실패 시 기본값 사용
      return {
        title: "유튜브 비디오",
        thumbnail: thumbnailUrl,
      };
    }
  } catch (error) {
    console.error("비디오 정보를 가져오는 중 오류 발생:", error);
    return { title: "알 수 없는 비디오", thumbnail: "" };
  }
};

// 유튜브 URL에서 비디오 ID 추출
const extractVideoId = (url: string) => {
  // YouTube URL 패턴 확인
  const regExp =
    /^.*(youtu.be\/|v\/|u\/\w\/|embed\/|watch\?v=|\&v=)([^#\&\?]*).*/;
  const match = url.match(regExp);

  if (match && match[2].length === 11) {
    return match[2];
  }

  return null;
};

// 선택한 URL 삭제
const removeSelected = () => {
  urlList.value = urlList.value.filter(
    (item: { url: string; selected: boolean }) => !item.selected
  );
};

// 모든 URL 선택/해제
const toggleSelectAll = (select: boolean) => {
  urlList.value.forEach(
    (item: { url: string; selected: boolean }) => (item.selected = select)
  );
};

// 선택된 URL 처리 시작 또는 중단
const startProcessing = () => {
  // 이미 처리 중이면 중단 신호 설정
  if (isProcessingQueue.value) {
    shouldStopProcessing.value = true;
    messages.value.push("현재 항목 다운로드 완료 후 중단합니다.");
    return;
  }

  // 다운로드 시작
  const selectedUrls = urlList.value.filter(
    (item: { url: string; selected: boolean }) => item.selected
  );
  if (selectedUrls.length === 0) return;

  isProcessingQueue.value = true;
  shouldStopProcessing.value = false; // 중단 플래그 초기화
  processNextInQueue();
};

// 큐에서 다음 URL 처리
const processNextInQueue = () => {
  if (!isProcessingQueue.value) return;

  // 중단 신호가 있고 현재 다운로드 중이 아니면 처리 중단
  if (shouldStopProcessing.value && !isDownloading.value) {
    isProcessingQueue.value = false;
    shouldStopProcessing.value = false;
    messages.value.push("다운로드가 중단되었습니다.");
    return;
  }

  // 이미 다운로드 중이면 리턴
  if (isDownloading.value) return;

  // 선택된 URL 중에서 처리할 다음 URL 찾기
  const nextItemIndex = urlList.value.findIndex(
    (item: { url: string; selected: boolean }) => item.selected
  );

  if (nextItemIndex === -1) {
    // 더 이상 처리할 URL이 없으면 종료
    isProcessingQueue.value = false;
    shouldStopProcessing.value = false;
    messages.value.push("모든 선택된 URL 처리 완료");
    return;
  }

  // 다음 URL 처리
  const item = urlList.value[nextItemIndex];
  sendUrlToServer(item.url);

  // 처리 시작한 항목은 목록에서 제거하거나 선택 해제
  urlList.value.splice(nextItemIndex, 1);
};

// Send URL to server
const sendUrlToServer = (url: string) => {
  if (socket.value && socket.value.readyState === WebSocket.OPEN) {
    const message = {
      url: url,
      format: selectedFormat.value,
    };
    socket.value.send(JSON.stringify(message));
    messages.value.push(`다운로드 요청 전송: ${url} (${selectedFormat.value})`);
  }
};

// 단일 항목 다운로드
const downloadSingleItem = (index: number) => {
  if (isDownloading.value || isProcessingQueue.value) return;

  const item = urlList.value[index];
  if (!item) return;

  // 해당 URL만 서버로 전송
  sendUrlToServer(item.url);

  // 처리 시작한 항목은 목록에서 제거
  urlList.value.splice(index, 1);
};

// 단일 항목 삭제
const removeSingleItem = (index: number) => {
  if (index >= 0 && index < urlList.value.length) {
    urlList.value.splice(index, 1);
  }
};

// 비디오 또는 재생목록 선택 처리
const handlePlaylistChoice = async (choosePlaylist: boolean) => {
  if (!pendingUrl.value) return;

  if (choosePlaylist) {
    // 재생목록 전체 다운로드
    if (socket.value && socket.value.readyState === WebSocket.OPEN) {
      const message = {
        url: pendingUrl.value,
        format: selectedFormat.value,
        type: "playlist",
      };
      socket.value.send(JSON.stringify(message));
      messages.value.push(`플레이리스트 처리 요청: ${pendingUrl.value}`);
    }
  } else {
    // 단일 비디오만 다운로드 - 플레이리스트 파라미터 제거
    const videoUrl = pendingUrl.value.split("&list=")[0];
    const videoInfo = await getVideoInfo(videoUrl);
    urlList.value.push({
      url: videoUrl,
      selected: true,
      title: videoInfo.title || "제목 없음",
      thumbnail: videoInfo.thumbnail || "",
    });
  }

  // 상태 초기화
  isShowingPlaylistModal.value = false;
  pendingUrl.value = "";
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
  <div class="container">
    <!-- 서버 선택 UI 추가 -->
    <div class="download-mode-selector">
      <label class="mode-label">
        <input
          type="radio"
          v-model="serverType"
          value="local"
          name="serverType"
        />
        <span>로컬 서버</span>
      </label>
      <label class="mode-label">
        <input
          type="radio"
          v-model="serverType"
          value="remote"
          name="serverType"
        />
        <span>원격 서버</span>
      </label>
    </div>

    <div class="download-mode-selector">
      <label class="mode-label">
        <input
          type="radio"
          v-model="downloadMode"
          value="single"
          name="downloadMode"
        />
        <span>단일 영상</span>
      </label>
      <label class="mode-label">
        <input
          type="radio"
          v-model="downloadMode"
          value="playlist"
          name="downloadMode"
        />
        <span>재생목록</span>
      </label>
    </div>

    <div class="url-input">
      <input
        v-model="newUrl"
        @keyup.enter="addUrl"
        placeholder="YouTube URL을 입력하세요..."
        class="url-input-field"
      />
      <select v-model="selectedFormat" class="format-selector">
        <option value="mp3">MP3</option>
        <option value="mp4">MP4</option>
      </select>
      <button @click="addUrl">추가</button>
    </div>

    <p v-if="messages.length > 0" class="message-header">logs</p>

    <div v-if="messages.length > 0" class="messages" ref="messagesContainer">
      <div
        v-for="(message, index) in messages"
        :key="index"
        class="message-item"
      >
        {{ message }}
      </div>
    </div>

    <div v-if="isDownloading" class="download-status">
      <p>다운로드 중... 잠시만 기다려주세요</p>
    </div>

    <div v-if="urlList.length > 0" class="url-list">
      <div class="url-list-header">
        <div class="url-list-header-buttons">
          <button @click="toggleSelectAll(true)">전체 선택</button>
          <button @click="toggleSelectAll(false)">전체 해제</button>
        </div>

        <div class="url-list-header-buttons">
          <button @click="removeSelected" class="delete-button">
            선택한 항목 삭제
          </button>
          <button
            @click="startProcessing"
            :class="['download-button', isProcessingQueue ? 'stop-button' : '']"
          >
            {{ isProcessingQueue ? "다운로드 중단" : "선택한 항목 다운로드" }}
          </button>
        </div>
      </div>

      <div v-for="(item, index) in urlList" :key="index" class="url-item">
        <input
          type="checkbox"
          v-model="item.selected"
          class="download-checkbox"
        />
        <div class="video-thumbnail">
          <img
            :src="item.thumbnail"
            alt="비디오 썸네일"
            v-if="item.thumbnail"
          />
          <div class="no-thumbnail" v-else></div>
        </div>
        <div class="video-info">
          <div class="video-title">{{ item.title }}</div>
          <div class="video-url">
            {{ item.url.replace("https://www.youtube.com/watch?v=", "") }}
          </div>
        </div>
        <div class="item-actions">
          <button
            @click="downloadSingleItem(index)"
            :disabled="isDownloading || isProcessingQueue"
            class="action-button each-download-button"
            tabindex="-1"
          >
            <svg
              viewBox="0 0 24 24"
              fill="none"
              xmlns="http://www.w3.org/2000/svg"
            >
              <g id="SVGRepo_bgCarrier" stroke-width="0"></g>
              <g
                id="SVGRepo_tracerCarrier"
                stroke-linecap="round"
                stroke-linejoin="round"
              ></g>
              <g id="SVGRepo_iconCarrier">
                <path
                  d="M12 7L12 14M12 14L15 11M12 14L9 11"
                  stroke="#1C274C"
                  stroke-width="1.9200000000000004"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                ></path>
                <path
                  d="M16 17H12H8"
                  stroke="#1C274C"
                  stroke-width="1.9200000000000004"
                  stroke-linecap="round"
                ></path>
                <path
                  d="M22 12C22 16.714 22 19.0711 20.5355 20.5355C19.0711 22 16.714 22 12 22C7.28595 22 4.92893 22 3.46447 20.5355C2 19.0711 2 16.714 2 12C2 7.28595 2 4.92893 3.46447 3.46447C4.92893 2 7.28595 2 12 2C16.714 2 19.0711 2 20.5355 3.46447C21.5093 4.43821 21.8356 5.80655 21.9449 8"
                  stroke="#1C274C"
                  stroke-width="1.9200000000000004"
                  stroke-linecap="round"
                ></path>
              </g>
            </svg>
          </button>
          <button
            @click="removeSingleItem(index)"
            class="action-button delete-button"
            tabindex="-1"
          >
            <svg
              viewBox="0 0 24 24"
              width="20"
              height="20"
              xmlns="http://www.w3.org/2000/svg"
            >
              <path
                fill="currentColor"
                fill-rule="evenodd"
                d="M12,2 C17.5228475,2 22,6.4771525 22,12 C22,17.5228475 17.5228475,22 12,22 C6.4771525,22 2,17.5228475 2,12 C2,6.4771525 6.4771525,2 12,2 Z M12,4 C7.581722,4 4,7.581722 4,12 C4,16.418278 7.581722,20 12,20 C16.418278,20 20,16.418278 20,12 C20,7.581722 16.418278,4 12,4 Z M7.29325,7.29325 C7.65417308,6.93232692 8.22044527,6.90456361 8.61296051,7.20996006 L8.70725,7.29325 L12.00025,10.58625 L15.29325,7.29325 C15.68425,6.90225 16.31625,6.90225 16.70725,7.29325 C17.0681731,7.65417308 17.0959364,8.22044527 16.7905399,8.61296051 L16.70725,8.70725 L13.41425,12.00025 L16.70725,15.29325 C17.09825,15.68425 17.09825,16.31625 16.70725,16.70725 C16.51225,16.90225 16.25625,17.00025 16.00025,17.00025 C15.7869167,17.00025 15.5735833,16.9321944 15.3955509,16.796662 L15.29325,16.70725 L12.00025,13.41425 L8.70725,16.70725 C8.51225,16.90225 8.25625,17.00025 8.00025,17.00025 C7.74425,17.00025 7.48825,16.90225 7.29325,16.70725 C6.93232692,16.3463269 6.90456361,15.7800547 7.20996006,15.3875395 L7.29325,15.29325 L10.58625,12.00025 L7.29325,8.70725 C6.90225,8.31625 6.90225,7.68425 7.29325,7.29325 Z"
              ></path>
            </svg>
          </button>
        </div>
      </div>
    </div>

    <!-- 재생목록/단일 비디오 선택 모달 -->
    <div v-if="isShowingPlaylistModal" class="modal-overlay">
      <div class="modal-content">
        <h3>다운로드 옵션</h3>
        <p>이 URL은 비디오와 재생목록을 모두 포함하고 있습니다.</p>
        <div class="modal-buttons">
          <button @click="handlePlaylistChoice(false)" class="modal-button">
            단일 비디오만 다운로드
          </button>
          <button
            @click="handlePlaylistChoice(true)"
            class="modal-button playlist-button"
          >
            재생목록 전체 다운로드
          </button>
        </div>
        <button
          @click="
            isShowingPlaylistModal = false;
            pendingUrl = '';
          "
          class="cancel-button"
        >
          취소
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  width: 100%;
  max-width: 350px;
  margin: 0 auto;
  padding: 0;
}

.download-checkbox {
  width: 20px;
  height: 20px;
  margin: 0 8px 0 0;
}

.url-input {
  display: flex;
  width: 350px;
  margin: 10px 0;
}

.url-input input {
  flex: 1;
  margin-right: 5px;
  padding: 8px;
}

.format-selector {
  margin-right: 5px;
}

.url-list {
  margin: 15px 0;
  border: 1px solid #ddd;
  border-radius: 4px;
  overflow: hidden;
}

.url-list-header {
  background: #f5f5f5;
  padding: 8px;
  display: flex;
  gap: 10px;
}

.url-item {
  padding: 8px;
  border-bottom: 1px solid #eee;
  display: flex;
  align-items: center;
}

.url-item:last-child {
  border-bottom: none;
}

.url-list-header-buttons button {
  width: 100%;
  font-size: 0.9em;
}

.video-thumbnail {
  width: 80px;
  height: 68px;
  margin-right: 3px;
  overflow: hidden;
  border-radius: 4px;
  background-color: #f0f0f0;
  display: flex;
}

.video-thumbnail img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.no-thumbnail {
  width: 100%;
  height: 100%;
  background-color: #ddd;
}

.video-info {
  flex: 1;
  overflow: hidden;
}

.video-title {
  font-weight: bold;
  margin-bottom: 3px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 0.8em;
}

.video-url {
  font-size: 0.85em;
  color: #666;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  text-align: left;
}

.item-actions {
  display: flex;
  flex-direction: column;
  gap: 3px;
  margin-left: 3px;
}

.action-button {
  width: 30px;
  height: 30px;
  padding: 3px;
  font-size: 0.8em;
  white-space: nowrap;
}

.download-button {
  background-color: #4caf50;
  color: white;
  border: none;
  width: 100%;
  font-size: 0.9em;
}

.download-button:hover:not(:disabled) {
  background-color: #388e3c;
}

.each-download-button {
  color: #4caf50;
  border: none;
  padding: 5px;
}

.each-download-button:hover:not(:disabled) {
  color: #388e3c;
}

.delete-button {
  color: #f44336;
  border: none;
  width: 100%;
  font-size: 0.9em;
}

.delete-button:hover {
  color: #d32f2f;
}

.messages {
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 10px;
  max-height: 200px;
  overflow-y: auto;
}

.message-header {
  font-size: 0.9em;
  text-align: left;
  margin: 0;
  margin-left: 10px;
}

.message-item {
  font-size: 0.9em;
  text-align: left;
  border-bottom: 1px solid #ddd;
  padding: 3px;
}

.download-status {
  margin-top: 15px;
  padding: 10px;
  background-color: #f0f8ff;
  border-radius: 4px;
  text-align: center;
}

button,
select {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: #f5f5f5;
  cursor: pointer;
}

button:hover {
  background: #e0e0e0;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background-color: white;
  padding: 20px;
  border-radius: 8px;
  max-width: 90%;
  width: 350px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.modal-content h3 {
  margin-top: 0;
  text-align: center;
}

.modal-content p {
  margin-bottom: 15px;
  text-align: center;
}

.modal-buttons {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 15px;
}

.modal-button {
  padding: 10px;
  text-align: center;
  cursor: pointer;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: #f5f5f5;
}

.modal-button:hover {
  background-color: #e0e0e0;
}

.playlist-button {
  background-color: #4caf50;
  color: white;
}

.playlist-button:hover {
  background-color: #388e3c;
}

.cancel-button {
  width: 100%;
  padding: 8px;
  background-color: #f1f1f1;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
}

.cancel-button:hover {
  background-color: #e0e0e0;
}

.download-mode-selector {
  display: flex;
  margin-bottom: 4px;
  gap: 15px;
  padding: 0;
}

.download-mode-selector input {
  margin: 0;
}

.mode-label {
  display: flex;
  align-items: center;
  cursor: pointer;
  user-select: none;
}

.mode-label input {
  margin-right: 5px;
}

.mode-label span {
  font-size: 0.9em;
}

.stop-button {
  background-color: #f44336;
}

.stop-button:hover {
  background-color: #d32f2f;
}
</style>
