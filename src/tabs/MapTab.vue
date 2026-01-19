<script>
  import { ref, onMounted, onUnmounted, watch, nextTick, computed } from 'vue';
  import L from 'leaflet';
  import 'leaflet/dist/leaflet.css';
  import { useDataStore } from '@/stores/dataStore.js';
  import { useDefineStore } from '@/stores/defineStore.js';

  export default {
    name: 'MapTab',
    emits: ['active-layer-change'],
    props: {
      containerHeight: { type: Number, default: 500 },
      isPanelDragging: { type: Boolean, default: false },
    },
    setup(props, { emit }) {
      const dataStore = useDataStore();
      const mapStore = useDefineStore();

      const mapEl = ref(null);
      let map = null;
      // 為每個圖層存儲獨立的地圖狀態
      const layerStates = new Map(); // layerId -> { center, zoom, tileLayer, townshipLayer, isTownshipVisible, geojsonLayers }
      let currentLayerId = null;
      let currentTileLayer = null;
      let townshipBoundaryLayer = null;
      const MAX_RETRY_COUNT = 3; // 最多重試3次

      // Get all visible layers that have geojson data for map rendering
      const visibleGeojsonLayers = computed(() =>
        dataStore.getAllLayers().filter((l) => l.visible && (l.geojsonData || l.jsonData))
      );

      // Get all visible layers (including those without geojson data)
      const allVisibleLayers = computed(() => dataStore.getAllLayers().filter((l) => l.visible));

      const activeLeftTab = computed(() => dataStore.activeLeftTab);

      const ensureMap = () => {
        // 如果地圖已存在且有效，只需要刷新尺寸
        if (map && mapEl.value) {
          try {
            map.invalidateSize();
            return;
          } catch (err) {
            // 地圖對象無效，需要重新創建
            // eslint-disable-next-line no-console
            console.warn('Map object invalid, recreating...', err);
            try {
              map.remove();
            } catch (removeErr) {
              void removeErr;
            }
            map = null;
          }
        }

        if (!mapEl.value) return;

        // 清理 DOM 元素上可能殘留的 Leaflet 實例
        if (mapEl.value._leaflet_id) {
          // 移除所有 Leaflet 相關的屬性
          delete mapEl.value._leaflet_id;
          mapEl.value.innerHTML = '';
          // 移除所有 leaflet 相關的 class
          mapEl.value.className = mapEl.value.className
            .split(' ')
            .filter((c) => !c.startsWith('leaflet-'))
            .join(' ');
        }

        // 確保容器有尺寸
        const rect = mapEl.value.getBoundingClientRect();
        if (rect.width === 0 || rect.height === 0) {
          // 如果容器沒有尺寸，延遲重試
          setTimeout(() => {
            ensureMap();
          }, 100);
          return;
        }

        try {
          // 創建新地圖
          map = L.map(mapEl.value, {
            center: mapStore.mapView.center,
            zoom: mapStore.mapView.zoom,
            zoomControl: true,
            attributionControl: false,
          });

          setBasemap();

          // 確保地圖正確渲染
          nextTick(() => {
            if (map) {
              map.invalidateSize();
            }
          });
        } catch (err) {
          // eslint-disable-next-line no-console
          console.error('Failed to create map:', err);
          map = null;
        }
      };

      const setBasemap = () => {
        if (!map) return;

        if (currentTileLayer) {
          map.removeLayer(currentTileLayer);
          currentTileLayer = null;
        }

        const config = mapStore.basemaps.find((b) => b.value === mapStore.selectedBasemap);
        if (config && config.url) {
          currentTileLayer = L.tileLayer(config.url, { attribution: '' });
          currentTileLayer.addTo(map);
        }

        // Set background color - 默认设为白色
        if (mapEl.value) {
          if (mapStore.selectedBasemap === 'blank') {
            mapEl.value.style.backgroundColor = '#ffffff';
          } else if (mapStore.selectedBasemap === 'black') {
            mapEl.value.style.backgroundColor = '#000000';
          } else {
            // 将默认背景改为白色而不是透明
            mapEl.value.style.backgroundColor = '#ffffff';
          }
        }
      };

      const changeBasemap = (basemapType) => {
        mapStore.setSelectedBasemap(basemapType);
        setBasemap();
      };

      const getBasemapLabel = (value) => {
        const basemap = mapStore.basemaps.find((b) => b.value === value);
        return basemap ? basemap.label : value;
      };

      const toggleTownshipBoundary = () => {
        if (!map) return;

        if (!townshipBoundaryLayer) {
          townshipBoundaryLayer = L.tileLayer(
            'https://wmts.nlsc.gov.tw/wmts/TOWN/default/EPSG:3857/{z}/{y}/{x}.png',
            {
              attribution: '內政部國土測繪中心',
              maxZoom: 20,
              opacity: 1,
            }
          );
        }

        if (townshipBoundaryLayer && map.hasLayer(townshipBoundaryLayer)) {
          map.removeLayer(townshipBoundaryLayer);
        } else {
          townshipBoundaryLayer.addTo(map);
          if (townshipBoundaryLayer.setZIndex) {
            townshipBoundaryLayer.setZIndex(100);
          }
        }
      };

      const townshipBoundaryButtonLabel = computed(() => {
        if (!map) return '顯示鄉鎮區界';
        return townshipBoundaryLayer && map.hasLayer(townshipBoundaryLayer)
          ? '隱藏鄉鎮區界'
          : '顯示鄉鎮區界';
      });

      const showAllFeatures = () => {
        if (!map) return;
        const bounds = L.latLngBounds([]);
        let hasValidBounds = false;

        // 獲取當前地圖上所有的 geojson 圖層
        map.eachLayer((layer) => {
          if (
            layer &&
            layer.getBounds &&
            layer !== currentTileLayer &&
            layer !== townshipBoundaryLayer
          ) {
            const layerBounds = layer.getBounds();
            if (layerBounds.isValid()) {
              bounds.extend(layerBounds);
              hasValidBounds = true;
            }
          }
        });

        if (hasValidBounds) {
          map.fitBounds(bounds, { padding: [50, 50] });
        }
      };

      const showFullCity = () => {
        if (!map) return;
        map.setView(mapStore.mapView.center, mapStore.mapView.zoom);
      };

      const isAnyLayerVisible = computed(() => {
        if (!map) return false;
        let hasGeoJsonLayer = false;
        map.eachLayer((layer) => {
          if (layer !== currentTileLayer && layer !== townshipBoundaryLayer) {
            hasGeoJsonLayer = true;
          }
        });
        return hasGeoJsonLayer;
      });

      const fitBoundsIfAny = () => {
        if (!map) return;
        const bounds = L.latLngBounds([]);
        let has = false;
        map.eachLayer((layer) => {
          if (
            layer &&
            layer.getBounds &&
            layer !== currentTileLayer &&
            layer !== townshipBoundaryLayer
          ) {
            const layerBounds = layer.getBounds();
            if (layerBounds.isValid()) {
              bounds.extend(layerBounds);
              has = true;
            }
          }
        });
        if (has) map.fitBounds(bounds, { padding: [40, 40] });
      };

      const loadOrSyncLayers = async (retryAttempt = 0) => {
        if (!mapEl.value) return;

        // 確保地圖已初始化
        ensureMap();

        // 等待地圖完全初始化
        await nextTick();

        if (!map) {
          // 如果超過最大重試次數，停止重試
          if (retryAttempt >= MAX_RETRY_COUNT) {
            // eslint-disable-next-line no-console
            console.error('Map initialization failed after maximum retries');
            return;
          }
          // eslint-disable-next-line no-console
          console.warn(`Map not initialized, retrying... (${retryAttempt + 1}/${MAX_RETRY_COUNT})`);
          setTimeout(() => {
            loadOrSyncLayers(retryAttempt + 1);
          }, 100);
          return;
        }

        // Only load the layer for the current active tab
        const currentLayer = allVisibleLayers.value.find((l) => l.layerId === activeLayerTab.value);

        // Remove all geojson layers from map
        try {
          map.eachLayer((layer) => {
            if (layer && layer.options && layer.options.layerId) {
              map.removeLayer(layer);
            }
          });
        } catch (err) {
          // eslint-disable-next-line no-console
          console.error('Error removing layers:', err);
        }

        // Add the current layer's geojson if it exists
        if (currentLayer) {
          try {
            let geojson = null;

            // 優先使用 geojsonData（如果存在）
            if (currentLayer.geojsonData && currentLayer.geojsonData.features && Array.isArray(currentLayer.geojsonData.features)) {
              geojson = currentLayer.geojsonData;
            }
            // 如果沒有 geojsonData，但有 geojsonFileName，從文件載入
            else if (currentLayer.geojsonFileName) {
              const baseUrl = process.env.BASE_URL || '/';
              const dataPath = `${baseUrl}data/${currentLayer.geojsonFileName}`;

              try {
                const response = await fetch(dataPath);
                if (response.ok) {
                  geojson = await response.json();
                } else {
                  // 嘗試備用路徑
                  const fallbackPath = `/data/${currentLayer.geojsonFileName}`;
                  const fallbackResponse = await fetch(fallbackPath);
                  if (fallbackResponse.ok) {
                    geojson = await fallbackResponse.json();
                  }
                }
              } catch (fetchError) {
                // eslint-disable-next-line no-console
                console.warn('Failed to load GeoJSON from file:', currentLayer.geojsonFileName, fetchError);
              }
            }

            // 如果沒有找到 GeoJSON 數據，返回
            if (!geojson) {
              return;
            }

            // 驗證 GeoJSON 格式
            if (!geojson.features || !Array.isArray(geojson.features)) {
              console.warn('Invalid GeoJSON format:', currentLayer.layerId);
              return;
            }

            // 分離路線和車站
            const routeFeatures = geojson.features.filter(
              (f) => f.properties && f.properties.type === 'way' &&
                     (f.geometry.type === 'LineString' || f.geometry.type === 'MultiLineString')
            );
            const stationFeatures = geojson.features.filter(
              (f) => f.properties && f.properties.type === 'node' && f.geometry.type === 'Point'
            );

            // 創建路線圖層組
            const routeLayerGroup = L.layerGroup();
            routeLayerGroup.options.layerId = currentLayer.layerId;

            // 繪製路線
            routeFeatures.forEach((feature) => {
              const tags = feature.properties.tags || {};
              const routeColor = tags.color || '#666666';

              const routeLayer = L.geoJSON(feature, {
                style: {
                  color: routeColor,
                  weight: 3, // 細一點的路線
                  opacity: 0.9,
                  fillColor: routeColor,
                  fillOpacity: 0.8,
                },
                pane: 'overlayPane', // 確保路線在 overlayPane
              });

              // 添加 hover 事件顯示 tags
              routeLayer.eachLayer((layer) => {
                // 確保每個子圖層都在 overlayPane
                if (layer.setPane) {
                  layer.setPane('overlayPane');
                }

                const tagsHtml = Object.entries(tags)
                  .map(([key, value]) => `<strong>${key}:</strong> ${value}`)
                  .join('<br>');

                layer.bindPopup(`<div style="max-width: 300px;">${tagsHtml || '無標籤資訊'}</div>`, {
                  closeButton: true,
                });

                layer.on('mouseover', function() {
                  this.openPopup();
                });
              });

              routeLayerGroup.addLayer(routeLayer);
            });

            // 創建車站圖層組（使用較高的 pane 確保在上方）
            const stationLayerGroup = L.layerGroup();
            stationLayerGroup.options.layerId = currentLayer.layerId;

            // 繪製車站（黑點）
            stationFeatures.forEach((feature) => {
              const tags = feature.properties.tags || {};

              const stationLayer = L.geoJSON(feature, {
                pointToLayer: (feature, latlng) =>
                  L.circleMarker(latlng, {
                    radius: 1.5, // 直徑等於路線寬度（weight: 3）
                    color: '#000000',
                    weight: 1,
                    fillColor: '#000000',
                    fillOpacity: 1,
                    pane: 'markerPane', // 使用 markerPane 確保在路線上方
                  }),
              });

              // 添加 hover 事件顯示 tags
              stationLayer.eachLayer((layer) => {
                // 確保車站在 markerPane（在路線上方）
                if (layer.setPane) {
                  layer.setPane('markerPane');
                }

                const tagsHtml = Object.entries(tags)
                  .map(([key, value]) => `<strong>${key}:</strong> ${value}`)
                  .join('<br>');

                layer.bindPopup(`<div style="max-width: 300px;">${tagsHtml || '無標籤資訊'}</div>`, {
                  closeButton: true,
                });

                layer.on('mouseover', function() {
                  this.openPopup();
                });
              });

              stationLayerGroup.addLayer(stationLayer);
            });

            // 先添加路線圖層（下層）
            routeLayerGroup.addTo(map);
            // 再添加車站圖層（上層）
            stationLayerGroup.addTo(map);

            // Fit bounds to the loaded layer
            await nextTick();
            fitBoundsIfAny();
          } catch (e) {
            // eslint-disable-next-line no-console
            console.error('Load GeoJSON from jsonData failed:', currentLayer.layerId, e);
          }
        }
      };

      const invalidateSize = () => {
        if (!map) return;
        nextTick(() => {
          try {
            map.invalidateSize();
          } catch (err) {
            void err;
          }
        });
      };

      onMounted(() => {
        // 延遲初始化，確保 DOM 已準備好
        nextTick(() => {
          setTimeout(() => {
            // 初始化地圖，不管是否有可見圖層
            ensureMap();

            // 如果有可見圖層，載入圖層數據
            if (allVisibleLayers.value.length > 0) {
              loadOrSyncLayers();
            }

            // 確保地圖尺寸正確
            setTimeout(() => {
              invalidateSize();
            }, 200);
          }, 100);
        });
      });

      // Watch basemap changes
      watch(
        () => mapStore.selectedBasemap,
        () => {
          if (map) {
            setBasemap();
          }
        }
      );

      onUnmounted(() => {
        // 清理地圖實例
        if (map) {
          try {
            map.remove();
          } catch (err) {
            void err;
          }
          map = null;
        }

        // 清理圖層狀態
        layerStates.clear();
        currentLayerId = null;
        currentTileLayer = null;
        townshipBoundaryLayer = null;

        // 清理 DOM 元素
        if (mapEl.value) {
          if (mapEl.value._leaflet_id) {
            delete mapEl.value._leaflet_id;
          }
          mapEl.value.innerHTML = '';
        }
      });

      watch(
        () => dataStore.layers,
        () => {
          loadOrSyncLayers();
        },
        { deep: true }
      );

      // Watch for changes in all visible layers (including those without geojson)
      watch(
        () => allVisibleLayers.value.length,
        (newLength, oldLength) => {
          // 當從沒有圖層變為有圖層時，強制重新創建地圖
          if (oldLength === 0 && newLength > 0) {
            // 清理舊的地圖實例
            if (map) {
              try {
                map.remove();
              } catch (err) {
                void err;
              }
              map = null;
            }

            nextTick(() => {
              setTimeout(() => {
                ensureMap();
                loadOrSyncLayers();
                setTimeout(() => {
                  invalidateSize();
                }, 100);
              }, 100);
            });
          } else if (newLength > 0) {
            // 正常情況下的更新
            nextTick(() => {
              // 確保地圖已初始化
              ensureMap();

              // 如果有可見圖層，載入圖層數據
              setTimeout(() => {
                loadOrSyncLayers();
              }, 50);

              // 確保地圖尺寸正確
              setTimeout(() => {
                invalidateSize();
              }, 100);
            });
          }
        }
      );

      // Tab functionality similar to D3jsTab
      const activeLayerTab = ref(null);

      // Use allVisibleLayers for tab functionality (includes layers without geojson)
      const visibleLayers = allVisibleLayers;

      // Set active layer tab
      const setActiveLayerTab = (layerId) => {
        if (activeLayerTab.value === layerId) {
          return;
        }

        // 保存當前地圖狀態
        if (map && currentLayerId) {
          try {
            const stateToSave = {
              center: map.getCenter(),
              zoom: map.getZoom(),
              isTownshipVisible: townshipBoundaryLayer
                ? map.hasLayer(townshipBoundaryLayer)
                : false,
              selectedBasemap: mapStore.selectedBasemap,
            };
            const existingState = layerStates.get(currentLayerId) || {};
            layerStates.set(currentLayerId, { ...existingState, ...stateToSave });
          } catch (err) {
            // eslint-disable-next-line no-console
            console.warn('Error saving layer state:', err);
          }
        }

        // 切換到新的圖層
        activeLayerTab.value = layerId;
        currentLayerId = layerId;

        // 通知父層目前 UpperView 的作用圖層
        emit('active-layer-change', activeLayerTab.value);

        // 確保地圖已初始化
        nextTick(() => {
          ensureMap();

          // 恢復圖層狀態（底圖）
          if (layerId && layerStates.has(layerId)) {
            const layerState = layerStates.get(layerId);
            if (
              layerState.selectedBasemap &&
              layerState.selectedBasemap !== mapStore.selectedBasemap
            ) {
              mapStore.setSelectedBasemap(layerState.selectedBasemap);
            }
          }

          // 載入當前圖層的 GeoJSON 數據
          setTimeout(() => {
            loadOrSyncLayers();

            // 恢復圖層狀態（視圖和邊界）
            if (layerId && layerStates.has(layerId) && map) {
              const layerState = layerStates.get(layerId);
              try {
                map.setView(layerState.center, layerState.zoom);

                // 恢復鄉鎮區界狀態
                if (
                  layerState.isTownshipVisible &&
                  (!townshipBoundaryLayer || !map.hasLayer(townshipBoundaryLayer))
                ) {
                  if (!townshipBoundaryLayer) {
                    townshipBoundaryLayer = L.tileLayer(
                      'https://wmts.nlsc.gov.tw/wmts/TOWN/default/EPSG:3857/{z}/{y}/{x}.png',
                      {
                        attribution: '內政部國土測繪中心',
                        maxZoom: 20,
                        opacity: 1,
                      }
                    );
                  }
                  townshipBoundaryLayer.addTo(map);
                  if (townshipBoundaryLayer.setZIndex) {
                    townshipBoundaryLayer.setZIndex(100);
                  }
                } else if (
                  !layerState.isTownshipVisible &&
                  townshipBoundaryLayer &&
                  map.hasLayer(townshipBoundaryLayer)
                ) {
                  map.removeLayer(townshipBoundaryLayer);
                }
              } catch (err) {
                // eslint-disable-next-line no-console
                console.warn('Error restoring layer state:', err);
              }
            }
          }, 100);
        });
      };

      // Get layer full title with group name
      const getLayerFullTitle = (layer) => {
        if (!layer) return { groupName: null, layerName: '未知圖層' };
        const groupName = dataStore.findGroupNameByLayerId(layer.layerId);
        return {
          groupName: groupName,
          layerName: layer.layerName,
        };
      };

      // Watch for visible layers changes and auto-select first layer
      watch(
        () => visibleLayers.value,
        (newLayers) => {
          if (newLayers.length === 0) {
            activeLayerTab.value = null;
            currentLayerId = null;
            return;
          }
          if (
            !activeLayerTab.value ||
            !newLayers.find((layer) => layer.layerId === activeLayerTab.value)
          ) {
            setActiveLayerTab(newLayers[0].layerId);
          }
        },
        { deep: true, immediate: true }
      );

      return {
        mapEl,
        invalidateSize,
        selectedBasemap: computed(() => mapStore.selectedBasemap),
        changeBasemap,
        getBasemapLabel,
        toggleTownshipBoundary,
        townshipBoundaryButtonLabel,
        showAllFeatures,
        showFullCity,
        isAnyLayerVisible,
        mapStore,
        visibleGeojsonLayers,
        allVisibleLayers,
        activeLeftTab,
        // Tab functionality
        activeLayerTab,
        visibleLayers,
        setActiveLayerTab,
        getLayerFullTitle,
      };
    },
  };
</script>

<template>
  <div class="d-flex flex-column h-100">
    <!-- 📑 圖層分頁導航 -->
    <div v-if="visibleLayers.length > 0" class="">
      <ul class="nav nav-tabs nav-fill">
        <li
          v-for="layer in visibleLayers"
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
            <span>
              <span v-if="getLayerFullTitle(layer).groupName" class="my-title-xs-gray"
                >{{ getLayerFullTitle(layer).groupName }} -
              </span>
              <span class="my-title-sm-black">{{ getLayerFullTitle(layer).layerName }}</span>
            </span>
          </div>
          <div class="w-100" :class="`my-bgcolor-${layer.colorName}`" style="min-height: 4px"></div>
        </li>
      </ul>
    </div>

    <!-- 有開啟圖層時的內容 -->
    <div
      v-if="allVisibleLayers.length > 0"
      class="flex-grow-1 d-flex flex-column my-bgcolor-white"
      style="min-height: 0"
    >
      <!-- 地圖容器 -->
      <div class="flex-grow-1 position-relative">
        <div ref="mapEl" class="w-100 h-100 map-container"></div>

        <!-- 地圖底部控制項區域 -->
        <div
          class="position-absolute map-bottom-controls d-flex align-items-center rounded-pill shadow my-blur gap-2 p-2 mb-3"
        >
          <div class="d-flex align-items-center">
            <div class="dropdown dropup">
              <button
                class="btn rounded-pill border-0 my-btn-transparent my-font-size-xs text-nowrap"
                type="button"
                data-bs-toggle="dropdown"
                aria-expanded="false"
              >
                {{ getBasemapLabel(selectedBasemap) }}
              </button>
              <ul class="dropdown-menu">
                <li v-for="basemap in mapStore.basemaps" :key="basemap.value">
                  <a
                    class="dropdown-item my-content-xs-black py-1"
                    href="#"
                    @click.prevent="changeBasemap(basemap.value)"
                  >
                    {{ basemap.label }}
                  </a>
                </li>
              </ul>
            </div>
          </div>

          <!-- 顯示鄉鎮區界 -->
          <button
            class="btn rounded-pill border-0 my-btn-transparent my-font-size-xs text-nowrap my-cursor-pointer"
            :title="
              townshipBoundaryButtonLabel === '隱藏鄉鎮區界'
                ? '隱藏鄉鎮區界圖層'
                : '顯示鄉鎮區界圖層'
            "
            @click="toggleTownshipBoundary"
          >
            {{ townshipBoundaryButtonLabel }}
          </button>

          <!-- 顯示全部 -->
          <button
            class="btn rounded-pill border-0 my-btn-transparent my-font-size-xs text-nowrap my-cursor-pointer"
            @click="showAllFeatures"
            :disabled="!isAnyLayerVisible"
            title="顯示圖面所有資料範圍"
          >
            顯示全部
          </button>

          <!-- 顯示全市 -->
          <button
            class="btn rounded-pill border-0 my-btn-transparent my-font-size-xs text-nowrap my-cursor-pointer"
            @click="showFullCity"
            title="回到預設地圖範圍"
          >
            顯示全市
          </button>
        </div>
      </div>
    </div>

    <!-- 沒有開啟圖層時的空狀態 -->
    <div v-else class="flex-grow-1 d-flex align-items-center justify-content-center">
      <div class="text-center">
        <div class="my-title-md-gray p-3">沒有開啟的圖層</div>
      </div>
    </div>
  </div>
</template>

<style scoped>
  .w-100 {
    width: 100%;
  }
  .h-100 {
    height: 100%;
  }

  /* 地圖容器樣式 */
  .map-container {
    position: relative;
    background-color: #f0f0f0;
  }

  /* 地圖底部控制項樣式 */
  .map-bottom-controls {
    bottom: 0px;
    left: 50%;
    transform: translateX(-50%);
    z-index: 2000;
  }
</style>
