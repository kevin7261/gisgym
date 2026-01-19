<template>
  <div class="rag-container">
    <div class="rag-card">
      <h1 class="title">📄 PDF 轉向量庫 (RAG)</h1>
      <p class="description">
        上傳您的 PDF/DOCX 文件，透過雲端 Server 進行 Embedding 運算，並下載打包好的 FAISS 索引。
      </p>

      <!-- 上傳區域 -->
      <div
        class="upload-area"
        @click="$refs.fileInput.click()"
        @dragover.prevent="handleDragOver"
        @dragleave.prevent="handleDragLeave"
        @drop.prevent="handleDrop"
        :class="{ 'drag-over': isDragOver }"
      >
        <input
          ref="fileInput"
          type="file"
          accept=".pdf,.docx,.txt"
          @change="handleFileSelect"
          style="display: none;"
        />
        <span>{{ uploadText }}</span>
      </div>

      <!-- 檔案名稱顯示 -->
      <div v-if="selectedFile" class="file-info">
        已選擇: {{ selectedFile.name }}
      </div>

      <!-- 提交按鈕 -->
      <button
        class="submit-btn"
        @click="processFile"
        :disabled="!selectedFile || isProcessing"
      >
        <span v-if="isProcessing" class="loader"></span>
        <span>{{ buttonText }}</span>
      </button>

      <!-- 狀態訊息 -->
      <div v-if="statusMessage" class="status" :class="statusClass">
        {{ statusMessage }}
      </div>

      <!-- 下載連結 -->
      <div v-if="downloadUrl" class="download-link">
        <a :href="downloadUrl" :download="downloadFileName" @click="handleDownload">
          🔗 點擊下載 {{ downloadFileName }}
        </a>
      </div>

      <!-- 注意事項 -->
      <div class="note">
        ⚠️ 注意：若雲端主機處於休眠狀態，首次執行可能需要等待 1-2 分鐘喚醒並載入模型，請耐心等候。
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue';

export default {
  name: 'RagView',

  setup() {
    // API 網址
    const API_URL = 'https://kevin7261-gisgym.hf.space/process';

    // 狀態變數
    const selectedFile = ref(null);
    const isProcessing = ref(false);
    const statusMessage = ref('');
    const statusType = ref(''); // 'success', 'error', 'info'
    const downloadUrl = ref('');
    const downloadFileName = ref('');
    const isDragOver = ref(false);

    // 計算屬性
    const uploadText = computed(() => {
      return isDragOver.value ? '放開以上傳檔案' : '點擊或拖曳檔案至此';
    });

    const buttonText = computed(() => {
      return isProcessing.value ? '正在處理中...' : '開始轉換並下載';
    });

    const statusClass = computed(() => {
      return `status-${statusType.value}`;
    });

    // 檔案選擇處理
    const handleFileSelect = (event) => {
      const files = event.target.files;
      if (files && files.length > 0) {
        selectedFile.value = files[0];
        statusMessage.value = '';
        downloadUrl.value = '';
      }
    };

    // 拖曳處理
    const handleDragOver = () => {
      isDragOver.value = true;
    };

    const handleDragLeave = () => {
      isDragOver.value = false;
    };

    const handleDrop = (event) => {
      isDragOver.value = false;
      const files = event.dataTransfer.files;
      if (files && files.length > 0) {
        selectedFile.value = files[0];
        statusMessage.value = '';
        downloadUrl.value = '';
      }
    };

    // 處理檔案
    const processFile = async () => {
      if (!selectedFile.value) {
        alert('請先選擇一個檔案！');
        return;
      }

      // 重置狀態
      downloadUrl.value = '';
      downloadFileName.value = '';
      isProcessing.value = true;
      statusType.value = 'info';
      statusMessage.value = '正在上傳並等待雲端運算...\n(如果是冷啟動，可能需要 1~3 分鐘)';

      try {
        const formData = new FormData();
        formData.append('file', selectedFile.value);

        // 發送 POST 請求
        const response = await fetch(API_URL, {
          method: 'POST',
          body: formData
        });

        if (!response.ok) {
          const errorText = await response.text();
          throw new Error(`伺服器錯誤: ${response.status} - ${errorText}`);
        }

        // 取得 Blob (二進制檔案)
        const blob = await response.blob();

        // 建立下載連結
        downloadUrl.value = window.URL.createObjectURL(blob);

        // 嘗試從 Header 取得檔名，或使用預設檔名
        downloadFileName.value = selectedFile.value.name.split('.')[0] + '_faiss.zip';
        const contentDisposition = response.headers.get('Content-Disposition');
        if (contentDisposition && contentDisposition.indexOf('attachment') !== -1) {
          const filenameRegex = /filename[^;=\n]*=((['"]).*?\2|[^;\n]*)/;
          const matches = filenameRegex.exec(contentDisposition);
          if (matches != null && matches[1]) {
            downloadFileName.value = matches[1].replace(/['"]/g, '');
          }
        }

        statusType.value = 'success';
        statusMessage.value = '✅ 轉換成功！請點擊下方連結下載檔案。';

      } catch (error) {
        console.error(error);
        statusType.value = 'error';
        statusMessage.value = `❌ 發生錯誤: ${error.message}\n請確認伺服器網址是否正確，或伺服器是否正在重啟。`;
      } finally {
        isProcessing.value = false;
      }
    };

    // 處理下載
    const handleDownload = () => {
      // 下載完成後清理 URL
      setTimeout(() => {
        if (downloadUrl.value) {
          window.URL.revokeObjectURL(downloadUrl.value);
        }
      }, 100);
    };

    return {
      selectedFile,
      isProcessing,
      statusMessage,
      statusClass,
      downloadUrl,
      downloadFileName,
      isDragOver,
      uploadText,
      buttonText,
      handleFileSelect,
      handleDragOver,
      handleDragLeave,
      handleDrop,
      processFile,
      handleDownload
    };
  }
};
</script>

<style scoped>
:root {
  --primary-color: #4f46e5;
  --primary-hover: #4338ca;
  --bg-color: #f3f4f6;
  --card-bg: #ffffff;
  --text-color: #1f2937;
}

.rag-container {
  min-height: 100vh;
  background-color: var(--bg-color);
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
  font-family: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
}

.rag-card {
  background-color: var(--card-bg);
  padding: 2rem;
  border-radius: 12px;
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
  width: 100%;
  max-width: 500px;
  text-align: center;
}

.title {
  margin-top: 0;
  color: var(--primary-color);
  font-size: 1.5rem;
  margin-bottom: 1rem;
}

.description {
  color: #6b7280;
  font-size: 0.95rem;
  line-height: 1.5;
  margin-bottom: 1.5rem;
}

.upload-area {
  border: 2px dashed #d1d5db;
  border-radius: 8px;
  padding: 2rem;
  margin: 1.5rem 0;
  transition: all 0.3s;
  cursor: pointer;
  background-color: transparent;
}

.upload-area:hover,
.upload-area.drag-over {
  border-color: var(--primary-color);
  background-color: #f9fafb;
}

.file-info {
  margin-top: 10px;
  margin-bottom: 10px;
  font-weight: 600;
  color: var(--primary-color);
  font-size: 0.9rem;
  word-break: break-all;
}

.submit-btn {
  background-color: var(--primary-color);
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  border-radius: 6px;
  cursor: pointer;
  width: 100%;
  transition: background-color 0.2s;
  font-weight: 600;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.submit-btn:hover:not(:disabled) {
  background-color: var(--primary-hover);
}

.submit-btn:disabled {
  background-color: #9ca3af;
  cursor: not-allowed;
}

.loader {
  border: 3px solid #f3f3f3;
  border-top: 3px solid var(--primary-color);
  border-radius: 50%;
  width: 20px;
  height: 20px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.status {
  margin-top: 1rem;
  font-size: 0.9rem;
  white-space: pre-line;
  min-height: 1.2em;
  padding: 0.75rem;
  border-radius: 6px;
}

.status-info {
  background-color: #eff6ff;
  color: #1e40af;
}

.status-success {
  background-color: #f0fdf4;
  color: #15803d;
}

.status-error {
  background-color: #fef2f2;
  color: #dc2626;
}

.download-link {
  margin-top: 1rem;
  padding: 1rem;
  background-color: #f0fdf4;
  border-radius: 6px;
}

.download-link a {
  color: var(--primary-color);
  text-decoration: none;
  font-weight: 600;
  font-size: 1rem;
  display: inline-block;
  padding: 0.5rem 1rem;
  background-color: white;
  border-radius: 6px;
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1);
  transition: all 0.2s;
}

.download-link a:hover {
  background-color: var(--primary-color);
  color: white;
  transform: translateY(-2px);
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.note {
  font-size: 0.8rem;
  color: #dc2626;
  margin-top: 15px;
  background: #fef2f2;
  padding: 10px;
  border-radius: 6px;
  line-height: 1.5;
}

/* 響應式設計 */
@media (max-width: 640px) {
  .rag-card {
    padding: 1.5rem;
  }

  .title {
    font-size: 1.25rem;
  }

  .description {
    font-size: 0.875rem;
  }

  .upload-area {
    padding: 1.5rem;
  }
}
</style>
