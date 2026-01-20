/** * 🎮 操作控制分頁組件 (Control Tab Component) * * 功能說明 (Features): * 1. 🚀
執行下一步：提供圖層處理流程的執行按鈕 * 2. 📊 圖層選擇：顯示當前可操作的圖層 * 3. 🔄
狀態管理：追蹤執行狀態和圖層資訊 * 4. 📱 響應式設計：適配不同螢幕尺寸的顯示需求 * * 技術特點
(Technical Features): * - 使用 Vue 3 Composition API 進行狀態管理 * - 整合 Pinia 狀態管理系統 * -
支援多圖層切換和操作 * * @file ControlTab.vue * @version 1.0.0 * @author Kevin Cheng * @since 1.0.0
*/
<script setup>
  // ==================== 📦 第三方庫引入 (Third-Party Library Imports) ====================

  /**
   * Vue 3 Composition API 核心功能引入
   * 提供響應式數據管理、計算屬性、生命週期鉤子等現代化 Vue 開發功能
   */
  import { ref, computed, watch, nextTick } from 'vue';

  /**
   * Pinia 狀態管理庫引入
   * 提供集中式狀態管理和跨組件數據共享能力
   */
  import { useDataStore } from '@/stores/dataStore.js';

  /**
   * 網格合併和縮減工具函數引入
   * 提供路線合併和網格縮減的核心功能
   */
  import {
    generateDataTableData_Test4 as generateDataTableDataUtil,
    mergeRoutesHorizontal,
    mergeRoutesVertical,
    reduceGrid as reduceGridUtil,
  } from '@/utils/gridMergeReduce.js';

  // ==================== 🏪 狀態管理初始化 (State Management Initialization) ====================

  /**
   * 獲取 Pinia 數據存儲實例
   * 用於訪問全域狀態和圖層數據
   */
  const dataStore = useDataStore();

  // ==================== 📊 響應式狀態定義 (Reactive State Definition) ====================

  /**
   * 📑 當前作用中的圖層分頁 (Active Layer Tab)
   * 追蹤使用者當前選中的圖層分頁
   */
  const activeLayerTab = ref(null);

  /**
   * 🔄 執行計算狀態 (Execution Loading State)
   * 追蹤 executeFunction 執行過程的狀態，用於顯示計算中指示器
   */
  const isExecuting = ref(false);

  // ==================== 🤖 RAG 問答功能 (RAG Q&A) ====================

  /**
   * RAG API 端點
   */
  const RAG_API_URL = 'https://kevin7261-gisgym.hf.space/ask_with_zip';

  /**
   * RAG 向量庫檔案路徑配置（支援開發和生產環境）
   */
  const RAG_ZIP_PATHS = {
    primary: '/gisgym/data/lectures_faiss_db.zip', // 生產環境
    fallback: '/data/lectures_faiss_db.zip', // 開發環境
  };

  const ragQuestion = ref('');
  const ragHistory = ref([]);
  const ragIsLoading = ref(false);
  const ragError = ref('');

  const ragHasApi = computed(() => typeof RAG_API_URL === 'string' && RAG_API_URL.trim());

  const isRagLayer = computed(() => {
    return currentLayer.value && currentLayer.value.layerId === 'test_layer';
  });

  const clearRagHistory = () => {
    ragHistory.value = [];
    ragError.value = '';
  };

  const askRag = async () => {
    const input = ragQuestion.value.trim();
    if (!input || !ragHasApi.value) return;

    ragIsLoading.value = true;
    ragError.value = '';

    try {
      // 從 data 目錄讀取 zip 檔案（嘗試主要路徑，失敗則使用備用路徑）
      let zipResponse;
      try {
        zipResponse = await fetch(RAG_ZIP_PATHS.primary);
        if (!zipResponse.ok) {
          throw new Error(`主要路徑載入失敗: ${RAG_ZIP_PATHS.primary}`);
        }
      } catch (primaryError) {
        console.warn(`⚠️ 主要路徑載入失敗，嘗試備用路徑: ${RAG_ZIP_PATHS.fallback}`);
        zipResponse = await fetch(RAG_ZIP_PATHS.fallback);
        if (!zipResponse.ok) {
          throw new Error(
            `無法載入向量庫檔案。主要路徑: ${RAG_ZIP_PATHS.primary}, 備用路徑: ${RAG_ZIP_PATHS.fallback}`
          );
        }
      }
      const zipBlob = await zipResponse.blob();

      const formData = new FormData();
      formData.append('file', zipBlob, 'lectures_faiss_db.zip');
      formData.append('question', input);

      const response = await fetch(RAG_API_URL, {
        method: 'POST',
        body: formData,
      });

      if (!response.ok) {
        let errorText;
        try {
          const errorData = await response.json();
          errorText = errorData.error || errorData.message || JSON.stringify(errorData);
        } catch {
          errorText = await response.text();
        }
        throw new Error(`HTTP ${response.status}: ${errorText}`);
      }

      const data = await response.json();
      const answer =
        data.answer ||
        data.response ||
        data.output ||
        data.message ||
        data.result ||
        JSON.stringify(data);
      const retrievedChunks = data.retrieved_chunks || data.context || data.chunks || null;

      ragHistory.value.push({
        question: input,
        answer,
        retrievedChunks,
      });

      ragQuestion.value = '';
    } catch (error) {
      ragError.value = `❌ 發生錯誤: ${error.message}`;
    } finally {
      ragIsLoading.value = false;
    }
  };

  // ==================== 📊 計算屬性定義 (Computed Properties Definition) ====================

  /**
   * 獲取所有可見且有資料的圖層 (Get All Visible Layers with Data)
   * 從全域狀態中篩選出可見且已載入資料的圖層
   */
  const visibleLayers = computed(() => {
    const allLayers = dataStore.getAllLayers();
    return allLayers.filter((layer) => layer && layer.visible);
  });

  /**
   * 獲取所有有效的可見圖層（確保每個圖層都有有效的 layerId）
   * 用於模板中的 v-for，避免渲染無效圖層
   */
  const validVisibleLayers = computed(() => {
    return visibleLayers.value.filter((layer) => layer && layer.layerId);
  });

  /**
   * 當前選中的圖層 (Current Selected Layer)
   * 根據 activeLayerTab 獲取當前選中的圖層物件
   */
  const currentLayer = computed(() => {
    if (!activeLayerTab.value || !visibleLayers.value || visibleLayers.value.length === 0) {
      return null;
    }
    return (
      visibleLayers.value.find((layer) => layer && layer.layerId === activeLayerTab.value) || null
    );
  });

  /**
   * 取得圖層完整標題 (包含群組名稱) (Get Layer Full Title with Group Name)
   * 組合群組名稱和圖層名稱，形成完整的圖層標題
   */
  const getLayerFullTitle = (layer) => {
    if (!layer) return { groupName: null, layerName: '未知圖層' };
    const groupName = dataStore.findGroupNameByLayerId(layer.layerId);
    return {
      groupName: groupName,
      layerName: layer.layerName,
    };
  };

  /**
   * 判斷當前圖層是否有 executeFunction 且屬於 Taipei 群組
   */
  const canExecuteLayer = computed(() => {
    if (!currentLayer.value) return false;

    // 檢查圖層是否屬於 Taipei 群組
    const groupName = dataStore.findGroupNameByLayerId(currentLayer.value.layerId);
    if (groupName !== 'Taipei') return false;

    // 檢查是否有 executeFunction
    return (
      currentLayer.value.executeFunction && typeof currentLayer.value.executeFunction === 'function'
    );
  });


  /**
   * 判斷當前圖層是否為 taipei_6_1_test
   * 只有此圖層才顯示"合併一筆路線"按鈕
   *
   * @type {ComputedRef<boolean>}
   * @returns {boolean} 是否為 taipei_6_1_test 圖層
   */
  const isTaipei6_1Test = computed(() => {
    return currentLayer.value && currentLayer.value.layerId === 'taipei_6_1_test';
  });

  /**
   * 📊 判斷是否為 taipei_6_1_test2 圖層 (Check if is taipei_6_1_test2 Layer)
   * 用於控制特定圖層專屬功能的顯示
   *
   * @type {ComputedRef<boolean>}
   * @returns {boolean} 是否為 taipei_6_1_test2 圖層
   */
  const isTaipei6_1Test2 = computed(() => {
    return currentLayer.value && currentLayer.value.layerId === 'taipei_6_1_test2';
  });

  /**
   * 📊 判斷是否為 taipei_6_1_test3 或 taipei_6_1_test4 圖層 (Check if is taipei_6_1_test3 or taipei_6_1_test4 Layer)
   * 用於控制特定圖層專屬功能的顯示（顯示 LayoutGridTab_Test4 的網格資料）
   *
   * @type {ComputedRef<boolean>}
   * @returns {boolean} 是否為 taipei_6_1_test3 或 taipei_6_1_test4 圖層
   */
  const isTaipei6_1Test3 = computed(() => {
    return (
      currentLayer.value &&
      (currentLayer.value.layerId === 'taipei_6_1_test3' ||
        currentLayer.value.layerId === 'taipei_6_1_test4')
    );
  });

  /**
   * 📊 取得 LayoutGridTab_Test2 當前尺寸 (Get LayoutGridTab_Test2 Current Dimensions)
   * 從 dataStore 中獲取 LayoutGridTab_Test2 的當前尺寸（以 pt 為單位）
   *
   * @type {ComputedRef<{x: number, y: number}>}
   * @returns {{x: number, y: number}} 當前尺寸的 x（寬度）和 y（高度）
   */
  const layoutGridTabTest2Dimensions = computed(() => {
    return dataStore.layoutGridTabTest2Dimensions;
  });

  /**
   * 📊 取得 LayoutGridTab_Test2 網格最小尺寸 (Get LayoutGridTab_Test2 Min Cell Dimensions)
   * 從 dataStore 中獲取 LayoutGridTab_Test2 的網格最小尺寸（以 pt 為單位）
   *
   * @type {ComputedRef<{minWidth: number, minHeight: number}>}
   * @returns {{minWidth: number, minHeight: number}} 最小寬度和最小高度
   */
  const layoutGridTabTest2MinCellDimensions = computed(() => {
    return dataStore.layoutGridTabTest2MinCellDimensions;
  });

  /**
   * 📊 取得 LayoutGridTab_Test3 當前尺寸 (Get LayoutGridTab_Test3 Current Dimensions)
   * 從 dataStore 中獲取 LayoutGridTab_Test3 的當前尺寸（以 pt 為單位）
   *
   * @type {ComputedRef<{x: number, y: number}>}
   * @returns {{x: number, y: number}} 當前尺寸的 x（寬度）和 y（高度）
   */
  const layoutGridTabTest3Dimensions = computed(() => {
    return dataStore.layoutGridTabTest3Dimensions;
  });

  /**
   * 📊 取得 LayoutGridTab_Test3 網格最小尺寸 (Get LayoutGridTab_Test3 Min Cell Dimensions)
   * 從 dataStore 中獲取 LayoutGridTab_Test3 的網格最小尺寸（以 pt 為單位）
   *
   * @type {ComputedRef<{minWidth: number, minHeight: number}>}
   * @returns {{minWidth: number, minHeight: number}} 最小寬度和最小高度
   */
  const layoutGridTabTest3MinCellDimensions = computed(() => {
    return dataStore.layoutGridTabTest3MinCellDimensions;
  });

  /**
   * 📊 取得 LayoutGridTab_Test4 當前尺寸 (Get LayoutGridTab_Test4 Current Dimensions)
   * 從 dataStore 中獲取 LayoutGridTab_Test4 的當前尺寸（以 pt 為單位）
   *
   * @type {ComputedRef<{x: number, y: number}>}
   * @returns {{x: number, y: number}} 當前尺寸的 x（寬度）和 y（高度）
   */
  const layoutGridTabTest4Dimensions = computed(() => {
    return dataStore.layoutGridTabTest4Dimensions;
  });

  /**
   * 📊 取得 LayoutGridTab_Test4 網格最小尺寸 (Get LayoutGridTab_Test4 Min Cell Dimensions)
   * 從 dataStore 中獲取 LayoutGridTab_Test4 的網格最小尺寸（以 pt 為單位）
   *
   * @type {ComputedRef<{minWidth: number, minHeight: number}>}
   * @returns {{minWidth: number, minHeight: number}} 最小寬度和最小高度
   */
  const layoutGridTabTest4MinCellDimensions = computed(() => {
    return dataStore.layoutGridTabTest4MinCellDimensions;
  });

  /**
   * 📊 取得 LayoutGridTab_Test4 滑鼠網格座標 (Get LayoutGridTab_Test4 Mouse Grid Coordinate)
   * 從 dataStore 中獲取 LayoutGridTab_Test4 的滑鼠網格座標
   */
  const layoutGridTabTest4MouseGridCoordinate = computed(() => {
    return dataStore.layoutGridTabTest4MouseGridCoordinate;
  });

  /**
   * 📊 取得當前網格實際長寬 (Get Current Grid Actual Dimensions)
   * 從 layoutGridJsonData_Test2（LayoutGridTab_Test2）中獲取當前網格的實際長寬（經過合併和縮減後）
   * 優先從 meta 中讀取，如果沒有則從實際座標計算
   * 使用 computed 確保在數據變化時自動更新
   *
   * @type {ComputedRef<{width: number, height: number}>}
   * @returns {{width: number, height: number}} 當前網格的寬度和高度
   */
  const currentGridDimensions = computed(() => {
    if (!currentLayer.value || !currentLayer.value.layoutGridJsonData_Test2) {
      return { width: 0, height: 0 };
    }

    const layoutData = currentLayer.value.layoutGridJsonData_Test2;

    // 處理兩種格式：Array 或 Object（有 meta）
    let routes;
    let meta;
    if (Array.isArray(layoutData)) {
      routes = layoutData;
      meta = null;
    } else if (layoutData && typeof layoutData === 'object' && Array.isArray(layoutData.routes)) {
      routes = layoutData.routes;
      meta = layoutData.meta || null;
    } else {
      return { width: 0, height: 0 };
    }

    // 優先從 meta 中讀取
    if (meta && typeof meta.gridWidth === 'number' && typeof meta.gridHeight === 'number') {
      return {
        width: meta.gridWidth,
        height: meta.gridHeight,
      };
    }

    // 如果沒有 meta，從實際座標計算
    const usedCols = new Set();
    const usedRows = new Set();

    routes.forEach((route) => {
      const segments = route?.segments || [];
      segments.forEach((seg) => {
        const points = Array.isArray(seg.points) ? seg.points : [];
        points.forEach((pt) => {
          const x = Array.isArray(pt) ? pt[0] : pt.x || 0;
          const y = Array.isArray(pt) ? pt[1] : pt.y || 0;
          usedCols.add(Math.round(x));
          usedRows.add(Math.round(y));
        });
      });
    });

    if (usedCols.size === 0 || usedRows.size === 0) {
      return { width: 0, height: 0 };
    }

    const sortedCols = Array.from(usedCols).sort((a, b) => a - b);
    const sortedRows = Array.from(usedRows).sort((a, b) => a - b);
    const minX = sortedCols[0];
    const maxX = sortedCols[sortedCols.length - 1];
    const minY = sortedRows[0];
    const maxY = sortedRows[sortedRows.length - 1];

    return {
      width: maxX - minX + 1,
      height: maxY - minY + 1,
    };
  });

  /**
   * 📊 取得當前網格實際長寬 (Get Current Grid Actual Dimensions for Test3)
   * 從 layoutGridJsonData_Test3（LayoutGridTab_Test3）中獲取當前網格的實際長寬（經過合併和縮減後）
   * 優先從 meta 中讀取，如果沒有則從實際座標計算
   * 使用 computed 確保在數據變化時自動更新
   *
   * @type {ComputedRef<{width: number, height: number}>}
   * @returns {{width: number, height: number}} 當前網格的寬度和高度
   */
  const currentGridDimensions3 = computed(() => {
    if (!currentLayer.value || !currentLayer.value.layoutGridJsonData_Test3) {
      return { width: 0, height: 0 };
    }

    const layoutData = currentLayer.value.layoutGridJsonData_Test3;

    // 處理兩種格式：Array 或 Object（有 meta）
    let routes;
    let meta;
    if (Array.isArray(layoutData)) {
      routes = layoutData;
      meta = null;
    } else if (layoutData && typeof layoutData === 'object' && Array.isArray(layoutData.routes)) {
      routes = layoutData.routes;
      meta = layoutData.meta || null;
    } else {
      return { width: 0, height: 0 };
    }

    // 優先從 meta 中讀取
    if (meta && typeof meta.gridWidth === 'number' && typeof meta.gridHeight === 'number') {
      return {
        width: meta.gridWidth,
        height: meta.gridHeight,
      };
    }

    // 如果沒有 meta，從實際座標計算
    const usedCols = new Set();
    const usedRows = new Set();

    routes.forEach((route) => {
      const segments = route?.segments || [];
      segments.forEach((seg) => {
        const points = Array.isArray(seg.points) ? seg.points : [];
        points.forEach((pt) => {
          const x = Array.isArray(pt) ? pt[0] : pt.x || 0;
          const y = Array.isArray(pt) ? pt[1] : pt.y || 0;
          usedCols.add(Math.round(x));
          usedRows.add(Math.round(y));
        });
      });
    });

    if (usedCols.size === 0 || usedRows.size === 0) {
      return { width: 0, height: 0 };
    }

    const sortedCols = Array.from(usedCols).sort((a, b) => a - b);
    const sortedRows = Array.from(usedRows).sort((a, b) => a - b);
    const minX = sortedCols[0];
    const maxX = sortedCols[sortedCols.length - 1];
    const minY = sortedRows[0];
    const maxY = sortedRows[sortedRows.length - 1];

    return {
      width: maxX - minX + 1,
      height: maxY - minY + 1,
    };
  });

  /**
   * 📊 取得當前網格實際長寬 (Get Current Grid Actual Dimensions for Test4)
   * 從 layoutGridJsonData_Test4（LayoutGridTab_Test4）中獲取當前網格的實際長寬（經過合併和縮減後）
   * 優先從 meta 中讀取，如果沒有則從實際座標計算
   * 使用 computed 確保在數據變化時自動更新
   *
   * @type {ComputedRef<{width: number, height: number}>}
   * @returns {{width: number, height: number}} 當前網格的寬度和高度
   */
  const currentGridDimensions4 = computed(() => {
    if (!currentLayer.value || !currentLayer.value.layoutGridJsonData_Test4) {
      return { width: 0, height: 0 };
    }

    const layoutData = currentLayer.value.layoutGridJsonData_Test4;

    // 處理兩種格式：Array 或 Object（有 meta）
    let routes;
    let meta;
    if (Array.isArray(layoutData)) {
      routes = layoutData;
      meta = null;
    } else if (layoutData && typeof layoutData === 'object' && Array.isArray(layoutData.routes)) {
      routes = layoutData.routes;
      meta = layoutData.meta || null;
    } else {
      return { width: 0, height: 0 };
    }

    // 優先從 meta 中讀取
    if (meta && typeof meta.gridWidth === 'number' && typeof meta.gridHeight === 'number') {
      return {
        width: meta.gridWidth,
        height: meta.gridHeight,
      };
    }

    // 如果沒有 meta，從實際座標計算
    const usedCols = new Set();
    const usedRows = new Set();

    routes.forEach((route) => {
      const segments = route?.segments || [];
      segments.forEach((seg) => {
        const points = Array.isArray(seg.points) ? seg.points : [];
        points.forEach((pt) => {
          const x = Array.isArray(pt) ? pt[0] : pt.x || 0;
          const y = Array.isArray(pt) ? pt[1] : pt.y || 0;
          usedCols.add(Math.round(x));
          usedRows.add(Math.round(y));
        });
      });
    });

    if (usedCols.size === 0 || usedRows.size === 0) {
      return { width: 0, height: 0 };
    }

    const sortedCols = Array.from(usedCols).sort((a, b) => a - b);
    const sortedRows = Array.from(usedRows).sort((a, b) => a - b);
    const minX = sortedCols[0];
    const maxX = sortedCols[sortedCols.length - 1];
    const minY = sortedRows[0];
    const maxY = sortedRows[sortedRows.length - 1];

    return {
      width: maxX - minX + 1,
      height: maxY - minY + 1,
    };
  });

  // ==================== 🔧 核心功能函數定義 (Core Function Definitions) ====================

  /**
   * 📑 設定作用中圖層分頁 (Set Active Layer Tab)
   * 切換當前選中的圖層分頁
   */
  const setActiveLayerTab = (layerId) => {
    activeLayerTab.value = layerId;
  };

  /**
   * 執行當前圖層的 executeFunction
   */
  const executeLayerFunction = async () => {
    if (!currentLayer.value || !currentLayer.value.executeFunction) {
      console.warn('當前圖層沒有 executeFunction');
      return;
    }

    // Taipei 流程：taipei_1_0 使用 geojsonData，後續流程使用 layoutGridJsonData / spaceNetworkGridJsonData
    // 測試圖層：可能使用 jsonData
    const isTaipeiLayer = currentLayer.value.layerId?.startsWith('taipei_');
    const jsonData = isTaipeiLayer
      ? currentLayer.value.geojsonData ||
        currentLayer.value.layoutGridJsonData ||
        currentLayer.value.spaceNetworkGridJsonData
      : currentLayer.value.geojsonData ||
        currentLayer.value.layoutGridJsonData ||
        currentLayer.value.spaceNetworkGridJsonData ||
        currentLayer.value.jsonData;
    if (!jsonData) {
      const missingFields = isTaipeiLayer
        ? 'geojsonData / layoutGridJsonData / spaceNetworkGridJsonData'
        : 'geojsonData / layoutGridJsonData / spaceNetworkGridJsonData / jsonData';
      console.warn(`當前圖層沒有 ${missingFields}`);
      return;
    }

    isExecuting.value = true;

    try {
      // 等待 UI 更新以顯示"計算中"畫面
      await nextTick();

      // 執行函數（同步執行）
      currentLayer.value.executeFunction(jsonData);

      // 稍微延遲後關閉，確保用戶能看到"計算中"畫面
      setTimeout(() => {
        isExecuting.value = false;
      }, 300);
    } catch (error) {
      console.error('執行圖層函數時發生錯誤:', error);
      isExecuting.value = false;
    }
  };


  // ==================== 👀 響應式監聽器 (Reactive Watchers) ====================

  /**
   * 記錄上一次的圖層列表用於比較變化
   * 用於偵測新增的圖層並自動切換到最新圖層
   */
  const previousLayers = ref([]);

  /**
   * 👀 監聽可見圖層變化，自動切換到新開啟的圖層分頁
   */
  watch(
    () => visibleLayers.value,
    (newLayers) => {
      // 確保 newLayers 是有效的陣列
      if (!Array.isArray(newLayers)) {
        activeLayerTab.value = null;
        previousLayers.value = [];
        return;
      }

      // 如果沒有可見圖層，清除選中的分頁
      if (newLayers.length === 0) {
        activeLayerTab.value = null;
        previousLayers.value = [];
        return;
      }

      // 確保所有圖層都有有效的 layerId
      const validLayers = newLayers.filter((layer) => layer && layer.layerId);

      // 如果沒有有效圖層，清除選中的分頁
      if (validLayers.length === 0) {
        activeLayerTab.value = null;
        previousLayers.value = [];
        return;
      }

      // 使用 nextTick 確保在 DOM 更新後再進行狀態更新
      nextTick(() => {
        // 找出新增的圖層（比較新舊圖層列表）
        const previousLayerIds = (previousLayers.value || [])
          .filter((layer) => layer && layer.layerId)
          .map((layer) => layer.layerId);
        const newLayerIds = validLayers.map((layer) => layer.layerId);
        const addedLayerIds = newLayerIds.filter((id) => !previousLayerIds.includes(id));

        // 如果有新增的圖層，自動切換到最新新增的圖層
        if (addedLayerIds.length > 0) {
          const newestAddedLayerId = addedLayerIds[addedLayerIds.length - 1];
          if (validLayers.find((layer) => layer.layerId === newestAddedLayerId)) {
            activeLayerTab.value = newestAddedLayerId;
          }
        }
        // 如果當前沒有選中分頁，或選中的分頁不在可見列表中，選中第一個
        else if (
          !activeLayerTab.value ||
          !validLayers.find((layer) => layer.layerId === activeLayerTab.value)
        ) {
          if (validLayers[0] && validLayers[0].layerId) {
            activeLayerTab.value = validLayers[0].layerId;
          }
        }

        // 更新記錄的圖層列表（只記錄有效的圖層）
        previousLayers.value = [...validLayers];
      });
    },
    { deep: false, immediate: true }
  );


  /**
   * 🎲 隨機產生權重 (Randomize Weights)
   * 根據指定的權重值和機率分佈重新產生所有權重
   * 同時更新 layoutGridJsonData_Test 和 dataTableData
   */
  const randomizeWeights = () => {
    if (!currentLayer.value) {
      console.warn('當前圖層不存在');
      return;
    }

    const layoutData = currentLayer.value.layoutGridJsonData_Test;
    if (!Array.isArray(layoutData)) {
      console.warn('layoutGridJsonData_Test 不是 Array');
      return;
    }

    // 權重值和機率分佈
    const WEIGHT_VALUES = [1, 2, 3, 4, 5, 6, 7, 8, 9];
    const WEIGHT_PROBS = [729, 512, 343, 216, 125, 64, 27, 8, 1];

    // 計算總機率
    const totalProb = WEIGHT_PROBS.reduce((sum, prob) => sum + prob, 0);

    // 產生隨機權重的函數
    const generateRandomWeight = () => {
      const random = Math.random() * totalProb;
      let cumulative = 0;
      for (let i = 0; i < WEIGHT_VALUES.length; i++) {
        cumulative += WEIGHT_PROBS[i];
        if (random <= cumulative) {
          return WEIGHT_VALUES[i];
        }
      }
      return WEIGHT_VALUES[0]; // 預設返回最小值
    };

    // 遍歷所有路線，重新產生 station_weights
    layoutData.forEach((route) => {
      if (!route || !Array.isArray(route.segments)) return;

      route.segments.forEach((seg) => {
        if (!seg || !Array.isArray(seg.station_weights)) return;

        // 重新產生每個 station_weight 的權重值
        seg.station_weights.forEach((weightInfo) => {
          if (weightInfo && typeof weightInfo.weight === 'number') {
            weightInfo.weight = generateRandomWeight();
          }
        });
      });
    });

    // 🔄 重新生成 dataTableData（基於更新後的權重）
    const dataTableData = [];

    // 輔助函數：從 node 物件中提取 station_name
    const getStationName = (node) => {
      if (!node) return '';
      return node.station_name || node.tags?.station_name || node.tags?.name || '';
    };

    const getStationId = (node) => {
      if (!node) return '';
      return node.station_id || node.tags?.station_id || '';
    };

    // 盡量用穩定且唯一的 key（優先 node.id，其次 station_id，其次 station_name + grid）
    const getNodeKey = (node) => {
      if (!node) return 'node:unknown';
      if (Number.isFinite(node.id)) return `id:${node.id}`;
      const sid = getStationId(node);
      if (sid) return `station_id:${sid}`;
      const name = getStationName(node) || 'unknown';
      const x = node.x_grid ?? node.tags?.x_grid ?? '';
      const y = node.y_grid ?? node.tags?.y_grid ?? '';
      return `name:${name}|x:${x}|y:${y}`;
    };

    // key -> { node, weights: number[], routeName: string, routeColor: string }
    const nodeAdj = new Map();

    const ensureNodeEntry = (node, routeName, routeColor) => {
      const key = `${routeName}|${getNodeKey(node)}`;
      if (!nodeAdj.has(key)) {
        nodeAdj.set(key, { node, weights: [], routeName, routeColor: routeColor || '' });
      } else {
        // 用資訊更完整的 node 覆蓋（避免先遇到空物件）
        const cur = nodeAdj.get(key);
        const curName = getStationName(cur.node);
        const newName = getStationName(node);
        if (!curName && newName) cur.node = node;
        if (!cur.routeColor && routeColor) cur.routeColor = routeColor;
      }
      return key;
    };

    const addWeightToNode = (node, weight, routeName, routeColor) => {
      if (!node) return;
      const key = ensureNodeEntry(node, routeName, routeColor);
      if (typeof weight === 'number' && Number.isFinite(weight)) {
        nodeAdj.get(key).weights.push(weight);
      }
    };

    // 遍歷所有路線和 segments，收集權重
    for (const route of layoutData) {
      const routeName = route.route_name || 'Unknown';
      const defaultRouteColor = route.original_props?.colour || route.color || '#999999';
      const segments = route.segments || [];

      for (let segIndex = 0; segIndex < segments.length; segIndex++) {
        const seg = segments[segIndex];
        const routeColor =
          seg.way_properties?.tags?.color || seg.way_properties?.tags?.colour || defaultRouteColor;
        const nodes = seg.nodes || [];
        const propertiesStart = seg.properties_start;
        const propertiesEnd = seg.properties_end;
        const stationWeights = Array.isArray(seg.station_weights) ? seg.station_weights : null;

        if (stationWeights && stationWeights.length > 0) {
          // 處理 station_weights（兩個黑點之間一個權重）
          for (let wIndex = 0; wIndex < stationWeights.length; wIndex++) {
            const wInfo = stationWeights[wIndex];
            const startIdx = Number.isFinite(wInfo?.start_idx) ? wInfo.start_idx : null;
            const endIdx = Number.isFinite(wInfo?.end_idx) ? wInfo.end_idx : null;
            const weight = wInfo?.weight;

            const startNode =
              startIdx === 0 ? propertiesStart || nodes[0] : nodes[startIdx] || nodes[0];
            const endNode =
              endIdx === nodes.length - 1
                ? propertiesEnd || nodes[endIdx]
                : nodes[endIdx] || nodes[nodes.length - 1];

            // 兩端節點都要收到這條邊的 weight
            addWeightToNode(startNode, weight, routeName, routeColor);
            addWeightToNode(endNode, weight, routeName, routeColor);
          }
        }
      }
    }

    // 產生 table rows：每個節點 1 row，取最小的兩個 weight（由小到大）
    let rowIndex = 1;
    for (const entry of nodeAdj.values()) {
      const node = entry.node;
      const weights = (entry.weights || []).filter(
        (w) => typeof w === 'number' && Number.isFinite(w)
      );
      weights.sort((a, b) => a - b);

      // 依需求：每筆一定要有 2 個與該黑點相連的 weight（不足 2 的通常是路線端點，直接略過）
      if (weights.length < 2) continue;

      const w1 = weights[0];
      const w2 = weights[1];

      const stationId = getStationId(node);
      const xGrid = node?.x_grid ?? node?.tags?.x_grid ?? null;
      const yGrid = node?.y_grid ?? node?.tags?.y_grid ?? null;

      dataTableData.push({
        '#': rowIndex++,
        route_name: entry.routeName || '',
        route_color: entry.routeColor || '',
        station_id: stationId || '',
        station_name: getStationName(node),
        node_type: node?.node_type ?? '',
        x_grid: xGrid,
        y_grid: yGrid,
        weight_1: w1,
        weight_2: w2,
        合併: 'X',
        合併2: 'X',
      });
    }

    // 排序：先看 weight_1，再看 weight_2（都由小到大）
    dataTableData.sort((a, b) => {
      const a1 = a.weight_1 ?? Number.POSITIVE_INFINITY;
      const b1 = b.weight_1 ?? Number.POSITIVE_INFINITY;
      if (a1 !== b1) return a1 - b1;
      const a2 = a.weight_2 ?? Number.POSITIVE_INFINITY;
      const b2 = b.weight_2 ?? Number.POSITIVE_INFINITY;
      return a2 - b2;
    });

    // 重新編號（排序後更新 # 欄位）
    dataTableData.forEach((row, index) => {
      row['#'] = index + 1;
    });

    // 🔄 更新 dataTableData
    currentLayer.value.dataTableData = dataTableData;

    // 🔄 用新引用觸發 LayoutGridTab_Test 的 deep watch，讓權重更新後自動重繪
    currentLayer.value.layoutGridJsonData_Test = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test)
    );

    // 🔄 將更新後的 layoutGridJsonData_Test 也複製到 layoutGridJsonData_Test2
    currentLayer.value.layoutGridJsonData_Test2 = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test)
    );

    // 🔄 將更新後的 layoutGridJsonData_Test 也複製到 layoutGridJsonData_Test3
    currentLayer.value.layoutGridJsonData_Test3 = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test)
    );

    console.log(
      '已重新產生所有權重值並更新 dataTableData、layoutGridJsonData_Test2 和 layoutGridJsonData_Test3'
    );
  };

  /**
   * 🔗 合併一筆路線 (Merge One Route)
   * 依據 dataTableData 裡面的順序，找到第一筆符合條件的項目，將其合併改成 "V"
   * 略過條件：
   * - node_type = 'connect' 的項目
   * - weight_1 與 weight_2 不同的項目
   * 每次點擊必須執行一筆，如果當前筆不符合條件就找下一筆，直到找到或全部執行完
   *
   * 🎯 新功能：實際刪除黑點並合併路段
   * 1. 找到對應的黑點在 layoutGridJsonData_Test 中的位置
   * 2. 把經過該點的兩段路（帶各自 weight）合併成一段
   * 3. 從 points 陣列中刪除該點
   * 4. 更新所有 station_weights 的索引
   */
  const mergeOneRoute = (gap = 0) => {
    if (!currentLayer.value) return;

    const layoutData = currentLayer.value.layoutGridJsonData_Test;
    if (!Array.isArray(layoutData)) {
      console.warn('layoutGridJsonData_Test 不是 Array（目前只支援 2-5 routes array 格式）');
      return;
    }

    // ✅ 直接從 station_weights 找「連續兩段 weight 差 <= gap」的中間點，不再依賴 station_id/station_name 來定位 points 索引
    // 規則：
    // - Math.abs(w1.weight - w2.weight) <= gap（允許權重差在 gap 範圍內）
    // - w1.end_idx === w2.start_idx（共用中間點）
    // - 中間點不是 connect（避免刪紅點/轉乘點）
    let merged = false;
    let mergedRouteName = '';

    const isConnectNodeAt = (seg, idx) => {
      const hasValue = (v) => v !== undefined && v !== null;
      const node = Array.isArray(seg.nodes) ? seg.nodes[idx] : null;
      const pt = Array.isArray(seg.points) ? seg.points[idx] : null;
      const ptProps = Array.isArray(pt) && pt.length > 2 ? pt[2] : {};
      const tags = ptProps?.tags || node?.tags || {};

      const nodeType =
        node?.node_type ||
        ptProps?.node_type ||
        tags?.node_type ||
        (hasValue(node?.connect_number) ? 'connect' : '');
      const hasConnectNumber =
        hasValue(node?.connect_number) ||
        hasValue(tags?.connect_number) ||
        hasValue(ptProps?.connect_number) ||
        hasValue(ptProps?.tags?.connect_number);
      return nodeType === 'connect' || hasConnectNumber;
    };

    /**
     * 判斷是否為真正的車站（有 station_name 的點）
     * 只有真正的車站才可被合併刪除
     * 幾何轉折點（無 station_name）和 connect 節點不應被刪除
     */
    const isRealStation = (seg, idx) => {
      const node = Array.isArray(seg.nodes) ? seg.nodes[idx] : null;
      const pt = Array.isArray(seg.points) ? seg.points[idx] : null;
      const ptProps = Array.isArray(pt) && pt.length > 2 ? pt[2] : {};
      const tags = ptProps?.tags || node?.tags || {};

      // 有 station_name 才算真正的車站
      const hasStationName = !!(
        node?.station_name ||
        ptProps?.station_name ||
        tags?.station_name ||
        ptProps?.tags?.station_name
      );

      return hasStationName;
    };

    /**
     * 判斷某個點是否為幾何轉折點（前後線段方向不同）
     * 如果是轉折點，刪除後需要保留座標以維持路線形狀
     */
    const isBendPoint = (points, idx) => {
      if (idx <= 0 || idx >= points.length - 1) return false;

      const prev = points[idx - 1];
      const curr = points[idx];
      const next = points[idx + 1];

      const px = Array.isArray(prev) ? prev[0] : prev.x || 0;
      const py = Array.isArray(prev) ? prev[1] : prev.y || 0;
      const cx = Array.isArray(curr) ? curr[0] : curr.x || 0;
      const cy = Array.isArray(curr) ? curr[1] : curr.y || 0;
      const nx = Array.isArray(next) ? next[0] : next.x || 0;
      const ny = Array.isArray(next) ? next[1] : next.y || 0;

      // 計算前段向量和後段向量
      const dx1 = cx - px;
      const dy1 = cy - py;
      const dx2 = nx - cx;
      const dy2 = ny - cy;

      // 如果向量方向不同（不共線），就是轉折點
      // 使用叉積判斷：如果叉積不為 0，表示不共線
      const crossProduct = dx1 * dy2 - dy1 * dx2;
      const epsilon = 0.001; // 容許微小誤差

      return Math.abs(crossProduct) > epsilon;
    };

    for (const route of layoutData) {
      if (merged) break;
      const routeName = route?.route_name || '';
      const segments = route?.segments || [];

      for (const seg of segments) {
        if (merged) break;
        const points = Array.isArray(seg.points) ? seg.points : [];
        const weights = Array.isArray(seg.station_weights) ? seg.station_weights : [];
        if (points.length < 3 || weights.length < 2) continue;

        for (let i = 0; i < weights.length - 1; i++) {
          const w1 = weights[i];
          const w2 = weights[i + 1];
          if (!w1 || !w2) continue;
          // 使用 gap 參數：允許 weight 差 <= gap
          const weightDiff = Math.abs((w1.weight || 0) - (w2.weight || 0));
          if (weightDiff > gap) continue;
          if (w1.end_idx !== w2.start_idx) continue;

          const midIdx = w1.end_idx;
          // 不刪端點，避免破壞線段
          if (midIdx <= 0 || midIdx >= points.length - 1) continue;
          // 不刪 connect/轉乘點
          if (isConnectNodeAt(seg, midIdx)) continue;
          // 不刪幾何轉折點（只刪真正的車站）
          if (!isRealStation(seg, midIdx)) continue;

          // 1) 合併兩段 weight：把 w1 end 延伸到 w2 end，並移除 w2
          w1.end_idx = w2.end_idx;
          weights.splice(i + 1, 1);

          // 2) 檢查是否為幾何轉折點
          const isBend = isBendPoint(points, midIdx);

          if (isBend) {
            // 如果是轉折點，保留座標但移除車站屬性
            // 將該點轉換為純幾何點 [x, y]
            const pt = points[midIdx];
            const x = Array.isArray(pt) ? pt[0] : pt.x || 0;
            const y = Array.isArray(pt) ? pt[1] : pt.y || 0;
            points[midIdx] = [x, y]; // 只保留座標，不保留屬性

            // 🎯 將 nodes 對應位置標記為幾何轉折點（不刪除，保持索引一致）
            if (Array.isArray(seg.nodes)) {
              // 確保 nodes 數組長度與 points 一致
              while (seg.nodes.length < points.length) {
                seg.nodes.push({});
              }
              // 將該位置標記為幾何點，node_type: 'line' 表示非車站點
              seg.nodes[midIdx] = { node_type: 'line' };
            }

            // ⚠️ 注意：因為 points 沒有刪除（只是改成純座標），所以 station_weights 的索引不需要調整
          } else {
            // 如果不是轉折點，直接刪除該點
            seg.points.splice(midIdx, 1);
            if (Array.isArray(seg.nodes) && seg.nodes.length > midIdx) {
              seg.nodes.splice(midIdx, 1);
            }

            // 3) 修正所有 station_weights 的索引（刪除點後，midIdx 之後的索引都要 -1）
            for (const w of weights) {
              if (w.start_idx > midIdx) w.start_idx--;
              if (w.end_idx > midIdx) w.end_idx--;
            }
          }

          merged = true;
          mergedRouteName = routeName;
          break;
        }
      }
    }

    if (!merged) {
      console.log('所有項目都已處理完畢，沒有需要合併的項目');
      return;
    }

    // 在 dataTableData 中把最可能對應的那一筆設成 V（維持 UI 的「合併欄位」行為）
    if (Array.isArray(currentLayer.value.dataTableData)) {
      const row = currentLayer.value.dataTableData.find(
        (r) =>
          r &&
          r.合併 !== 'V' &&
          r.node_type !== 'connect' &&
          Math.abs((r.weight_1 || 0) - (r.weight_2 || 0)) <= gap &&
          String(r.route_name || '') === String(mergedRouteName || '')
      );
      if (row) row.合併 = 'V';
    }

    // 🔄 用新引用觸發 LayoutGridTab_Test 的 deep watch，讓點消失後自動重繪
    currentLayer.value.layoutGridJsonData_Test = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test)
    );

    // 🔄 同步更新到 layoutGridJsonData_Test2
    currentLayer.value.layoutGridJsonData_Test2 = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test)
    );
  };

  /**
   * 🔗 合併一筆路線2 (Merge One Route 2)
   * - 依據 DataTableTab 目前的 dataTableData 順序，找到第一筆可合併的項目（使用「合併2」欄位記錄）
   * - 使用該筆的 route_name + weight_1/weight_2（必要時再用 station_id/station_name）去 layoutGridJsonData_Test2 定位並合併
   * - 合併成功後：將該 row 的 合併2 設為 "V"
   *
   * @param {number} gap - 允許的權重差
   * @param {string} direction - 方向篩選：'V' (垂直線) 或 'H' (水平線) 或 null (不篩選)
   * @returns {boolean} 是否有成功合併
   */
  const mergeOneRoute2 = (gap = 0, direction = null) => {
    if (!currentLayer.value) return false;

    const tableData = currentLayer.value.dataTableData;
    if (!Array.isArray(tableData) || tableData.length === 0) {
      console.warn('dataTableData 為空，無法依表格順序合併');
      return false;
    }

    const layoutData2 = currentLayer.value.layoutGridJsonData_Test2;
    if (!Array.isArray(layoutData2)) {
      console.warn('layoutGridJsonData_Test2 不是 Array（目前只支援 2-5 routes array 格式）');
      return false;
    }

    const isConnectNodeAt = (seg, idx) => {
      const hasValue = (v) => v !== undefined && v !== null;
      const node = Array.isArray(seg.nodes) ? seg.nodes[idx] : null;
      const pt = Array.isArray(seg.points) ? seg.points[idx] : null;
      const ptProps = Array.isArray(pt) && pt.length > 2 ? pt[2] : {};
      const tags = ptProps?.tags || node?.tags || {};

      const nodeType =
        node?.node_type ||
        ptProps?.node_type ||
        tags?.node_type ||
        (hasValue(node?.connect_number) ? 'connect' : '');
      const hasConnectNumber =
        hasValue(node?.connect_number) ||
        hasValue(tags?.connect_number) ||
        hasValue(ptProps?.connect_number) ||
        hasValue(ptProps?.tags?.connect_number);
      return nodeType === 'connect' || hasConnectNumber;
    };

    const isRealStation = (seg, idx) => {
      const node = Array.isArray(seg.nodes) ? seg.nodes[idx] : null;
      const pt = Array.isArray(seg.points) ? seg.points[idx] : null;
      const ptProps = Array.isArray(pt) && pt.length > 2 ? pt[2] : {};
      const tags = ptProps?.tags || node?.tags || {};

      const hasStationName = !!(
        node?.station_name ||
        ptProps?.station_name ||
        tags?.station_name ||
        ptProps?.tags?.station_name
      );
      return hasStationName;
    };

    const isBendPoint = (points, idx) => {
      if (idx <= 0 || idx >= points.length - 1) return false;

      const prev = points[idx - 1];
      const curr = points[idx];
      const next = points[idx + 1];

      const px = Array.isArray(prev) ? prev[0] : prev.x || 0;
      const py = Array.isArray(prev) ? prev[1] : prev.y || 0;
      const cx = Array.isArray(curr) ? curr[0] : curr.x || 0;
      const cy = Array.isArray(curr) ? curr[1] : curr.y || 0;
      const nx = Array.isArray(next) ? next[0] : next.x || 0;
      const ny = Array.isArray(next) ? next[1] : next.y || 0;

      const dx1 = cx - px;
      const dy1 = cy - py;
      const dx2 = nx - cx;
      const dy2 = ny - cy;

      const crossProduct = dx1 * dy2 - dy1 * dx2;
      const epsilon = 0.001;
      return Math.abs(crossProduct) > epsilon;
    };

    const matchesTableWeights = (wA, wB, segW1, segW2) => {
      // 允許交換順序
      return (wA === segW1 && wB === segW2) || (wA === segW2 && wB === segW1);
    };

    const getStationMetaAt = (seg, idx) => {
      const node = Array.isArray(seg.nodes) ? seg.nodes[idx] : null;
      const pt = Array.isArray(seg.points) ? seg.points[idx] : null;
      const ptProps = Array.isArray(pt) && pt.length > 2 ? pt[2] : {};
      const tags = ptProps?.tags || node?.tags || {};

      const stationId =
        node?.station_id ||
        ptProps?.station_id ||
        tags?.station_id ||
        ptProps?.tags?.station_id ||
        '';
      const stationName =
        node?.station_name ||
        ptProps?.station_name ||
        tags?.station_name ||
        ptProps?.tags?.station_name ||
        '';

      return {
        station_id: String(stationId || ''),
        station_name: String(stationName || ''),
      };
    };

    // 依 DataTable 的順序逐筆嘗試合併：
    // 若某筆 row 找不到對應可合併點，不能直接 return false（否則「執行完成2」會提早停止）
    // 而是跳過它，繼續往後找下一筆。
    const candidateRows = tableData.filter(
      (r) =>
        r &&
        r.node_type !== 'connect' &&
        r.合併2 !== 'V' &&
        r.合併2 !== 'F' &&
        Math.abs((r.weight_1 || 0) - (r.weight_2 || 0)) <= gap &&
        // 如果有指定方向，只處理符合方向的節點
        (direction === null || !r['V/H'] || r['V/H'] === direction)
    );

    if (candidateRows.length === 0) {
      return false;
    }

    for (const targetRow of candidateRows) {
      const targetRouteName = String(targetRow.route_name || '');
      const targetW1 = Number(targetRow.weight_1);
      const targetW2 = Number(targetRow.weight_2);
      const targetStationId = String(targetRow.station_id || '');
      const targetStationName = String(targetRow.station_name || '');

      let merged = false;

      for (const route of layoutData2) {
        if (merged) break;
        const routeName = String(route?.route_name || '');

        // 如果表格有 route_name，就先限制只在該路線中找
        if (targetRouteName && routeName && routeName !== targetRouteName) continue;

        const segments = route?.segments || [];
        for (const seg of segments) {
          if (merged) break;

          const points = Array.isArray(seg.points) ? seg.points : [];
          const weights = Array.isArray(seg.station_weights) ? seg.station_weights : [];
          if (points.length < 3 || weights.length < 2) continue;

          for (let i = 0; i < weights.length - 1; i++) {
            const w1 = weights[i];
            const w2 = weights[i + 1];
            if (!w1 || !w2) continue;

            const ww1 = Number(w1.weight);
            const ww2 = Number(w2.weight);
            if (!Number.isFinite(ww1) || !Number.isFinite(ww2)) continue;

            // 需要對應表格的那一筆（weight_1/weight_2）
            if (!matchesTableWeights(targetW1, targetW2, ww1, ww2)) continue;

            // 沿用原本合併規則：gap 允許範圍內、且兩段 weight 是連續的中間點
            const weightDiff = Math.abs((w1.weight || 0) - (w2.weight || 0));
            if (weightDiff > gap) continue;
            if (w1.end_idx !== w2.start_idx) continue;

            const midIdx = w1.end_idx;
            if (midIdx <= 0 || midIdx >= points.length - 1) continue; // 不刪端點
            if (isConnectNodeAt(seg, midIdx)) continue; // 不刪 connect
            if (!isRealStation(seg, midIdx)) continue; // 只刪真正車站

            // 如果表格提供 station_id/station_name，就再做一次精準比對
            if (targetStationId || targetStationName) {
              const meta = getStationMetaAt(seg, midIdx);
              if (targetStationId && meta.station_id && meta.station_id !== targetStationId)
                continue;
              if (targetStationName && meta.station_name && meta.station_name !== targetStationName)
                continue;
            }

            // 1) 合併兩段 weight：把 w1 end 延伸到 w2 end，並移除 w2
            w1.end_idx = w2.end_idx;
            weights.splice(i + 1, 1);

            // 2) 檢查是否為幾何轉折點
            const isBend = isBendPoint(points, midIdx);

            if (isBend) {
              const pt = points[midIdx];
              const x = Array.isArray(pt) ? pt[0] : pt.x || 0;
              const y = Array.isArray(pt) ? pt[1] : pt.y || 0;
              points[midIdx] = [x, y];

              // 🎯 將 nodes 對應位置標記為幾何轉折點（不刪除，保持索引一致）
              if (Array.isArray(seg.nodes)) {
                // 確保 nodes 數組長度與 points 一致
                while (seg.nodes.length < points.length) {
                  seg.nodes.push({});
                }
                // 將該位置標記為幾何點，node_type: 'line' 表示非車站點
                seg.nodes[midIdx] = { node_type: 'line' };
              }
              // points 未刪除 => station_weights 索引不需要調整
            } else {
              seg.points.splice(midIdx, 1);
              if (Array.isArray(seg.nodes) && seg.nodes.length > midIdx) {
                seg.nodes.splice(midIdx, 1);
              }
              // 3) 修正索引
              for (const w of weights) {
                if (w.start_idx > midIdx) w.start_idx--;
                if (w.end_idx > midIdx) w.end_idx--;
              }
            }

            merged = true;
            break;
          }
        }
      }

      if (merged) {
        // 記錄在 DataTable 的「合併2」欄位
        targetRow.合併2 = 'V';

        // 觸發重繪：只更新 Test2（LayoutGridTab_Test2 的 watch 會自動監聽並重繪）
        // 重繪時會自動更新最小尺寸（在 drawGridNodes 函數中）
        currentLayer.value.layoutGridJsonData_Test2 = JSON.parse(
          JSON.stringify(currentLayer.value.layoutGridJsonData_Test2)
        );

        return true;
      }

      // 這筆 row 在 layoutGridJsonData_Test2 找不到可合併點：標記為 F，避免「執行完成2」一直卡在同一筆
      targetRow.合併2 = 'F';
    }

    return false;
  };

  /**
   * 📉 縮減網格2 (Reduce Grid 2)
   * 專門針對 layoutGridJsonData_Test2 的縮減網格功能
   * 刪除整個 col 或 row 沒有黑點或路線的網格，並調整座標讓網格大小縮減
   */
  const reduceGrid2 = () => {
    if (!currentLayer.value) return;

    const layoutData = currentLayer.value.layoutGridJsonData_Test2;
    if (!layoutData) {
      console.warn('layoutGridJsonData_Test2 為空');
      return;
    }

    // 處理兩種格式：Array 或 Object（有 meta）
    let routes;
    let meta;
    if (Array.isArray(layoutData)) {
      routes = layoutData;
      meta = null;
    } else if (layoutData && typeof layoutData === 'object' && Array.isArray(layoutData.routes)) {
      routes = layoutData.routes;
      meta = layoutData.meta || null;
    } else {
      console.warn(
        'layoutGridJsonData_Test2 格式不支援（目前只支援 Array 或 {routes, meta} 格式）'
      );
      return;
    }

    // 1. 統計所有使用的座標
    const usedCols = new Set();
    const usedRows = new Set();

    routes.forEach((route) => {
      const segments = route?.segments || [];
      segments.forEach((seg) => {
        const points = Array.isArray(seg.points) ? seg.points : [];
        points.forEach((pt) => {
          const x = Array.isArray(pt) ? pt[0] : pt.x || 0;
          const y = Array.isArray(pt) ? pt[1] : pt.y || 0;
          usedCols.add(Math.round(x));
          usedRows.add(Math.round(y));
        });
      });
    });

    // 2. 找出空 col/row 並建立映射表（舊座標 -> 新座標）
    const colMap = new Map(); // 舊 x -> 新 x
    const rowMap = new Map(); // 舊 y -> 新 y

    if (usedCols.size === 0 || usedRows.size === 0) {
      console.warn('沒有找到任何使用的座標');
      return;
    }

    // 計算 col 映射（刪除空的 col）
    const sortedCols = Array.from(usedCols).sort((a, b) => a - b);
    const minX = sortedCols[0];
    const maxX = sortedCols[sortedCols.length - 1];

    let newX = 0;
    for (let oldX = minX; oldX <= maxX; oldX++) {
      if (usedCols.has(oldX)) {
        colMap.set(oldX, newX);
        newX++;
      }
    }

    // 計算 row 映射（刪除空的 row）
    const sortedRows = Array.from(usedRows).sort((a, b) => a - b);
    const minY = sortedRows[0];
    const maxY = sortedRows[sortedRows.length - 1];

    let newY = 0;
    for (let oldY = minY; oldY <= maxY; oldY++) {
      if (usedRows.has(oldY)) {
        rowMap.set(oldY, newY);
        newY++;
      }
    }

    const removedCols = maxX - minX + 1 - newX;
    const removedRows = maxY - minY + 1 - newY;

    if (removedCols === 0 && removedRows === 0) {
      console.log('沒有空的 col/row 需要刪除');
      return;
    }

    console.log(`📉 縮減網格2：刪除 ${removedCols} 個空 col，${removedRows} 個空 row`);

    // 3. 調整所有點的座標
    routes.forEach((route) => {
      const segments = route?.segments || [];
      segments.forEach((seg) => {
        const points = Array.isArray(seg.points) ? seg.points : [];
        points.forEach((pt, idx) => {
          if (Array.isArray(pt)) {
            const oldX = pt[0];
            const oldY = pt[1];
            const newXCoord = colMap.get(Math.round(oldX)) ?? oldX;
            const newYCoord = rowMap.get(Math.round(oldY)) ?? oldY;

            if (pt.length > 2) {
              // [x, y, props] 格式，保留 props
              points[idx] = [newXCoord, newYCoord, pt[2]];
            } else {
              // [x, y] 格式
              points[idx] = [newXCoord, newYCoord];
            }
          } else if (pt && typeof pt === 'object') {
            // {x, y} 格式
            const oldX = pt.x || 0;
            const oldY = pt.y || 0;
            pt.x = colMap.get(Math.round(oldX)) ?? oldX;
            pt.y = rowMap.get(Math.round(oldY)) ?? oldY;
          }
        });
      });
    });

    // 4. 更新 meta.gridWidth 和 meta.gridHeight（如果存在）
    if (meta) {
      meta.gridWidth = newX;
      meta.gridHeight = newY;
      if (typeof meta.width === 'number') meta.width = newX;
      if (typeof meta.height === 'number') meta.height = newY;
    }

    console.log(`✅ 網格已縮減2：新尺寸 ${newX} x ${newY}`);

    // 5. 觸發資料更新（只更新 Test2，不影響 Test）
    // LayoutGridTab_Test2 的 watch 會監聽 layoutGridJsonData_Test2 的變化並自動重繪
    // 重繪時會自動更新最小尺寸（在 drawGridNodes 函數中）
    currentLayer.value.layoutGridJsonData_Test2 = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test2)
    );
  };

  /**
   * 合併一筆路線2 後立刻縮減網格
   * @param {number} gap - 允許的權重差
   * @param {string} direction - 方向篩選：'V' (垂直線) 或 'H' (水平線) 或 null (不篩選)
   */
  const mergeOneRoute2AndReduce = (gap = 0, direction = null) => {
    const didMerge = mergeOneRoute2(gap, direction);
    if (didMerge) {
      reduceGrid2(); // 使用專門的 reduceGrid2
    }
  };

  /**
   * 🔗 執行完成2 (Merge All Routes 2)
   * 重複執行「合併一筆路線2」直到沒有更多可合併的項目
   * @param {number} gap - 允許的權重差
   * @param {string} direction - 方向篩選：'V' (垂直線) 或 'H' (水平線) 或 null (不篩選)
   */
  const mergeAllRoutes2AndReduce = (gap = 0, direction = null) => {
    if (!currentLayer.value) return;

    let mergedCount = 0;
    // 依需求：每合併一筆就立刻縮減網格
    while (mergeOneRoute2(gap, direction)) {
      mergedCount++;
      reduceGrid2(); // 使用專門的 reduceGrid2
    }

    if (mergedCount > 0) {
      console.log(`🎉 執行完成2！共合併 ${mergedCount} 筆（每筆皆已縮減網格）`);
    }
  };

  /**
   * 🔗 執行完成 (Merge All Routes)
   * 重複執行「合併一筆路線」直到沒有更多可合併的項目
   */
  const mergeAllRoutes = (gap = 0) => {
    if (!currentLayer.value) return;

    const layoutData = currentLayer.value.layoutGridJsonData_Test;
    if (!Array.isArray(layoutData)) {
      console.warn('layoutGridJsonData_Test 不是 Array（目前只支援 2-5 routes array 格式）');
      return;
    }

    const isConnectNodeAt = (seg, idx) => {
      const hasValue = (v) => v !== undefined && v !== null;
      const node = Array.isArray(seg.nodes) ? seg.nodes[idx] : null;
      const pt = Array.isArray(seg.points) ? seg.points[idx] : null;
      const ptProps = Array.isArray(pt) && pt.length > 2 ? pt[2] : {};
      const tags = ptProps?.tags || node?.tags || {};

      const nodeType =
        node?.node_type ||
        ptProps?.node_type ||
        tags?.node_type ||
        (hasValue(node?.connect_number) ? 'connect' : '');
      const hasConnectNumber =
        hasValue(node?.connect_number) ||
        hasValue(tags?.connect_number) ||
        hasValue(ptProps?.connect_number) ||
        hasValue(ptProps?.tags?.connect_number);
      return nodeType === 'connect' || hasConnectNumber;
    };

    /**
     * 判斷是否為真正的車站（有 station_name 的點）
     * 只有真正的車站才可被合併刪除
     * 幾何轉折點（無 station_name）和 connect 節點不應被刪除
     */
    const isRealStation = (seg, idx) => {
      const node = Array.isArray(seg.nodes) ? seg.nodes[idx] : null;
      const pt = Array.isArray(seg.points) ? seg.points[idx] : null;
      const ptProps = Array.isArray(pt) && pt.length > 2 ? pt[2] : {};
      const tags = ptProps?.tags || node?.tags || {};

      // 有 station_name 才算真正的車站
      const hasStationName = !!(
        node?.station_name ||
        ptProps?.station_name ||
        tags?.station_name ||
        ptProps?.tags?.station_name
      );

      return hasStationName;
    };

    /**
     * 判斷某個點是否為幾何轉折點（前後線段方向不同）
     * 如果是轉折點，刪除後需要保留座標以維持路線形狀
     */
    const isBendPoint = (points, idx) => {
      if (idx <= 0 || idx >= points.length - 1) return false;

      const prev = points[idx - 1];
      const curr = points[idx];
      const next = points[idx + 1];

      const px = Array.isArray(prev) ? prev[0] : prev.x || 0;
      const py = Array.isArray(prev) ? prev[1] : prev.y || 0;
      const cx = Array.isArray(curr) ? curr[0] : curr.x || 0;
      const cy = Array.isArray(curr) ? curr[1] : curr.y || 0;
      const nx = Array.isArray(next) ? next[0] : next.x || 0;
      const ny = Array.isArray(next) ? next[1] : next.y || 0;

      // 計算前段向量和後段向量
      const dx1 = cx - px;
      const dy1 = cy - py;
      const dx2 = nx - cx;
      const dy2 = ny - cy;

      // 如果向量方向不同（不共線），就是轉折點
      // 使用叉積判斷：如果叉積不為 0，表示不共線
      const crossProduct = dx1 * dy2 - dy1 * dx2;
      const epsilon = 0.001; // 容許微小誤差

      return Math.abs(crossProduct) > epsilon;
    };

    let totalMerged = 0;
    let hasMore = true;

    // 循環執行合併，直到沒有更多可合併的項目
    while (hasMore) {
      hasMore = false;
      let mergedInThisRound = false;

      for (const route of layoutData) {
        if (mergedInThisRound) break;
        const segments = route?.segments || [];

        for (const seg of segments) {
          if (mergedInThisRound) break;
          const points = Array.isArray(seg.points) ? seg.points : [];
          const weights = Array.isArray(seg.station_weights) ? seg.station_weights : [];
          if (points.length < 3 || weights.length < 2) continue;

          for (let i = 0; i < weights.length - 1; i++) {
            const w1 = weights[i];
            const w2 = weights[i + 1];
            if (!w1 || !w2) continue;
            // 使用 gap 參數：允許 weight 差 <= gap
            const weightDiff = Math.abs((w1.weight || 0) - (w2.weight || 0));
            if (weightDiff > gap) continue;
            if (w1.end_idx !== w2.start_idx) continue;

            const midIdx = w1.end_idx;
            // 不刪端點，避免破壞線段
            if (midIdx <= 0 || midIdx >= points.length - 1) continue;
            // 不刪 connect/轉乘點
            if (isConnectNodeAt(seg, midIdx)) continue;
            // 不刪幾何轉折點（只刪真正的車站）
            if (!isRealStation(seg, midIdx)) continue;

            // 1) 合併兩段 weight：把 w1 end 延伸到 w2 end，並移除 w2
            w1.end_idx = w2.end_idx;
            weights.splice(i + 1, 1);

            // 2) 檢查是否為幾何轉折點
            const isBend = isBendPoint(points, midIdx);

            if (isBend) {
              // 如果是轉折點，保留座標但移除車站屬性
              // 將該點轉換為純幾何點 [x, y]
              const pt = points[midIdx];
              const x = Array.isArray(pt) ? pt[0] : pt.x || 0;
              const y = Array.isArray(pt) ? pt[1] : pt.y || 0;
              points[midIdx] = [x, y]; // 只保留座標，不保留屬性

              // 🎯 將 nodes 對應位置標記為幾何轉折點（不刪除，保持索引一致）
              if (Array.isArray(seg.nodes)) {
                // 確保 nodes 數組長度與 points 一致
                while (seg.nodes.length < points.length) {
                  seg.nodes.push({});
                }
                // 將該位置標記為幾何點，node_type: 'line' 表示非車站點
                seg.nodes[midIdx] = { node_type: 'line' };
              }

              // ⚠️ 注意：因為 points 沒有刪除（只是改成純座標），所以 station_weights 的索引不需要調整
            } else {
              // 如果不是轉折點，直接刪除該點
              seg.points.splice(midIdx, 1);
              if (Array.isArray(seg.nodes) && seg.nodes.length > midIdx) {
                seg.nodes.splice(midIdx, 1);
              }

              // 3) 修正所有 station_weights 的索引（刪除點後，midIdx 之後的索引都要 -1）
              for (const w of weights) {
                if (w.start_idx > midIdx) w.start_idx--;
                if (w.end_idx > midIdx) w.end_idx--;
              }
            }

            mergedInThisRound = true;
            hasMore = true;
            totalMerged++;
            break;
          }
        }
      }

      // 如果這一輪有合併，更新 dataTableData 中對應的 row（設為 V）
      if (mergedInThisRound && Array.isArray(currentLayer.value.dataTableData)) {
        // 找到第一個還沒設成 V 的項目（使用 gap 參數）
        const row = currentLayer.value.dataTableData.find(
          (r) =>
            r &&
            r.合併 !== 'V' &&
            r.node_type !== 'connect' &&
            Math.abs((r.weight_1 || 0) - (r.weight_2 || 0)) <= gap
        );
        if (row) row.合併 = 'V';
      }
    }

    if (totalMerged > 0) {
      console.log(`🎉 執行完成！共合併 ${totalMerged} 個點`);

      // 🔄 用新引用觸發 LayoutGridTab_Test 的 deep watch，讓點消失後自動重繪
      currentLayer.value.layoutGridJsonData_Test = JSON.parse(
        JSON.stringify(currentLayer.value.layoutGridJsonData_Test)
      );

      // 🔄 同步更新到 layoutGridJsonData_Test2
      currentLayer.value.layoutGridJsonData_Test2 = JSON.parse(
        JSON.stringify(currentLayer.value.layoutGridJsonData_Test)
      );
    } else {
      console.log('所有項目都已處理完畢，沒有需要合併的項目');
    }
  };

  /**
   * 📉 縮減網格 (Reduce Grid)
   * 刪除整個 col 或 row 沒有黑點或路線的網格，並調整座標讓網格大小縮減
   */
  const reduceGrid = () => {
    if (!currentLayer.value) return;

    const layoutData = currentLayer.value.layoutGridJsonData_Test;
    if (!layoutData) {
      console.warn('layoutGridJsonData_Test 為空');
      return;
    }

    // 處理兩種格式：Array 或 Object（有 meta）
    let routes;
    let meta;
    if (Array.isArray(layoutData)) {
      routes = layoutData;
      meta = null;
    } else if (layoutData && typeof layoutData === 'object' && Array.isArray(layoutData.routes)) {
      routes = layoutData.routes;
      meta = layoutData.meta || null;
    } else {
      console.warn('layoutGridJsonData_Test 格式不支援（目前只支援 Array 或 {routes, meta} 格式）');
      return;
    }

    // 1. 統計所有使用的座標
    const usedCols = new Set();
    const usedRows = new Set();

    routes.forEach((route) => {
      const segments = route?.segments || [];
      segments.forEach((seg) => {
        const points = Array.isArray(seg.points) ? seg.points : [];
        points.forEach((pt) => {
          const x = Array.isArray(pt) ? pt[0] : pt.x || 0;
          const y = Array.isArray(pt) ? pt[1] : pt.y || 0;
          usedCols.add(Math.round(x));
          usedRows.add(Math.round(y));
        });
      });
    });

    // 2. 找出空 col/row 並建立映射表（舊座標 -> 新座標）
    const colMap = new Map(); // 舊 x -> 新 x
    const rowMap = new Map(); // 舊 y -> 新 y

    if (usedCols.size === 0 || usedRows.size === 0) {
      console.warn('沒有找到任何使用的座標');
      return;
    }

    // 計算 col 映射（刪除空的 col）
    const sortedCols = Array.from(usedCols).sort((a, b) => a - b);
    const minX = sortedCols[0];
    const maxX = sortedCols[sortedCols.length - 1];

    let newX = 0;
    for (let oldX = minX; oldX <= maxX; oldX++) {
      if (usedCols.has(oldX)) {
        colMap.set(oldX, newX);
        newX++;
      }
    }

    // 計算 row 映射（刪除空的 row）
    const sortedRows = Array.from(usedRows).sort((a, b) => a - b);
    const minY = sortedRows[0];
    const maxY = sortedRows[sortedRows.length - 1];

    let newY = 0;
    for (let oldY = minY; oldY <= maxY; oldY++) {
      if (usedRows.has(oldY)) {
        rowMap.set(oldY, newY);
        newY++;
      }
    }

    const removedCols = maxX - minX + 1 - newX;
    const removedRows = maxY - minY + 1 - newY;

    if (removedCols === 0 && removedRows === 0) {
      console.log('沒有空的 col/row 需要刪除');
      return;
    }

    console.log(`📉 縮減網格：刪除 ${removedCols} 個空 col，${removedRows} 個空 row`);

    // 3. 調整所有點的座標
    routes.forEach((route) => {
      const segments = route?.segments || [];
      segments.forEach((seg) => {
        const points = Array.isArray(seg.points) ? seg.points : [];
        points.forEach((pt, idx) => {
          if (Array.isArray(pt)) {
            const oldX = pt[0];
            const oldY = pt[1];
            const newX = colMap.get(Math.round(oldX)) ?? oldX;
            const newY = rowMap.get(Math.round(oldY)) ?? oldY;

            if (pt.length > 2) {
              // [x, y, props] 格式，保留 props
              points[idx] = [newX, newY, pt[2]];
            } else {
              // [x, y] 格式
              points[idx] = [newX, newY];
            }
          } else if (pt && typeof pt === 'object') {
            // {x, y} 格式
            const oldX = pt.x || 0;
            const oldY = pt.y || 0;
            pt.x = colMap.get(Math.round(oldX)) ?? oldX;
            pt.y = rowMap.get(Math.round(oldY)) ?? oldY;
          }
        });
      });
    });

    // 4. 更新 meta.gridWidth 和 meta.gridHeight（如果存在）
    if (meta) {
      meta.gridWidth = newX;
      meta.gridHeight = newY;
      if (typeof meta.width === 'number') meta.width = newX;
      if (typeof meta.height === 'number') meta.height = newY;
    }

    console.log(`✅ 網格已縮減：新尺寸 ${newX} x ${newY}`);

    // 5. 觸發資料更新
    currentLayer.value.layoutGridJsonData_Test = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test)
    );

    // 🔄 同步更新到 layoutGridJsonData_Test2
    currentLayer.value.layoutGridJsonData_Test2 = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test)
    );
  };

  /**
   * 🔗 執行完成4 (Merge All Routes for Test4)
   * 重複執行「合併一筆路線」直到沒有更多可合併的項目（針對 layoutGridJsonData_Test4）
   */
  // ✅ 供 mergeAllRoutes4 / reduceGrid4 共用：從 routesData 重新生成 dataTableData（taipei_6_1_test3 專用格式）
  const generateDataTableData_Test4 = (routesData) => {
    const gridNodes = new Map(); // key: "x,y", value: { xGrid, yGrid, maxWeight: number, weights: number[] }

    const addWeightAt = (x, y, weight) => {
      if (!Number.isFinite(x) || !Number.isFinite(y)) return;
      if (typeof weight !== 'number' || !Number.isFinite(weight)) return;
      const xGrid = Math.round(x);
      const yGrid = Math.round(y);
      const key = `${xGrid},${yGrid}`;
      if (!gridNodes.has(key)) {
        gridNodes.set(key, { xGrid, yGrid, maxWeight: weight, weights: [weight] });
      } else {
        const existing = gridNodes.get(key);
        if (weight > (existing.maxWeight ?? -Infinity)) {
          existing.maxWeight = weight;
          existing.weights = [weight];
        }
      }
    };

    const rasterizeAndAddWeight = (ax, ay, bx, by, weight) => {
      ax = Math.round(ax);
      ay = Math.round(ay);
      bx = Math.round(bx);
      by = Math.round(by);

      const dx = Math.abs(bx - ax);
      const dy = Math.abs(by - ay);

      if (dx === 0 && dy === 0) {
        addWeightAt(ax, ay, weight);
        return;
      }

      if (dy === 0) {
        const x0 = Math.min(ax, bx);
        const x1 = Math.max(ax, bx);
        for (let x = x0; x <= x1; x++) addWeightAt(x, ay, weight);
        return;
      }

      if (dx === 0) {
        const y0 = Math.min(ay, by);
        const y1 = Math.max(ay, by);
        for (let y = y0; y <= y1; y++) addWeightAt(ax, y, weight);
        return;
      }

      // 斜線（Bresenham）
      let x = ax;
      let y = ay;
      const sx = ax < bx ? 1 : -1;
      const sy = ay < by ? 1 : -1;
      let err = dx - dy;
      const maxSteps = dx + dy + 1;
      for (let steps = 0; steps < maxSteps; steps++) {
        addWeightAt(x, y, weight);
        if (x === bx && y === by) break;
        const e2 = 2 * err;
        if (e2 > -dy) {
          err -= dy;
          x += sx;
        }
        if (e2 < dx) {
          err += dx;
          y += sy;
        }
      }
    };

    // 將 station_weights / edge_weights 依照端點座標分配到節點
    for (const route of routesData || []) {
      const segments = route?.segments || [];
      for (const seg of segments) {
        const points = Array.isArray(seg.points) ? seg.points : [];
        if (points.length < 2) continue;

        const stationWeights = Array.isArray(seg.station_weights) ? seg.station_weights : null;
        const edgeWeights = Array.isArray(seg.edge_weights) ? seg.edge_weights : null;

        if (stationWeights && stationWeights.length > 0) {
          for (const wInfo of stationWeights) {
            const sIdx = Number.isFinite(wInfo?.start_idx) ? wInfo.start_idx : null;
            const eIdx = Number.isFinite(wInfo?.end_idx) ? wInfo.end_idx : null;
            const w = wInfo?.weight;
            if (
              sIdx === null ||
              eIdx === null ||
              sIdx < 0 ||
              eIdx < 0 ||
              sIdx >= points.length ||
              eIdx >= points.length
            ) {
              continue;
            }

            const step = sIdx <= eIdx ? 1 : -1;
            for (let i = sIdx; i !== eIdx; i += step) {
              const p1 = points[i];
              const p2 = points[i + step];
              const x1 = Array.isArray(p1) ? p1[0] : p1?.x;
              const y1 = Array.isArray(p1) ? p1[1] : p1?.y;
              const x2 = Array.isArray(p2) ? p2[0] : p2?.x;
              const y2 = Array.isArray(p2) ? p2[1] : p2?.y;
              rasterizeAndAddWeight(Number(x1), Number(y1), Number(x2), Number(y2), Number(w));
            }
          }
        } else if (edgeWeights && edgeWeights.length > 0) {
          const nEdges = Math.min(edgeWeights.length, points.length - 1);
          for (let i = 0; i < nEdges; i++) {
            const w = edgeWeights[i];
            const p1 = points[i];
            const p2 = points[i + 1];
            const x1 = Array.isArray(p1) ? p1[0] : p1?.x;
            const y1 = Array.isArray(p1) ? p1[1] : p1?.y;
            const x2 = Array.isArray(p2) ? p2[0] : p2?.x;
            const y2 = Array.isArray(p2) ? p2[1] : p2?.y;
            rasterizeAndAddWeight(Number(x1), Number(y1), Number(x2), Number(y2), Number(w));
          }
        }
      }
    }

    // 找出網格的有效範圍
    let minRow = Infinity;
    let maxRow = -Infinity;
    let minCol = Infinity;
    let maxCol = -Infinity;

    for (const node of gridNodes.values()) {
      minRow = Math.min(minRow, node.yGrid);
      maxRow = Math.max(maxRow, node.yGrid);
      minCol = Math.min(minCol, node.xGrid);
      maxCol = Math.max(maxCol, node.xGrid);
    }

    if (
      minRow === Infinity ||
      maxRow === -Infinity ||
      minCol === Infinity ||
      maxCol === -Infinity
    ) {
      return [];
    }

    // 計算每一列/行的最大值
    const colMaxValues = {};
    const rowMaxValues = {};

    for (let col = minCol; col <= maxCol; col++) colMaxValues[col] = 0;
    for (let row = minRow; row <= maxRow; row++) rowMaxValues[row] = 0;

    for (const node of gridNodes.values()) {
      const maxWeight = node.weights.length > 0 ? Math.max(...node.weights) : 0;
      colMaxValues[node.xGrid] = Math.max(colMaxValues[node.xGrid] || 0, maxWeight);
      rowMaxValues[node.yGrid] = Math.max(rowMaxValues[node.yGrid] || 0, maxWeight);
    }

    const colSingles = [];
    const rowSingles = [];

    for (let col = minCol; col <= maxCol; col++) {
      colSingles.push({ actualCol: col, colMaxWeight: colMaxValues[col] ?? 0 });
    }
    for (let row = minRow; row <= maxRow; row++) {
      rowSingles.push({ actualRow: row, rowMaxWeight: rowMaxValues[row] ?? 0 });
    }

    // 過濾出「奇數座標」的 col / row（直接用座標 parity 判斷，避免 minCol/minRow parity 導致漏抓）
    const colSinglesOdd = colSingles.filter((single) => single.actualCol % 2 !== 0);
    const rowSinglesOdd = rowSingles.filter((single) => single.actualRow % 2 !== 0);

    const dataTableData = [];

    // col：每個奇數 col 與下一個奇數 col 一組（相鄰兩個奇數座標應差 2）
    for (let i = 0; i < colSinglesOdd.length; i++) {
      const col1 = colSinglesOdd[i];
      const col2 = colSinglesOdd[i + 1];
      if (!col1 || !col2) continue;
      if (col2.actualCol !== col1.actualCol + 2) continue;
      dataTableData.push({
        type: 'col',
        // ✅ 直接存「實際座標」，mergeAllRoutes4 不再依賴 minCol 偏移
        idx1: col1.actualCol,
        idx2: col2.actualCol,
        idx1_max_weight: col1.colMaxWeight ?? 0,
        idx2_max_weight: col2.colMaxWeight ?? 0,
        合併: 'X',
      });
    }

    // row：每個奇數 row 與下一個奇數 row 一組（相鄰兩個奇數座標應差 2）
    for (let i = 0; i < rowSinglesOdd.length; i++) {
      const row1 = rowSinglesOdd[i];
      const row2 = rowSinglesOdd[i + 1];
      if (!row1 || !row2) continue;
      if (row2.actualRow !== row1.actualRow + 2) continue;
      dataTableData.push({
        type: 'row',
        idx1: row1.actualRow,
        idx2: row2.actualRow,
        idx1_max_weight: row1.rowMaxWeight ?? 0,
        idx2_max_weight: row2.rowMaxWeight ?? 0,
        合併: 'X',
      });
    }

    // 排序：先 col 再 row；同 type 內用 sum 由小到大
    dataTableData.sort((a, b) => {
      if (a.type !== b.type) return a.type === 'col' ? -1 : 1;
      const aSum = (a.idx1_max_weight ?? 0) + (a.idx2_max_weight ?? 0);
      const bSum = (b.idx1_max_weight ?? 0) + (b.idx2_max_weight ?? 0);
      return aSum - bSum;
    });

    return dataTableData.map((item, index) => ({
      '#': index + 1,
      type: item.type,
      idx1: item.idx1,
      idx2: item.idx2,
      idx1_max_weight: item.idx1_max_weight,
      idx2_max_weight: item.idx2_max_weight,
      合併: item.合併 ?? 'X',
    }));
  };

  /**
   * 🎲 隨機產生權重4 (Randomize Weights for Test4)
   * 根據指定的權重值和機率分佈重新產生所有權重（針對 layoutGridJsonData_Test4）
   * 同時更新 layoutGridJsonData_Test4 和 dataTableData
   */
  const randomizeWeights4 = () => {
    if (!currentLayer.value) {
      console.warn('當前圖層不存在');
      return;
    }

    const layoutData = currentLayer.value.layoutGridJsonData_Test4;
    if (!Array.isArray(layoutData)) {
      console.warn('layoutGridJsonData_Test4 不是 Array');
      return;
    }

    // 權重值和機率分佈（與 randomizeWeights 相同）
    const WEIGHT_VALUES = [1, 2, 3, 4, 5, 6, 7, 8, 9];
    const WEIGHT_PROBS = [729, 512, 343, 216, 125, 64, 27, 8, 1];

    // 計算總機率
    const totalProb = WEIGHT_PROBS.reduce((sum, prob) => sum + prob, 0);

    // 產生隨機權重的函數
    const generateRandomWeight = () => {
      const random = Math.random() * totalProb;
      let cumulative = 0;
      for (let i = 0; i < WEIGHT_VALUES.length; i++) {
        cumulative += WEIGHT_PROBS[i];
        if (random <= cumulative) {
          return WEIGHT_VALUES[i];
        }
      }
      return WEIGHT_VALUES[0]; // 預設返回最小值
    };

    // 遍歷所有路線，重新產生 station_weights
    layoutData.forEach((route) => {
      if (!route || !Array.isArray(route.segments)) return;

      route.segments.forEach((seg) => {
        if (!seg || !Array.isArray(seg.station_weights)) return;

        // 重新產生每個 station_weight 的權重值
        seg.station_weights.forEach((weightInfo) => {
          if (weightInfo && typeof weightInfo.weight === 'number') {
            weightInfo.weight = generateRandomWeight();
          }
        });
      });
    });

    // 🔄 重新生成 dataTableData（使用 generateDataTableData_Test4）
    currentLayer.value.dataTableData = generateDataTableData_Test4(layoutData);

    // 🔄 用新引用觸發 LayoutGridTab_Test4 的 deep watch，讓權重更新後自動重繪
    currentLayer.value.layoutGridJsonData_Test4 = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test4)
    );

    console.log('已重新產生所有權重值並更新 dataTableData 和 layoutGridJsonData_Test4');
  };

  const mergeAllRoutes4 = (gap = 0) => {
    if (!currentLayer.value || currentLayer.value.layerId !== 'taipei_6_1_test3') return;

    const layoutData = currentLayer.value.layoutGridJsonData_Test4;
    if (!Array.isArray(layoutData)) {
      console.warn('layoutGridJsonData_Test4 不是 Array（目前只支援 2-5 routes array 格式）');
      return;
    }

    // ✅ 每次合併後都重新生成 dataTableData，直到整個畫面沒有可合併項目才停止

    // 計算最小座標值（用於將 idx 轉換為實際座標）
    let minCol = Infinity;
    let minRow = Infinity;
    for (const route of layoutData) {
      const segments = route?.segments || [];
      for (const seg of segments) {
        const points = Array.isArray(seg.points) ? seg.points : [];
        for (const pt of points) {
          const x = Array.isArray(pt) ? pt[0] : pt?.x;
          const y = Array.isArray(pt) ? pt[1] : pt?.y;
          if (Number.isFinite(x)) minCol = Math.min(minCol, Math.round(x));
          if (Number.isFinite(y)) minRow = Math.min(minRow, Math.round(y));
        }
      }
    }
    if (minCol === Infinity) minCol = 0;
    if (minRow === Infinity) minRow = 0;

    const isConnectNodeAt = (seg, idx) => {
      const hasValue = (v) => v !== undefined && v !== null;
      const node = Array.isArray(seg.nodes) ? seg.nodes[idx] : null;
      const pt = Array.isArray(seg.points) ? seg.points[idx] : null;
      const ptProps = Array.isArray(pt) && pt.length > 2 ? pt[2] : {};
      const tags = ptProps?.tags || node?.tags || {};

      const nodeType =
        node?.node_type ||
        ptProps?.node_type ||
        tags?.node_type ||
        (hasValue(node?.connect_number) ? 'connect' : '');
      const hasConnectNumber =
        hasValue(node?.connect_number) ||
        hasValue(tags?.connect_number) ||
        hasValue(ptProps?.connect_number) ||
        hasValue(ptProps?.tags?.connect_number);
      return nodeType === 'connect' || hasConnectNumber;
    };

    /**
     * 判斷某個點是否為真正的黑點（station）且可以被合併
     * 只有黑點才可以合併，紅點（connect）、端點不能合併
     * ✅ 判斷標準：在 station_weights 中出現 或 有 station_name/station_id 或 node_type='station'
     * ✅ 轉折點也可以合併（但合併時會保留座標，轉換成 line 類型）
     */
    const isRealStation = (seg, idx) => {
      const node = Array.isArray(seg.nodes) ? seg.nodes[idx] : null;
      const pt = Array.isArray(seg.points) ? seg.points[idx] : null;
      const ptProps = Array.isArray(pt) && pt.length > 2 ? pt[2] : {};
      const tags = ptProps?.tags || node?.tags || {};

      // 先檢查是否為 connect 點（紅點），如果是則返回 false
      if (isConnectNodeAt(seg, idx)) return false;

      // 檢查是否在 station_weights 中出現（權重的端點一定是站點，即使它是轉折點）
      const weights = Array.isArray(seg.station_weights) ? seg.station_weights : [];
      const inWeights = weights.some(
        (w) =>
          (Number.isFinite(w?.start_idx) && w.start_idx === idx) ||
          (Number.isFinite(w?.end_idx) && w.end_idx === idx)
      );
      if (inWeights) return true;

      // 檢查是否有 station_name 或 station_id（真正的車站）
      const hasStationName = !!(
        node?.station_name ||
        ptProps?.station_name ||
        tags?.station_name ||
        ptProps?.tags?.station_name
      );
      const hasStationId = !!(
        node?.station_id ||
        ptProps?.station_id ||
        tags?.station_id ||
        ptProps?.tags?.station_id
      );

      // 或者 node_type 為 'station'（但不是 'connect' 或 'line'）
      const nodeType = node?.node_type || ptProps?.node_type || tags?.node_type || '';
      const isStationType = nodeType === 'station';

      return hasStationName || hasStationId || isStationType;
    };

    /**
     * 判斷某個點是否為幾何轉折點（前後線段方向不同）
     * 如果是轉折點，刪除後需要保留座標以維持路線形狀
     */
    const isBendPoint = (points, idx) => {
      if (idx <= 0 || idx >= points.length - 1) return false;

      const prev = points[idx - 1];
      const curr = points[idx];
      const next = points[idx + 1];

      const px = Array.isArray(prev) ? prev[0] : prev.x || 0;
      const py = Array.isArray(prev) ? prev[1] : prev.y || 0;
      const cx = Array.isArray(curr) ? curr[0] : curr.x || 0;
      const cy = Array.isArray(curr) ? curr[1] : curr.y || 0;
      const nx = Array.isArray(next) ? next[0] : next.x || 0;
      const ny = Array.isArray(next) ? next[1] : next.y || 0;

      // 計算前段向量和後段向量
      const dx1 = cx - px;
      const dy1 = cy - py;
      const dx2 = nx - cx;
      const dy2 = ny - cy;

      // 如果向量方向不同（不共線），就是轉折點
      // 使用叉積判斷：如果叉積不為 0，表示不共線
      const crossProduct = dx1 * dy2 - dy1 * dx2;
      const epsilon = 0.001; // 容許微小誤差

      return Math.abs(crossProduct) > epsilon;
    };

    let totalMerged = 0;
    let safety = 0;

    console.log(`🚀 開始合併路線 (gap<=${gap})...`);

    while (safety < 10000) {
      safety++;

      // 重新生成 dataTableData（用最新 routes）
      const dataTableData = generateDataTableData_Test4(layoutData);
      currentLayer.value.dataTableData = dataTableData;

      if (safety % 100 === 0) {
        console.log(
          `📊 第 ${safety} 輪，已合併 ${totalMerged} 個點，可選項目：${dataTableData.length}`
        );
      }

      let mergedThisRound = false;

      for (const item of dataTableData) {
        if (!item || item.合併 === 'V') continue;

        // 檢查權重是否滿足合併條件
        const w1 = Number(item.idx1_max_weight ?? 0);
        const w2 = Number(item.idx2_max_weight ?? 0);
        const weightDiff = Math.abs(w1 - w2);
        const eps = 1e-9;
        if (weightDiff > gap + eps) continue;

        // 計算實際的網格座標
        const odd1Coord = Number(item.idx1);
        const odd2Coord = Number(item.idx2);
        const evenCoord = (odd1Coord + odd2Coord) / 2;

        // 驗證：odd1Coord/odd2Coord 應為奇數、evenCoord 應為偶數
        if (
          odd1Coord % 2 === 0 ||
          odd2Coord % 2 === 0 ||
          evenCoord % 2 !== 0 ||
          odd2Coord !== odd1Coord + 2
        ) {
          continue;
        }

        let mergedInThisItem = false;
        let deletedPointsCount = 0;
        let changedWeightsCount = 0;
        const mergedWeight = Math.max(item.idx1_max_weight ?? 0, item.idx2_max_weight ?? 0);

        // 遍歷所有路線，處理在偶數排上的點和合併奇數排的路線
        for (const route of layoutData) {
          const segments = route?.segments || [];
          for (const seg of segments) {
            const points = Array.isArray(seg.points) ? seg.points : [];
            const weights = Array.isArray(seg.station_weights) ? seg.station_weights : [];
            if (points.length === 0) continue;

            // 1. 刪除偶數排上的點（從後往前遍歷，避免索引變化影響）
            for (let idx = points.length - 1; idx >= 0; idx--) {
              const pt = points[idx];
              const x = Array.isArray(pt) ? pt[0] : pt?.x || 0;
              const y = Array.isArray(pt) ? pt[1] : pt?.y || 0;
              const xGrid = Math.round(x);
              const yGrid = Math.round(y);

              // 判斷點是否在要刪除的偶數排上
              let shouldDelete = false;
              if (item.type === 'col') shouldDelete = xGrid === evenCoord;
              else shouldDelete = yGrid === evenCoord;

              if (!shouldDelete) continue;

              // 不刪端點，避免破壞線段
              if (idx <= 0 || idx >= points.length - 1) {
                if (safety <= 5) {
                  console.log(`⏭️ 跳過端點 idx=${idx}, 座標 (${xGrid}, ${yGrid})`);
                }
                continue;
              }
              // 不刪 connect/轉乘點（紅點）
              if (isConnectNodeAt(seg, idx)) {
                if (safety <= 5) {
                  console.log(`⏭️ 跳過紅點 idx=${idx}, 座標 (${xGrid}, ${yGrid})`);
                }
                continue;
              }
              // 🎯 只合併黑點（station），不合併紅點（connect）和端點
              if (!isRealStation(seg, idx)) {
                if (safety <= 5) {
                  console.log(`⏭️ 跳過非站點 idx=${idx}, 座標 (${xGrid}, ${yGrid})`);
                }
                continue;
              }

              // 檢查是否為幾何轉折點
              const isBend = isBendPoint(points, idx);

              if (isBend) {
                // 轉折點：保留座標作為幾何點，移除站點屬性，並合併相關的 station_weights
                const pt = points[idx];
                const x = Array.isArray(pt) ? pt[0] : pt.x || 0;
                const y = Array.isArray(pt) ? pt[1] : pt.y || 0;
                points[idx] = [x, y]; // 只保留座標，不保留屬性

                if (Array.isArray(seg.nodes)) {
                  while (seg.nodes.length < points.length) seg.nodes.push({});
                  seg.nodes[idx] = { node_type: 'line' };
                }

                // 合併 station_weights：找到以 idx 為端點的 weights 並合併
                const wIn = weights.find((w) => w.end_idx === idx);
                const wOut = weights.find((w) => w.start_idx === idx);

                if (wIn && wOut) {
                  const combinedWeight = Math.max(
                    Number(wIn.weight) || 0,
                    Number(wOut.weight) || 0
                  );
                  // 更新第一個 weight，刪除第二個
                  wIn.end_idx = wOut.end_idx;
                  wIn.weight = combinedWeight;
                  const outIdx = weights.indexOf(wOut);
                  if (outIdx >= 0) weights.splice(outIdx, 1);

                  // 標記已合併，避免重複處理
                  changedWeightsCount++;
                }
              } else {
                // 非轉折點：直接刪除點
                seg.points.splice(idx, 1);
                if (Array.isArray(seg.nodes) && seg.nodes.length > idx) seg.nodes.splice(idx, 1);

                // 更新 station_weights 的索引
                for (const w of weights) {
                  if (w.start_idx > idx) w.start_idx--;
                  if (w.end_idx > idx) w.end_idx--;
                }
              }

              mergedInThisItem = true;
              totalMerged++;
              deletedPointsCount++;
            }

            // 2. 更新經過 odd1/odd2 的 station_weights 的 weight
            for (const w of weights) {
              if (
                w.start_idx < 0 ||
                w.end_idx < 0 ||
                w.start_idx >= points.length ||
                w.end_idx >= points.length
              )
                continue;

              let passesOdd1 = false;
              let passesOdd2 = false;

              const step = w.start_idx <= w.end_idx ? 1 : -1;
              for (let i = w.start_idx; i !== w.end_idx; i += step) {
                const pt = points[i];
                const x = Array.isArray(pt) ? pt[0] : pt?.x || 0;
                const y = Array.isArray(pt) ? pt[1] : pt?.y || 0;
                const xGrid = Math.round(x);
                const yGrid = Math.round(y);

                if (item.type === 'col') {
                  if (xGrid === odd1Coord) passesOdd1 = true;
                  if (xGrid === odd2Coord) passesOdd2 = true;
                } else {
                  if (yGrid === odd1Coord) passesOdd1 = true;
                  if (yGrid === odd2Coord) passesOdd2 = true;
                }
              }

              if (passesOdd1 || passesOdd2) {
                const prev = Number(w.weight);
                if (!Number.isFinite(prev) || Math.abs(prev - Number(mergedWeight)) > eps) {
                  w.weight = mergedWeight;
                  mergedInThisItem = true;
                  changedWeightsCount++;
                }
              }
            }
          }
        }

        // ✅ 只有真的有改變（刪點或改 weight）才算合併成功
        if (mergedInThisItem && (deletedPointsCount > 0 || changedWeightsCount > 0)) {
          item.合併 = 'V';
          mergedThisRound = true;
          // 重要：完成一筆後立刻跳出，下一輪用最新資料重新算 dataTableData
          break;
        }
      }

      if (!mergedThisRound) break;
    }

    if (safety >= 10000) {
      console.warn('⚠️ mergeAllRoutes4 達到安全上限，停止避免無限迴圈');
    }

    if (totalMerged > 0) {
      console.log(`🎉 執行完成4！共處理 ${totalMerged} 個點（直到沒有可合併項目）`);
      currentLayer.value.layoutGridJsonData_Test4 = JSON.parse(JSON.stringify(layoutData));
      currentLayer.value.dataTableData = generateDataTableData_Test4(layoutData);
    } else {
      console.log('沒有找到符合條件的項目可以合併');
    }
  };

  /**
   * 🔀 合併路線4-H (Merge Routes 4 - Horizontal Only)
   * 只合併位在水平線上的黑點
   */
  const mergeAllRoutes4H = (gap = 0) => {
    if (!currentLayer.value || currentLayer.value.layerId !== 'taipei_6_1_test3') return;

    let layoutData = currentLayer.value.layoutGridJsonData_Test4;
    if (!Array.isArray(layoutData)) {
      console.warn('layoutGridJsonData_Test4 不是 Array（目前只支援 2-5 routes array 格式）');
      return;
    }

    const result = mergeRoutesHorizontal(layoutData, gap);
    if (result.modified) {
      currentLayer.value.layoutGridJsonData_Test4 = JSON.parse(JSON.stringify(result.layoutData));
      currentLayer.value.dataTableData = generateDataTableDataUtil(result.layoutData);
    }
  };

  /**
   * 🔀 合併路線4-V (Merge Routes 4 - Vertical Only)
   * 只合併位在垂直線上的黑點
   */
  const mergeAllRoutes4V = (gap = 0) => {
    if (!currentLayer.value || currentLayer.value.layerId !== 'taipei_6_1_test3') return;

    let layoutData = currentLayer.value.layoutGridJsonData_Test4;
    if (!Array.isArray(layoutData)) {
      console.warn('layoutGridJsonData_Test4 不是 Array（目前只支援 2-5 routes array 格式）');
      return;
    }

    const result = mergeRoutesVertical(layoutData, gap);
    if (result.modified) {
      currentLayer.value.layoutGridJsonData_Test4 = JSON.parse(JSON.stringify(result.layoutData));
      currentLayer.value.dataTableData = generateDataTableDataUtil(result.layoutData);
    }
  };

  /**
   * 🗜️ 縮減網格4 (Reduce Grid 4)
   * 刪除所有空的偶數網格（淺紅色），並合併相鄰的奇數網格
   */
  const reduceGrid4 = () => {
    if (!currentLayer.value) return;

    const layoutData = currentLayer.value.layoutGridJsonData_Test4;
    if (!layoutData) {
      console.warn('layoutGridJsonData_Test4 為空');
      return;
    }

    const result = reduceGridUtil(layoutData);
    if (result.modified) {
      currentLayer.value.layoutGridJsonData_Test4 = JSON.parse(JSON.stringify(result.layoutData));
      if (currentLayer.value.layerId === 'taipei_6_1_test3') {
        const routes = Array.isArray(result.layoutData)
          ? result.layoutData
          : result.layoutData.routes || [];
        currentLayer.value.dataTableData = generateDataTableDataUtil(routes);
      }
    }
  };

  // ==================== 🗑️ Test2: 固定 row/col 為 1pt（不刪除資料） ====================

  /**
   * 確保 layoutGridJsonData_Test3 具備 meta（若目前是 Array，轉成 {routes, meta}）
   * @returns {Object|null} Object with routes and meta properties, or null
   */
  const ensureTest3LayoutAndMeta = () => {
    if (!currentLayer.value) return null;
    const layoutData = currentLayer.value.layoutGridJsonData_Test3;
    if (!layoutData) return null;

    if (Array.isArray(layoutData)) {
      const wrapped = { routes: layoutData, meta: {} };
      currentLayer.value.layoutGridJsonData_Test3 = wrapped;
      return wrapped;
    }

    if (layoutData && typeof layoutData === 'object' && Array.isArray(layoutData.routes)) {
      layoutData.meta =
        layoutData.meta && typeof layoutData.meta === 'object' ? layoutData.meta : {};
      return { routes: layoutData.routes, meta: layoutData.meta };
    }

    console.warn('layoutGridJsonData_Test3 格式不支援');
    return null;
  };

  /**
   * 從 routes/segments/points 推算目前資料的最小座標（用來把 idx 轉成實際座標）
   */
  const getTest3MinXY = (routes) => {
    let minX = Infinity;
    let minY = Infinity;
    (routes || []).forEach((route) => {
      const segments = route?.segments || [];
      segments.forEach((seg) => {
        const points = Array.isArray(seg.points) ? seg.points : [];
        points.forEach((pt) => {
          const x = Array.isArray(pt) ? pt[0] : pt?.x;
          const y = Array.isArray(pt) ? pt[1] : pt?.y;
          if (Number.isFinite(Number(x))) minX = Math.min(minX, Math.round(Number(x)));
          if (Number.isFinite(Number(y))) minY = Math.min(minY, Math.round(Number(y)));
        });
      });
    });
    return {
      minX: Number.isFinite(minX) ? minX : 0,
      minY: Number.isFinite(minY) ? minY : 0,
    };
  };

  const addUniqueSorted = (arr, value) => {
    const v = Math.round(value);
    const base = Array.isArray(arr) ? arr.map((x) => Math.round(x)) : [];
    if (!base.includes(v)) base.push(v);
    base.sort((a, b) => a - b);
    return base;
  };

  /**
   * 🗑️ 刪除row：不刪除資料，只把該 row 的高度固定成 1pt（寫入 meta.fixedRows）
   * 依 dataTableData 的順序：type=row、removable=V、且 刪除 != 'V'
   * 注意：只能改 dataTableData 的「刪除」欄位
   */
  const deleteOneRow = () => {
    if (!currentLayer.value) return false;
    const tableData = currentLayer.value.dataTableData;
    if (!Array.isArray(tableData) || tableData.length === 0) return false;

    const payload = ensureTest3LayoutAndMeta();
    if (!payload) return false;

    const targetRow = tableData.find(
      (r) => r && r.type === 'row' && r.removable === 'V' && r.刪除 !== 'V'
    );
    if (!targetRow) return false;

    const idx = targetRow.idx;
    if (typeof idx !== 'number' || !Number.isFinite(idx)) return false;

    const { minY } = getTest3MinXY(payload.routes);
    const actualRow = minY + idx;

    payload.meta.fixedRows = addUniqueSorted(payload.meta.fixedRows, actualRow);
    targetRow.刪除 = 'V';

    currentLayer.value.layoutGridJsonData_Test3 = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test3)
    );
    return true;
  };

  /**
   * 🗑️ 刪除col：不刪除資料，只把該 col 的寬度固定成 1pt（寫入 meta.fixedCols）
   * 依 dataTableData 的順序：type=col、removable=V、且 刪除 != 'V'
   * 注意：只能改 dataTableData 的「刪除」欄位
   */
  const deleteOneCol = () => {
    if (!currentLayer.value) return false;
    const tableData = currentLayer.value.dataTableData;
    if (!Array.isArray(tableData) || tableData.length === 0) return false;

    const payload = ensureTest3LayoutAndMeta();
    if (!payload) return false;

    const targetCol = tableData.find(
      (r) => r && r.type === 'col' && r.removable === 'V' && r.刪除 !== 'V'
    );
    if (!targetCol) return false;

    const idx = targetCol.idx;
    if (typeof idx !== 'number' || !Number.isFinite(idx)) return false;

    const { minX } = getTest3MinXY(payload.routes);
    const actualCol = minX + idx;

    payload.meta.fixedCols = addUniqueSorted(payload.meta.fixedCols, actualCol);
    targetCol.刪除 = 'V';

    currentLayer.value.layoutGridJsonData_Test3 = JSON.parse(
      JSON.stringify(currentLayer.value.layoutGridJsonData_Test3)
    );
    return true;
  };

  const deleteAllRows = () => {
    while (deleteOneRow()) {
      // keep deleting
    }
  };

  const deleteAllCols = () => {
    while (deleteOneCol()) {
      // keep deleting
    }
  };
</script>

<template>
  <!-- 🎮 操作控制分頁組件 -->
  <div class="d-flex flex-column my-bgcolor-gray-200 h-100">
    <!-- 📑 圖層分頁導航 -->
    <div v-if="visibleLayers.length > 0" class="">
      <ul class="nav nav-tabs nav-fill">
        <li
          v-for="layer in validVisibleLayers"
          :key="layer.layerId"
          class="nav-item d-flex flex-column align-items-center"
        >
          <!-- tab按鈕 -->
          <div
            class="btn nav-link rounded-0 border-0 position-relative d-flex align-items-center justify-content-center my-bgcolor-gray-200"
            :class="{
              active: activeLayerTab === layer.layerId,
            }"
            @click="setActiveLayerTab(layer.layerId)"
          >
            <span class="my-title-sm-black">
              <span v-if="getLayerFullTitle(layer).groupName" class="my-title-xs-gray"
                >{{ getLayerFullTitle(layer).groupName }} -
              </span>
              <span>{{ getLayerFullTitle(layer).layerName }}</span>
            </span>
          </div>
          <div class="w-100" :class="`my-bgcolor-${layer.colorName}`" style="min-height: 4px"></div>
        </li>
      </ul>
    </div>

    <!-- 📋 圖層操作內容區域 -->
    <div v-if="visibleLayers.length > 0" class="flex-grow-1 overflow-auto p-3 my-bgcolor-white">
      <div
        v-for="layer in validVisibleLayers"
        :key="layer.layerId"
        v-show="activeLayerTab === layer.layerId"
      >
        <!-- 執行中提示 -->
        <div v-if="isExecuting" class="pb-2 mb-3 border-bottom">
          <div class="my-title-xs-gray pb-2">計算中...</div>
          <div class="d-flex justify-content-center">
            <div class="spinner-border spinner-border-sm" role="status">
              <span class="visually-hidden">載入中...</span>
            </div>
          </div>
        </div>

        <!-- 執行按鈕區域 -->
        <div v-if="canExecuteLayer && currentLayer" class="pb-3 mb-3 border-bottom">
          <button
            v-if="currentLayer"
            class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer"
            @click="executeLayerFunction"
            :disabled="
              !currentLayer.geojsonData &&
              !currentLayer.layoutGridJsonData &&
              !currentLayer.spaceNetworkGridJsonData
            "
          >
            執行下一步
          </button>
        </div>

        <!-- RAG 問答區（僅 test_layer 顯示） -->
        <div v-if="isRagLayer" class="pb-3 mb-3 border-bottom">
          <div class="my-title-xs-gray pb-2">RAG 問答</div>
          <div class="my-title-xs-gray mb-2">
            使用向量庫: lectures_faiss_db.zip
          </div>
          <div class="mb-2">
            <textarea
              v-model="ragQuestion"
              class="form-control my-font-size-xs"
              rows="3"
              placeholder="請輸入問題..."
            ></textarea>
          </div>
          <div class="d-flex gap-2">
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer"
              @click="askRag"
              :disabled="ragIsLoading || !ragQuestion.trim() || !ragHasApi"
            >
              發問
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-transparent my-font-size-xs text-nowrap w-100 my-cursor-pointer"
              @click="clearRagHistory"
              :disabled="ragIsLoading || ragHistory.length === 0"
            >
              清除對話
            </button>
          </div>
          <div v-if="!ragHasApi" class="my-title-xs-gray mt-2">
            請設定 RAG API URL 才能使用
          </div>
          <div v-if="ragIsLoading" class="my-title-xs-gray mt-2">查詢中...</div>
          <div v-if="ragError" class="text-danger my-font-size-xs mt-2">{{ ragError }}</div>

          <div v-if="ragHistory.length > 0" class="mt-3">
            <div
              v-for="(item, index) in ragHistory"
              :key="`rag-${index}`"
              class="border rounded p-2 mb-2"
            >
              <div class="my-title-xs-gray mb-1">Q{{ index + 1 }}: {{ item.question }}</div>
              <div class="my-title-sm-black">A: {{ item.answer }}</div>
              <div v-if="item.retrievedChunks" class="my-title-xs-gray mt-2">
                參考資料：
                <pre class="rag-chunks">{{ item.retrievedChunks }}</pre>
              </div>
            </div>
          </div>
        </div>


        <!-- LayoutGridTab_Test2 當前尺寸顯示（即時顯示） -->
        <div
          v-if="layoutGridTabTest2Dimensions.x > 0 || layoutGridTabTest2Dimensions.y > 0"
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">LayoutGridTab_Test2 當前尺寸</div>
          <div class="text-center">
            <div class="my-title-sm-black">
              X: {{ layoutGridTabTest2Dimensions.x }} pt × Y:
              {{ layoutGridTabTest2Dimensions.y }} pt
            </div>
            <small class="text-muted">寬度 × 高度</small>
          </div>
        </div>

        <!-- LayoutGridTab_Test2 網格最小尺寸顯示（即時顯示） -->
        <div
          v-if="
            layoutGridTabTest2MinCellDimensions.minWidth > 0 ||
            layoutGridTabTest2MinCellDimensions.minHeight > 0
          "
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">LayoutGridTab_Test2 網格最小尺寸</div>
          <div class="text-center">
            <div class="my-title-sm-black">
              最小寬度: {{ layoutGridTabTest2MinCellDimensions.minWidth }} pt × 最小高度:
              {{ layoutGridTabTest2MinCellDimensions.minHeight }} pt
            </div>
            <small class="text-muted">最小寬度 × 最小高度</small>
          </div>
        </div>

        <!-- LayoutGridTab_Test3 當前尺寸顯示（即時顯示） -->
        <div
          v-if="layoutGridTabTest3Dimensions.x > 0 || layoutGridTabTest3Dimensions.y > 0"
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">LayoutGridTab_Test3 當前尺寸</div>
          <div class="text-center">
            <div class="my-title-sm-black">
              X: {{ layoutGridTabTest3Dimensions.x }} pt × Y:
              {{ layoutGridTabTest3Dimensions.y }} pt
            </div>
            <small class="text-muted">寬度 × 高度</small>
          </div>
        </div>

        <!-- LayoutGridTab_Test3 網格最小尺寸顯示（即時顯示） -->
        <div
          v-if="
            layoutGridTabTest3MinCellDimensions.minWidth > 0 ||
            layoutGridTabTest3MinCellDimensions.minHeight > 0
          "
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">LayoutGridTab_Test3 網格最小尺寸</div>
          <div class="text-center">
            <div class="my-title-sm-black">
              最小寬度: {{ layoutGridTabTest3MinCellDimensions.minWidth }} pt × 最小高度:
              {{ layoutGridTabTest3MinCellDimensions.minHeight }} pt
            </div>
            <small class="text-muted">最小寬度 × 最小高度</small>
          </div>
        </div>

        <!-- LayoutGridTab_Test4 當前尺寸顯示（即時顯示） -->
        <div
          v-if="layoutGridTabTest4Dimensions.x > 0 || layoutGridTabTest4Dimensions.y > 0"
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">LayoutGridTab_Test4 當前尺寸</div>
          <div class="text-center">
            <div class="my-title-sm-black">
              X: {{ layoutGridTabTest4Dimensions.x }} pt × Y:
              {{ layoutGridTabTest4Dimensions.y }} pt
            </div>
            <small class="text-muted">寬度 × 高度</small>
          </div>
        </div>

        <!-- LayoutGridTab_Test4 網格最小尺寸顯示（即時顯示） -->
        <div
          v-if="
            layoutGridTabTest4MinCellDimensions.minWidth > 0 ||
            layoutGridTabTest4MinCellDimensions.minHeight > 0
          "
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">LayoutGridTab_Test4 網格最小尺寸</div>
          <div class="text-center">
            <div class="my-title-sm-black">
              最小寬度: {{ layoutGridTabTest4MinCellDimensions.minWidth }} pt × 最小高度:
              {{ layoutGridTabTest4MinCellDimensions.minHeight }} pt
            </div>
            <small class="text-muted">最小寬度 × 最小高度</small>
          </div>
        </div>

        <!-- 當前執行的合併操作顯示（僅在 taipei_6_1_test3 圖層顯示） -->
        <div
          v-if="
            currentLayer &&
            currentLayer.layerId === 'taipei_6_1_test3' &&
            dataStore.currentMergeOperation4
          "
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">正在執行</div>
          <div class="d-flex align-items-center justify-content-center p-2 rounded" style="background-color: #e3f2fd; border: 1px solid #90caf9;">
            <div class="spinner-border spinner-border-sm text-primary me-2" role="status" style="width: 1rem; height: 1rem;">
              <span class="visually-hidden">執行中...</span>
            </div>
            <div>
              <strong class="text-primary">{{ dataStore.currentMergeOperation4 }}</strong>
            </div>
          </div>
        </div>

        <!-- 當前網格長寬顯示（僅在 taipei_6_1_test 圖層顯示） -->
        <div
          v-if="isTaipei6_1Test && currentLayer && currentLayer.layoutGridJsonData_Test2"
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">當前網格尺寸</div>
          <div class="text-center">
            <div class="my-title-sm-black">
              {{ currentGridDimensions.width }} × {{ currentGridDimensions.height }}
            </div>
            <small class="text-muted">長 × 寬</small>
          </div>
        </div>

        <!-- 當前網格長寬顯示（僅在 taipei_6_1_test2 圖層顯示） -->
        <div
          v-if="isTaipei6_1Test2 && currentLayer && currentLayer.layoutGridJsonData_Test3"
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">layoutGridJsonData_Test3 當前網格尺寸</div>
          <div class="text-center">
            <div class="my-title-sm-black">
              {{ currentGridDimensions3.width }} × {{ currentGridDimensions3.height }}
            </div>
            <small class="text-muted">長 × 寬</small>
          </div>
        </div>

        <!-- 當前網格長寬顯示（僅在 taipei_6_1_test3 或 taipei_6_1_test4 圖層顯示） -->
        <div
          v-if="isTaipei6_1Test3 && currentLayer && currentLayer.layoutGridJsonData_Test4"
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">layoutGridJsonData_Test4 當前網格尺寸</div>
          <div class="text-center">
            <div class="my-title-sm-black">
              {{ currentGridDimensions4.width }} × {{ currentGridDimensions4.height }}
            </div>
            <small class="text-muted">長 × 寬</small>
          </div>
        </div>

        <!-- LayoutGridTab_Test4 滑鼠網格座標顯示（僅在 taipei_6_1_test3 或 taipei_6_1_test4 圖層顯示） -->
        <div
          v-if="isTaipei6_1Test3 && currentLayer && currentLayer.layoutGridJsonData_Test4"
          class="pb-3 mb-3 border-bottom"
        >
          <div class="my-title-xs-gray pb-2">滑鼠網格座標</div>
          <div class="text-center">
            <div
              v-if="
                layoutGridTabTest4MouseGridCoordinate.x !== null &&
                layoutGridTabTest4MouseGridCoordinate.y !== null
              "
              class="my-title-sm-black"
            >
              ({{ layoutGridTabTest4MouseGridCoordinate.x }},
              {{ layoutGridTabTest4MouseGridCoordinate.y }})
            </div>
            <div v-else class="my-title-xs-gray">請將滑鼠移至網格上</div>
          </div>
        </div>

        <!-- 合併路線和縮減網格按鈕（僅在 taipei_6_1_test3 圖層顯示） -->
        <div
          v-if="
            currentLayer &&
            currentLayer.layerId === 'taipei_6_1_test3' &&
            currentLayer.layoutGridJsonData_Test4
          "
          class="pb-3 mb-3 border-bottom"
        >
          <!-- 隨機產生權重 -->
          <button
            class="btn rounded-pill border-0 my-btn-orange my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-3"
            @click="randomizeWeights4"
          >
            隨機產生權重
          </button>

          <!-- LayoutGridTab_Test4：顯示/比例開關（樣式同 LeftView 圖層開關） -->
          <div class="mb-3">
            <div class="my-title-xs-gray pb-2">LayoutGridTab_Test4 顯示設定</div>

            <div class="d-flex align-items-center justify-content-between mb-2">
              <div class="my-content-sm-black">顯示網格</div>
              <div class="layer-toggle" @click.stop>
                <input
                  type="checkbox"
                  id="switch-test4-showGrid"
                  :checked="dataStore.showGrid"
                  @change="dataStore.setShowGrid($event.target.checked)"
                />
                <label for="switch-test4-showGrid"></label>
              </div>
            </div>

            <div class="d-flex align-items-center justify-content-between mb-2">
              <div class="my-content-sm-black">顯示權重</div>
              <div class="layer-toggle" @click.stop>
                <input
                  type="checkbox"
                  id="switch-test4-showWeightLabels"
                  :checked="dataStore.showWeightLabels"
                  @change="dataStore.setShowWeightLabels($event.target.checked)"
                />
                <label for="switch-test4-showWeightLabels"></label>
              </div>
            </div>

            <div class="d-flex align-items-center justify-content-between mb-2">
              <div class="my-content-sm-black">顯示粗細</div>
              <div class="layer-toggle" @click.stop>
                <input
                  type="checkbox"
                  id="switch-test4-showRouteThickness"
                  :checked="dataStore.showRouteThickness"
                  @change="dataStore.setShowRouteThickness($event.target.checked)"
                />
                <label for="switch-test4-showRouteThickness"></label>
              </div>
            </div>

            <div class="d-flex align-items-center justify-content-between mb-2">
              <div class="my-content-sm-black">權重放大</div>
              <div class="layer-toggle" @click.stop>
                <input
                  type="checkbox"
                  id="switch-test4-enableWeightScaling"
                  :checked="dataStore.enableWeightScaling"
                  @change="dataStore.setEnableWeightScaling($event.target.checked)"
                />
                <label for="switch-test4-enableWeightScaling"></label>
              </div>
            </div>

            <div class="d-flex align-items-center justify-content-between mb-2">
              <div class="my-content-sm-black">顯示站名</div>
              <div class="layer-toggle" @click.stop>
                <input
                  type="checkbox"
                  id="switch-test3-showStationNames"
                  :checked="dataStore.showStationNames"
                  @change="dataStore.setShowStationNames($event.target.checked)"
                />
                <label for="switch-test3-showStationNames"></label>
              </div>
            </div>

            <div class="d-flex align-items-center justify-content-between mb-2">
              <div class="my-content-sm-black">自動合併閾值 (pt)</div>
              <input
                type="number"
                min="0"
                step="0.1"
                :value="dataStore.autoMergeThreshold"
                @input="dataStore.setAutoMergeThreshold($event.target.value)"
                class="form-control form-control-sm"
                style="width: 80px; display: inline-block;"
              />
            </div>

            <div class="d-flex align-items-center justify-content-between mb-2">
              <div class="my-content-sm-black">權重放大倍數</div>
              <input
                type="number"
                min="1"
                step="1"
                :value="dataStore.weightScalingMultiplier"
                @input="dataStore.setWeightScalingMultiplier($event.target.value)"
                class="form-control form-control-sm"
                style="width: 80px; display: inline-block;"
              />
            </div>

            <div class="d-flex align-items-center justify-content-between">
              <div class="my-content-sm-black">縮放指數</div>
              <input
                type="number"
                min="0.1"
                max="10"
                step="0.1"
                :value="dataStore.weightScalingExponent"
                @input="dataStore.setWeightScalingExponent($event.target.value)"
                class="form-control form-control-sm"
                style="width: 80px; display: inline-block;"
              />
            </div>
          </div>

          <!-- gap <= 0 -->
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
              @click="mergeAllRoutes4(0)"
            >
              合併路線 (gap &lt;= 0)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeAllRoutes4H(0)"
            >
              合併路線-H (gap &lt;= 0)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
              @click="mergeAllRoutes4V(0)"
            >
              合併路線-V (gap &lt;= 0)
            </button>
          </div>
          <!-- gap <= 1 -->
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
              @click="mergeAllRoutes4(1)"
            >
              合併路線 (gap &lt;= 1)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeAllRoutes4H(1)"
            >
              合併路線-H (gap &lt;= 1)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
              @click="mergeAllRoutes4V(1)"
            >
              合併路線-V (gap &lt;= 1)
            </button>
          </div>
          <!-- gap <= 2 -->
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
              @click="mergeAllRoutes4(2)"
            >
              合併路線 (gap &lt;= 2)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeAllRoutes4H(2)"
            >
              合併路線-H (gap &lt;= 2)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
              @click="mergeAllRoutes4V(2)"
            >
              合併路線-V (gap &lt;= 2)
            </button>
          </div>
          <!-- gap <= 3 -->
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
              @click="mergeAllRoutes4(3)"
            >
              合併路線 (gap &lt;= 3)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeAllRoutes4H(3)"
            >
              合併路線-H (gap &lt;= 3)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
              @click="mergeAllRoutes4V(3)"
            >
              合併路線-V (gap &lt;= 3)
            </button>
          </div>
          <!-- 縮減網格 -->
          <button
            class="btn rounded-pill border-0 my-btn-red my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
            @click="reduceGrid4"
          >
            縮減網格
          </button>
        </div>

        <!-- 刪除 row/col（固定為 1pt，不刪除資料；僅 taipei_6_1_test2 顯示） -->
        <div
          v-if="isTaipei6_1Test2 && currentLayer && currentLayer.dataTableData"
          class="pb-3 mb-3 border-bottom"
        >
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-orange my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="deleteOneRow"
            >
              刪除row
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer"
              @click="deleteAllRows"
            >
              執行完成-row
            </button>
          </div>
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-orange my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="deleteOneCol"
            >
              刪除col
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer"
              @click="deleteAllCols"
            >
              執行完成-col
            </button>
          </div>
        </div>

        <!-- 隨機產生權重按鈕區域（僅在 taipei_6_1_test 圖層顯示） -->
        <div
          v-if="isTaipei6_1Test && currentLayer && currentLayer.layoutGridJsonData_Test"
          class="pb-3 mb-3 border-bottom"
        >
          <button
            class="btn rounded-pill border-0 my-btn-orange my-font-size-xs text-nowrap w-100 my-cursor-pointer"
            @click="randomizeWeights"
          >
            隨機產生權重
          </button>
        </div>

        <!-- 合併一筆路線2（放在隨機產生權重下方） -->
        <div
          v-if="isTaipei6_1Test && currentLayer && currentLayer.dataTableData"
          class="pb-3 mb-3 border-bottom"
        >
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-orange my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeOneRoute2AndReduce(4, 'V')"
            >
              合併一筆路線2-V (gap &lt;= 4)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeAllRoutes2AndReduce(4, 'V')"
            >
              執行完成2-V (gap &lt;= 4)
            </button>
          </div>
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-orange my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeOneRoute2AndReduce(4, 'H')"
            >
              合併一筆路線2-H (gap &lt;= 4)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer"
              @click="mergeAllRoutes2AndReduce(4, 'H')"
            >
              執行完成2-H (gap &lt;= 4)
            </button>
          </div>
        </div>

        <!-- 合併一筆路線按鈕區域（僅在 taipei_6_1_test 圖層顯示） -->
        <div
          v-if="isTaipei6_1Test && currentLayer && currentLayer.dataTableData"
          class="pb-3 mb-3 border-bottom"
        >
          <!-- gap <= 0 -->
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeOneRoute(0)"
            >
              合併一筆路線 (gap ＜= 0)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer"
              @click="mergeAllRoutes(0)"
            >
              執行完成 (gap ＜= 0)
            </button>
          </div>
          <!-- gap <= 1 -->
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeOneRoute(1)"
            >
              合併一筆路線 (gap ＜= 1)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer"
              @click="mergeAllRoutes(1)"
            >
              執行完成 (gap ＜= 1)
            </button>
          </div>
          <!-- gap <= 2 -->
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeOneRoute(2)"
            >
              合併一筆路線 (gap ＜= 2)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer"
              @click="mergeAllRoutes(2)"
            >
              執行完成 (gap ＜= 2)
            </button>
          </div>
          <!-- gap <= 3 -->
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeOneRoute(3)"
            >
              合併一筆路線 (gap ＜= 3)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer"
              @click="mergeAllRoutes(3)"
            >
              執行完成 (gap ＜= 3)
            </button>
          </div>
          <!-- gap <= 4 -->
          <div class="mb-2">
            <button
              class="btn rounded-pill border-0 my-btn-blue my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-1"
              @click="mergeOneRoute(4)"
            >
              合併一筆路線 (gap ＜= 4)
            </button>
            <button
              class="btn rounded-pill border-0 my-btn-green my-font-size-xs text-nowrap w-100 my-cursor-pointer mb-2"
              @click="mergeAllRoutes(4)"
            >
              執行完成 (gap ＜= 4)
            </button>
          </div>
          <!-- 縮減網格 -->
          <button
            class="btn rounded-pill border-0 my-btn-red my-font-size-xs text-nowrap w-100 my-cursor-pointer"
            @click="reduceGrid"
          >
            縮減網格
          </button>
        </div>

        <!-- 沒有可執行操作的提示 -->
        <div v-else-if="currentLayer" class="pb-3 mb-3">
          <div class="my-title-xs-gray text-center">
            {{
              currentLayer.executeFunction ? '此圖層不屬於 Taipei 群組' : '此圖層沒有可執行的操作'
            }}
          </div>
        </div>
      </div>
    </div>

    <!-- 沒有開啟的圖層 -->
    <div v-else class="d-flex align-items-center justify-content-center h-100">
      <div class="my-title-md-gray text-center p-3">沒有開啟的圖層</div>
    </div>
  </div>
</template>

<style scoped>
  /* 🎨 開關樣式：同 LeftView 的 LayersTab toggle（input + label） */
  .layer-toggle input[type='checkbox'] {
    height: 0;
    width: 0;
    visibility: hidden;
  }

  .layer-toggle label {
    cursor: pointer;
    width: 28px;
    height: 16px;
    background: var(--my-color-gray-300);
    display: block;
    border-radius: 16px;
    position: relative;
    transition: background-color 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .layer-toggle label:after {
    content: '';
    position: absolute;
    top: 2px;
    left: 2px;
    width: 12px;
    height: 12px;
    background: var(--my-color-white);
    border-radius: 12px;
    transition: transform 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .layer-toggle input:checked + label {
    background: var(--my-color-green);
  }

  .layer-toggle input:checked + label:after {
    transform: translateX(12px);
  }

  .layer-toggle input:disabled + label {
    cursor: not-allowed;
    opacity: 0.6;
  }

  .rag-chunks {
    margin: 0.5rem 0 0;
    padding: 0.5rem;
    background-color: #f8f9fa;
    border: 1px solid #dee2e6;
    border-radius: 4px;
    font-size: 10pt;
    line-height: 1.4;
    white-space: pre-wrap;
  }
</style>
