/** * 🌟 上半部區域組件 (Upper Area Component) * * 功能說明 (Features): * 1. 📊
分頁管理：管理儀表板和 D3.js 圖表的分頁切換 * 2. 🗺️ 地圖顯示：提供地圖視覺化和互動功能 * 3. 📈
數據視覺化：整合 D3.js 進行各種圖表繪製 * 4. 📱 響應式設計：適配不同螢幕尺寸的顯示需求 * 5. 🎯
高亮功能：提供地圖要素高亮顯示功能 * 6. 🔄 狀態同步：與父組件保持狀態同步 * * 技術特點 (Technical
Features): * - 使用 Vue 2 Options API 進行組件管理 * - 整合多個分頁組件，提供統一的介面 * -
支援響應式佈局和動態尺寸調整 * - 提供完整的事件處理和狀態管理 * - 整合地圖和圖表視覺化功能 * *
包含分頁 (Included Tabs): * - DashboardTab：儀表板分頁，顯示統計圖表和摘要資訊 * - D3jsTab：D3.js
圖表分頁，提供進階數據視覺化 * * @file UpperView.vue * @version 2.0.0 * @author Kevin Cheng * @since
1.0.0 */
<script>
  // ==================== 📦 第三方庫引入 (Third-Party Library Imports) ====================

  /**
   * Vue Composition API 核心功能引入
   * 提供響應式數據、生命週期鉤子等功能
   *
   * @see https://vuejs.org/
   */
  import { ref, watch, nextTick, computed } from 'vue';

  // ==================== 🧩 子組件引入 (Subcomponent Imports) ====================

  /**
   * 儀表板分頁組件引入
   * 提供統計圖表和數據摘要顯示功能
   *
   * @see ../tabs/DashboardTab.vue
   */
  import DashboardTab from '../tabs/DashboardTab.vue';

  /**
   * 網格縮放分頁組件引入
   * 提供網格縮放視覺化功能
   *
   * @see ../tabs/GridScalingTab.vue
   */
  import GridScalingTab from '../tabs/GridScalingTab.vue';

  /**
   * 處理後 JSON 數據分頁組件引入
   * 顯示圖層的處理後 JSON 數據
   *
   * @see ../tabs/ProcessedJsonDataTab.vue
   */
  import ProcessedJsonDataTab from '../tabs/ProcessedJsonDataTab.vue';

  /**
   * 原始 JSON 數據分頁組件引入
   * 顯示圖層的原始 JSON 數據
   *
   * @see ../tabs/JsonDataTab.vue
   */
  import JsonDataTab from '../tabs/JsonDataTab.vue';

  /**
   * 繪製數據分頁組件引入
   * 顯示圖層的繪製數據
   *
   * @see ../tabs/DrawJsonDataTab.vue
   */
  import DrawJsonDataTab from '../tabs/DrawJsonDataTab.vue';
  import { getIcon } from '../utils/utils.js';
  import { useDataStore } from '../stores/dataStore.js';

  export default {
    name: 'UpperView',

    /**
     * 🧩 組件註冊 (Component Registration)
     * 註冊上半部面板內使用的子組件
     */
    components: {
      DashboardTab,
      GridScalingTab,
      ProcessedJsonDataTab,
      JsonDataTab,
      DrawJsonDataTab,
    },

    /**
     * 🔧 組件屬性定義 (Component Props)
     * 接收來自父組件的配置和狀態數據
     */
    props: {
      activeUpperTab: { type: String, default: 'grid-scaling' },
      mainPanelWidth: { type: Number, default: 60 },
      contentHeight: { type: Number, default: 500 },
      selectedFilter: { type: String, default: '' },
      zoomLevel: { type: Number, default: 11 },
      isPanelDragging: { type: Boolean, default: false },
      activeMarkers: { type: Number, default: 0 },
    },

    /**
     * 📡 組件事件定義 (Component Events)
     * 定義向父組件發送的事件類型
     */
    emits: [
      'update:activeUpperTab', // 更新作用中分頁
      'update:zoomLevel', // 更新地圖縮放等級
      'update:currentCoords', // 更新當前座標
      'update:activeMarkers', // 更新作用中標記數量
      'feature-selected', // 選中地圖特徵
    ],

    /**
     * 🔧 組件設定函數 (Component Setup)
     * 使用 Composition API 設定組件邏輯
     */
    setup(props, { emit }) {
      const dataStore = useDataStore();
      // 📚 子組件引用 (Child Component References)
      /** 📊 儀表板視圖組件引用 */
      const DashboardTab = ref(null);
      /** 📊 儀表板容器引用 (用於控制滑鼠事件) */
      const dashboardContainerRef = ref(null);
      /** 📊 網格縮放視圖組件引用 */
      const GridScalingTab = ref(null);
      /** 📊 網格縮放容器引用 (用於控制滑鼠事件) */
      const gridScalingContainerRef = ref(null);
      /** 📊 處理後 JSON 數據組件引用 */
      const ProcessedJsonDataTab = ref(null);
      /** 📊 處理後 JSON 數據容器引用 */
      const processedJsonDataContainerRef = ref(null);
      /** 📊 原始 JSON 數據組件引用 */
      const JsonDataTab = ref(null);
      /** 📊 原始 JSON 數據容器引用 */
      const jsonDataContainerRef = ref(null);
      /** 📊 繪製 JSON 數據組件引用 */
      const DrawJsonDataTab = ref(null);
      /** 📊 繪製 JSON 數據容器引用 */
      const drawJsonDataContainerRef = ref(null);

      // 目前 UpperView 所選圖層（由各子 Tab 回傳）
      const activeUpperLayerId = ref(null);

      // 所有可能的 tabs 列表
      const allPossibleTabs = [
        'grid-scaling',
        'dashboard',
        'processed-json-data',
        'json-data',
        'draw-json-data',
      ];

      // 計算每個 tab 是否啟用（基於當前激活圖層的 upperViewTabs）
      const isTabEnabled = computed(() => {
        const enabledMap = {};
        const layer = activeUpperLayerId.value
          ? dataStore.findLayerById(activeUpperLayerId.value)
          : null;
        const allowedTabs = Array.isArray(layer?.upperViewTabs) ? layer.upperViewTabs : [];

        allPossibleTabs.forEach((tabId) => {
          enabledMap[tabId] = allowedTabs.includes(tabId);
        });

        return enabledMap;
      });

      const handleActiveLayerChange = (layerId) => {
        activeUpperLayerId.value = layerId;
      };

      // ==================== 🔄 左側開啟圖層 → UpperView 自動切換 (Auto switch on newly opened layer) ====================
      // 目標：在 LeftView 開啟圖層時，UpperView 直接切到該圖層，並優先顯示可視化分頁（grid-scaling）
      const visibleLayers = computed(() => dataStore.getAllLayers().filter((l) => l.visible));

      // 檢查是否有可見圖層
      const hasVisibleLayers = computed(() => visibleLayers.value.length > 0);
      const previousVisibleLayerIds = ref([]);
      let isUpdatingTab = false; // 防止遞迴更新的標誌（在兩個 watch 之間共享）

      watch(
        () => visibleLayers.value.map((l) => l.layerId),
        (newIds) => {
          // 防止遞迴更新
          if (isUpdatingTab) return;

          // 沒有可見圖層
          if (!Array.isArray(newIds) || newIds.length === 0) {
            activeUpperLayerId.value = null;
            previousVisibleLayerIds.value = [];
            return;
          }

          const prevIds = previousVisibleLayerIds.value || [];
          const added = newIds.filter((id) => !prevIds.includes(id));

          // 如果有新增圖層：切到最新新增的圖層
          if (added.length > 0) {
            const newestAddedLayerId = added[added.length - 1];
            activeUpperLayerId.value = newestAddedLayerId;

            // 優先顯示可視化分頁（grid-scaling）
            if (props.activeUpperTab !== 'grid-scaling') {
              const layer = dataStore.findLayerById(newestAddedLayerId);
              const tabs = Array.isArray(layer?.upperViewTabs) ? layer.upperViewTabs : [];
              if (tabs.includes('grid-scaling')) {
                isUpdatingTab = true;
                emit('update:activeUpperTab', 'grid-scaling');
                nextTick(() => {
                  isUpdatingTab = false;
                });
              }
            }
          }
          // 如果目前選中的圖層不在可見列表中：回到第一個可見圖層
          else if (!activeUpperLayerId.value || !newIds.includes(activeUpperLayerId.value)) {
            activeUpperLayerId.value = newIds[0];
          }

          previousVisibleLayerIds.value = [...newIds];
        },
        { immediate: true }
      );

      /**
       * 👀 監聽激活圖層變化
       * 當圖層變化時，如果當前 tab 不在新圖層的 upperViewTabs 中，切換到第一個可用的 tab
       * 注意：只監聽 activeUpperLayerId，避免同時監聽 activeUpperTab 造成遞迴更新
       */
      watch(
        () => activeUpperLayerId.value,
        (layerId) => {
          if (!layerId || isUpdatingTab) return;

          const layer = dataStore.findLayerById(layerId);
          const allowedTabs = Array.isArray(layer?.upperViewTabs) ? layer.upperViewTabs : [];
          const currentTab = props.activeUpperTab;

          // 如果當前 tab 不在允許列表中，切換到第一個可用的 tab
          if (currentTab && allowedTabs.length > 0 && !allowedTabs.includes(currentTab)) {
            const next = allowedTabs[0] || null;
            if (next && next !== currentTab) {
              isUpdatingTab = true;
              emit('update:activeUpperTab', next);
              // 使用 nextTick 確保更新完成後重置標誌
              nextTick(() => {
                isUpdatingTab = false;
              });
            }
          }
        },
        { immediate: true }
      );

      /**
       * 👀 監聽拖曳狀態和分頁變化 (Watch Dragging State and Tab Changes)
       * 調整儀表板容器的滑鼠指標事件，防止拖曳時的干擾
       */
      watch(
        [() => props.isPanelDragging, () => props.activeUpperTab],
        ([dragging, tab]) => {
          nextTick(() => {
            // 處理儀表板容器
            if (dashboardContainerRef.value) {
              if (dragging && tab === 'dashboard') {
                // 拖曳時禁用儀表板的滑鼠事件
                dashboardContainerRef.value.style.pointerEvents = 'none';
              } else {
                // 恢復儀表板的滑鼠事件
                dashboardContainerRef.value.style.pointerEvents = 'auto';
              }
            }

            // 處理網格縮放容器
            if (gridScalingContainerRef.value) {
              if (dragging && tab === 'grid-scaling') {
                // 拖曳時禁用網格縮放容器的滑鼠事件
                gridScalingContainerRef.value.style.pointerEvents = 'none';
              } else {
                // 恢復網格縮放容器的滑鼠事件
                gridScalingContainerRef.value.style.pointerEvents = 'auto';
              }
            }

            // 處理處理後 JSON 數據容器
            if (processedJsonDataContainerRef.value) {
              if (dragging && tab === 'processed-json-data') {
                processedJsonDataContainerRef.value.style.pointerEvents = 'none';
              } else {
                processedJsonDataContainerRef.value.style.pointerEvents = 'auto';
              }
            }

            // 處理原始 JSON 數據容器
            if (jsonDataContainerRef.value) {
              if (dragging && tab === 'json-data') {
                jsonDataContainerRef.value.style.pointerEvents = 'none';
              } else {
                jsonDataContainerRef.value.style.pointerEvents = 'auto';
              }
            }

            // 處理繪製 JSON 數據容器
            if (drawJsonDataContainerRef.value) {
              if (dragging && tab === 'draw-json-data') {
                drawJsonDataContainerRef.value.style.pointerEvents = 'none';
              } else {
                drawJsonDataContainerRef.value.style.pointerEvents = 'auto';
              }
            }

          });
        },
        { immediate: true }
      ); // immediate: true 表示立即執行一次

      /**
       * 👀 監聽分頁變化 (Watch Tab Changes)
       * 當切換分頁時觸發相應的更新動作，確保組件正常顯示
       */
      watch(
        () => props.activeUpperTab,
        (newTab) => {
          if (newTab === 'grid-scaling') {
            nextTick(() => {
              setTimeout(() => {
                if (GridScalingTab.value && GridScalingTab.value.resize) {
                  GridScalingTab.value.resize();
                }
              }, 100); // 給容器一些時間來完成顯示動畫
            });
          }
        }
      );

      /**
       * 👀 監聽面板大小變化 (Watch Panel Size Changes)
       * 當面板寬度或高度變化時，更新相應的子組件
       */
      watch([() => props.mainPanelWidth, () => props.contentHeight], () => {
        nextTick(() => {
          // 觸發各個 Tab 重新調整以適應新的容器尺寸
          if (GridScalingTab.value && GridScalingTab.value.resize) {
            GridScalingTab.value.resize();
          }
        });
      });

      /**
       * 📏 使地圖尺寸失效 (Invalidate Map Size)
       * 強制重新計算地圖尺寸並重繪示意圖
       * 用於響應容器尺寸變化
       */
      const invalidateMapSize = () => {
        // 觸發 GridScalingTab 重新繪製
        if (GridScalingTab.value && GridScalingTab.value.resize) {
          GridScalingTab.value.resize();
        }

        // 觸發全域 resize 事件作為備用方案
        setTimeout(() => {
          const event = new Event('resize');
          window.dispatchEvent(event);
        }, 50);
      };

      return {
        DashboardTab, // 儀表板組件引用
        GridScalingTab, // 網格縮放組件引用
        ProcessedJsonDataTab, // 處理後 JSON 數據組件引用
        JsonDataTab, // 原始 JSON 數據組件引用
        DrawJsonDataTab, // 繪製 JSON 數據組件引用
        dashboardContainerRef, // 儀表板容器引用
        gridScalingContainerRef, // 網格縮放容器引用
        processedJsonDataContainerRef, // 處理後 JSON 數據容器引用
        jsonDataContainerRef, // 原始 JSON 數據容器引用
        drawJsonDataContainerRef, // 繪製 JSON 數據容器引用
        invalidateMapSize, // 刷新地圖尺寸功能

        // 🛠️ 工具函數
        getIcon, // 圖標獲取函數

        // 動態分頁可用性
        isTabEnabled, // 每個 tab 是否啟用的映射
        handleActiveLayerChange,
        activeUpperLayerId, // 當前選中的圖層 ID
        hasVisibleLayers, // 是否有可見圖層
      };
    },
  };
</script>

<template>
  <div class="d-flex flex-column h-100 overflow-hidden">
    <!-- 📑 分頁導航按鈕 -->
    <!-- 顯示所有 tabs，沒有圖層支持的 tabs 會被禁用 -->
    <div class="d-flex justify-content-start my-bgcolor-gray-200 p-3">
      <div class="d-flex align-items-center rounded-pill shadow my-blur gap-1 p-2">
        <!-- 📊 網格縮放按鈕 (Grid Scaling Button) -->
        <button
          class="btn rounded-circle border-0 d-flex align-items-center justify-content-center my-btn-transparent my-font-size-xs"
          :class="{
            'my-btn-blue': activeUpperTab === 'grid-scaling',
          }"
          :disabled="!isTabEnabled['grid-scaling']"
          @click="$emit('update:activeUpperTab', 'grid-scaling')"
          title="網格縮放視覺化"
          style="width: 30px; height: 30px"
        >
          <i :class="getIcon('chart_line').icon"></i>
        </button>
        <!-- 📄 處理後 JSON 數據按鈕 (Processed JSON Data Button) -->
        <button
          class="btn rounded-circle border-0 d-flex align-items-center justify-content-center my-btn-transparent my-font-size-xs"
          :class="{
            'my-btn-blue': activeUpperTab === 'processed-json-data',
          }"
          :disabled="!isTabEnabled['processed-json-data']"
          @click="$emit('update:activeUpperTab', 'processed-json-data')"
          title="處理後 JSON 數據"
          style="width: 30px; height: 30px"
        >
          <i :class="getIcon('code').icon"></i>
        </button>
        <!-- 📄 原始 JSON 數據按鈕 (Original JSON Data Button) -->
        <button
          class="btn rounded-circle border-0 d-flex align-items-center justify-content-center my-btn-transparent my-font-size-xs"
          :class="{
            'my-btn-blue': activeUpperTab === 'json-data',
          }"
          :disabled="!isTabEnabled['json-data']"
          @click="$emit('update:activeUpperTab', 'json-data')"
          title="原始 JSON 數據"
          style="width: 30px; height: 30px"
        >
          <i :class="getIcon('json_data').icon"></i>
        </button>
        <!-- 🎨 繪製數據按鈕 (Draw Data Button) -->
        <button
          class="btn rounded-circle border-0 d-flex align-items-center justify-content-center my-btn-transparent my-font-size-xs"
          :class="{
            'my-btn-blue': activeUpperTab === 'draw-json-data',
          }"
          :disabled="!isTabEnabled['draw-json-data']"
          @click="$emit('update:activeUpperTab', 'draw-json-data')"
          title="繪製數據"
          style="width: 30px; height: 30px"
        >
          <i :class="getIcon('draw_data').icon"></i>
        </button>
        <!-- 📊 儀表板按鈕 (Dashboard Button) -->
        <button
          class="btn rounded-circle border-0 d-flex align-items-center justify-content-center my-btn-transparent my-font-size-xs"
          :class="{
            'my-btn-blue': activeUpperTab === 'dashboard',
          }"
          :disabled="!isTabEnabled['dashboard']"
          @click="$emit('update:activeUpperTab', 'dashboard')"
          title="資料儀表板"
          style="width: 30px; height: 30px"
        >
          <i :class="getIcon('chart_bar').icon"></i>
        </button>
      </div>
    </div>

    <!-- 📄 分頁內容區域 -->
    <div class="flex-grow-1 overflow-auto my-bgcolor-gray-200">
      <!-- 沒有開啟圖層的提示 -->
      <div v-if="!hasVisibleLayers" class="h-100 d-flex align-items-center justify-content-center">
        <div class="text-center">
          <div class="my-title-md-gray">沒有開啟的圖層</div>
        </div>
      </div>

      <!-- 網格縮放分頁內容 -->
      <div v-show="hasVisibleLayers && activeUpperTab === 'grid-scaling'" ref="gridScalingContainerRef" class="h-100">
        <GridScalingTab
          ref="GridScalingTab"
          class="flex-grow-1 d-flex flex-column"
          :containerHeight="contentHeight"
          :isPanelDragging="isPanelDragging"
          :activeMarkers="activeMarkers"
          @active-layer-change="handleActiveLayerChange"
        />
      </div>

      <!-- 儀表板分頁內容 -->
      <div v-show="hasVisibleLayers && activeUpperTab === 'dashboard'" ref="dashboardContainerRef" class="h-100">
        <DashboardTab
          ref="DashboardTab"
          :containerHeight="contentHeight"
          :isPanelDragging="isPanelDragging"
          :activeMarkers="activeMarkers"
          @active-layer-change="handleActiveLayerChange"
        />
      </div>

      <!-- 處理後 JSON 數據分頁內容 -->
      <div
        v-show="hasVisibleLayers && activeUpperTab === 'processed-json-data'"
        ref="processedJsonDataContainerRef"
        class="h-100"
      >
        <ProcessedJsonDataTab
          ref="ProcessedJsonDataTab"
          :containerHeight="contentHeight"
          :isPanelDragging="isPanelDragging"
          :activeMarkers="activeMarkers"
          @active-layer-change="handleActiveLayerChange"
        />
      </div>

      <!-- 原始 JSON 數據分頁內容 -->
      <div v-show="hasVisibleLayers && activeUpperTab === 'json-data'" ref="jsonDataContainerRef" class="h-100">
        <JsonDataTab
          ref="JsonDataTab"
          :containerHeight="contentHeight"
          :isPanelDragging="isPanelDragging"
          :activeMarkers="activeMarkers"
          @active-layer-change="handleActiveLayerChange"
        />
      </div>

      <!-- 繪製數據分頁內容 -->
      <div
        v-show="hasVisibleLayers && activeUpperTab === 'draw-json-data'"
        ref="drawJsonDataContainerRef"
        class="h-100"
      >
        <DrawJsonDataTab
          ref="DrawJsonDataTab"
          :containerHeight="contentHeight"
          :isPanelDragging="isPanelDragging"
          :activeMarkers="activeMarkers"
          @active-layer-change="handleActiveLayerChange"
        />
      </div>

    </div>
  </div>
</template>

<style scoped></style>
