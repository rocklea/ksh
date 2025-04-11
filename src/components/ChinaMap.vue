<template>
  <div class="container">
    <div v-if="loading" class="loading">加载中...</div>
    <div v-if="error" class="error">{{ error }}</div>
    <div ref="mapContainer" class="map-container"></div>
    <div v-if="!loading && !error" class="camera-info">
      <h3>当前区域: {{ currentArea }}</h3>
      <div class="button-group">
        <button v-if="currentArea !== '湖南省'" @click="backToProvince">返回湖南省</button>
        <button @click="resetCamera">复位视角</button>
      </div>
      <p>距离: {{ cameraDistance }}</p>
      <p>俯仰角: {{ cameraAlpha }}°</p>
      <p>方位角: {{ cameraBeta }}°</p>
      <p>中心坐标: [{{ cameraCenter.join(', ') }}]</p>
    </div>
  </div>
</template>

<script>
import * as echarts from 'echarts';
import 'echarts-gl';
import { onMounted, onUnmounted, ref, nextTick } from 'vue';
import hunanMapData from '../map/430000.json';
import HNImg from '../assets/textures/wenli.png';

// 使用数组形式存储城市坐标数据
const cityCoordinates = [
  ['长沙市', 112.982279, 28.19409, '430100'],
  ['株洲市', 113.151737, 27.835806, '430200'],
  ['湘潭市', 112.944052, 27.82973, '430300'],
  ['衡阳市', 112.607693, 26.900358, '430400'],
  ['邵阳市', 111.46923, 27.237842, '430500'],
  ['岳阳市', 113.132855, 29.37029, '430600'],
  ['常德市', 111.691347, 29.040225, '430700'],
  ['张家界市', 110.479921, 29.127401, '430800'],
  ['益阳市', 112.355042, 28.570066, '430900'],
  ['郴州市', 113.032067, 25.793589, '431000'],
  ['永州市', 111.608019, 26.434516, '431100'],
  ['怀化市', 109.97824, 27.550082, '431200'],
  ['娄底市', 111.994482, 27.700062, '431300'],
  ['湘西土家族苗族自治州', 109.739735, 28.314296, '433100']
];

// 将数组转换为对象形式（用于保持其他功能兼容）
const HUNAN_CITIES = cityCoordinates.reduce((acc, [name, lng, lat, code]) => {
  acc[name] = {
    center: [lng, lat],
    code: code
  };
  return acc;
}, {});

export default {
  name: 'HunanMap',
  setup() {
    const mapContainer = ref(null);
    let chart = null;
    const loading = ref(true);
    const error = ref('');
    const currentArea = ref('湖南省');
    
    // 默认视角参数
    const DEFAULT_CAMERA = {
      distance: 160,
      alpha: 70,
      beta: 0,
      center: [0.735583, 23.998643, 11.526896]
    };
    
    // 修改默认视角参数
    const cameraDistance = ref(DEFAULT_CAMERA.distance);
    const cameraAlpha = ref(DEFAULT_CAMERA.alpha);
    const cameraBeta = ref(DEFAULT_CAMERA.beta);
    const cameraCenter = ref([...DEFAULT_CAMERA.center]);

    // 修改生成随机数据的函数
    const generateRandomData = (mapData, isProvince = true) => {
      if (isProvince) {
        return cityCoordinates.map(([name, lng, lat]) => ({
          name,
          value: [lng, lat, Math.random() * 100]
        }));
      } else {
        // 处理市州级别的数据
        if (!mapData || !mapData.features) return [];
        return mapData.features.map(feature => ({
          name: feature.properties.name,
          value: [
            feature.properties.center[0],
            feature.properties.center[1],
            Math.random() * 100
          ]
        }));
      }
    };

    // 更新摄像头参数
    const updateCameraParams = () => {
      if (!chart) return;
      try {
        const option = chart.getOption();
        if (option?.geo3D?.[0]?.viewControl) {
          const viewControl = option.geo3D[0].viewControl;
          cameraDistance.value = Math.round(viewControl.distance);
          cameraAlpha.value = Math.round(viewControl.alpha);
          cameraBeta.value = Math.round(viewControl.beta % 360);
          if (viewControl.center) {
            cameraCenter.value = viewControl.center.map(v => Number(v.toFixed(6)));
          }
        }
      } catch (error) {
        console.warn('更新相机参数失败:', error);
      }
    };

    // 添加实时监听函数
    const addRealTimeListener = () => {
      if (!chart) return;
      
      // 监听鼠标事件
      ['mousedown', 'mouseup', 'mousemove', 'mousewheel'].forEach(eventName => {
        chart.getZr().on(eventName, () => {
          requestAnimationFrame(updateCameraParams);
        });
      });
      
      // 监听触摸事件
      ['touchstart', 'touchmove', 'touchend'].forEach(eventName => {
        chart.getZr().on(eventName, () => {
          requestAnimationFrame(updateCameraParams);
        });
      });
    };

    // 返回湖南省地图
    const backToProvince = async () => {
      currentArea.value = '湖南省';
      try {
        // 使用本地地图数据
        await initChart('湖南', hunanMapData);
      } catch (err) {
        error.value = err.message;
        console.error(err);
      }
    };

    // 复位视角
    const resetCamera = () => {
      if (!chart) return;
      
      const option = {
        geo3D: {
          viewControl: {
            projection: 'perspective',
            alpha: DEFAULT_CAMERA.alpha,
            beta: DEFAULT_CAMERA.beta,
            distance: DEFAULT_CAMERA.distance,
            minDistance: 40,
            maxDistance: 200,
            minAlpha: 10,
            maxAlpha: 90,
            minBeta: -180,
            maxBeta: 180,
            animation: true,
            animationDurationUpdate: 1000,
            orthographicSize: 100,
            center: DEFAULT_CAMERA.center
          }
        }
      };
      
      chart.setOption(option);
      
      // 更新显示的参数
      cameraDistance.value = DEFAULT_CAMERA.distance;
      cameraAlpha.value = DEFAULT_CAMERA.alpha;
      cameraBeta.value = DEFAULT_CAMERA.beta;
      cameraCenter.value = [...DEFAULT_CAMERA.center];
    };

    // 计算地图边界和尺寸比例
    const calculateMapBounds = (mapData) => {
      if (!mapData || !mapData.features || !mapData.features.length) {
        return { width: 100, depth: 100 }; // 默认值
      }

      let minX = Infinity;
      let maxX = -Infinity;
      let minY = Infinity;
      let maxY = -Infinity;

      // 遍历所有坐标点找出边界
      mapData.features.forEach(feature => {
        if (feature.geometry && feature.geometry.coordinates) {
          const coordinates = feature.geometry.coordinates;
          const processCoord = (coord) => {
            minX = Math.min(minX, coord[0]);
            maxX = Math.max(maxX, coord[0]);
            minY = Math.min(minY, coord[1]);
            maxY = Math.max(maxY, coord[1]);
          };

          // 处理多边形数据
          const traverse = (coords) => {
            if (typeof coords[0] === 'number') {
              processCoord(coords);
            } else {
              coords.forEach(c => traverse(c));
            }
          };

          traverse(coordinates);
        }
      });

      // 计算宽度和深度
      const width = maxX - minX;
      const depth = maxY - minY;
      
      // 根据实际比例计算容器尺寸
      const maxSize = 120; // 最大容器尺寸
      const ratio = width / depth;
      
      if (ratio > 1) {
        // 宽度大于深度
        return {
          width: maxSize,
          depth: Math.round(maxSize / ratio)
        };
      } else {
        // 深度大于宽度
        return {
          width: Math.round(maxSize * ratio),
          depth: maxSize
        };
      }
    };

    // 初始化地图
    const initChart = async (mapName = '湖南', mapData = null) => {
      try {
        loading.value = true;
        error.value = '';
        
        await nextTick();
        
        if (!mapContainer.value) {
          throw new Error('地图容器未找到');
        }
        
        if (chart) {
          chart.dispose();
        }
        
        chart = echarts.init(mapContainer.value);
        
        // 注册地图数据
        if (mapData) {
          echarts.registerMap(mapName, mapData);
        }
        
        const isProvince = mapName === '湖南';
        // 计算地图尺寸
        const mapSize = calculateMapBounds(mapData);
        
        const option = {
          backgroundColor: '#012248',
          tooltip: {
            show: true,
            formatter: function(params) {
              return `${params.name}<br/>
                      坐标: [${params.value[0].toFixed(4)}, ${params.value[1].toFixed(4)}]<br/>
                      数值: ${params.value[2].toFixed(2)}`;
            }
          },
          geo3D: {
            map: mapName,
            shading: 'realistic',
            realisticMaterial: {
              detailTexture: HNImg,  // 使用地形纹理
              textureTiling: 1  // 纹理平铺次数 [x, y
            },
            environment: '#012248',
            boxWidth: mapSize.width,
            boxHeight: 8,  // 统一高度为8
            boxDepth: mapSize.depth,
            regionHeight: 6,  // 统一区域高度为6
            groundPlane: {
              show: false
            },
            light: {
              main: {
                intensity: 1,
                alpha: 60,
                beta: 180
              },
              ambient: {
                intensity: 0.5
              }
            },
            viewControl: {
              projection: 'perspective',
              alpha: DEFAULT_CAMERA.alpha,
              beta: DEFAULT_CAMERA.beta,
              distance: DEFAULT_CAMERA.distance,
              minDistance: 40,
              maxDistance: 200,
              minAlpha: 10,
              maxAlpha: 90,
              minBeta: -180,
              maxBeta: 180,
              animation: true,
              animationDurationUpdate: 1000,
              orthographicSize: 100,
              center: DEFAULT_CAMERA.center
            },
            itemStyle: {
              color: '#00BBFF',
              opacity: 0.8,
              borderWidth: 1,
              borderColor: '#fff'
            },
            emphasis: {
              itemStyle: {
                color: '#00EAFF',
                opacity: 1
              }
            },
            regions: isProvince 
              ? Object.keys(HUNAN_CITIES).map(name => ({
                  name,
                  itemStyle: {
                    color: '#4169E1',
                    opacity: 0.8
                  }
                })) 
              : mapData.features.map(feature => ({
                  name: feature.properties.name,
                  itemStyle: {
                    color: '#4169E1',
                    opacity: 0.8
                  }
                }))
          },
          series: [{
            type: 'bar3D',
            coordinateSystem: 'geo3D',
            data: generateRandomData(mapData, isProvince),
            barSize: 2,
            minHeight: 10,
            silent: false,
            itemStyle: {
              color: '#FFD700',
              opacity: 0.6,
            },
            emphasis: {
              itemStyle: {
                color: '#00FF00',
                opacity: 0.8
              }
            },
            label: {
              show: true,
              formatter: function(params) {
                return Math.round(params.value[2]);  // 只显示数值
              },
              position: 'top',
              distance: 2,
              textStyle: {
                fontSize: 20,
                fontWeight: '600',
                color: '#000000',
                fontFamily: 'TCloud, Arial, sans-serif',
                padding: [2, 4],
                textBorderColor: '#ffffff',
                textBorderWidth: 3,
                opacity: 1,
                align: 'center'
              }
            }
          }, {
            type: 'scatter3D',
            coordinateSystem: 'geo3D',
            data: generateRandomData(mapData, isProvince).map(item => ({
              name: item.name,
              value: [item.value[0], item.value[1], 0]
            })),
            symbolSize: 0,
            label: {
              show: true,
              formatter: function(params) {
                return params.name;
              },
              position: 'left',
              distance: 10,
              textStyle: {
                fontSize: 12,
                fontWeight: 'bold',
                color: '#ffffff',
                padding: [2, 4],
                textBorderColor: '#000000',
                textBorderWidth: 3,
                opacity: 0.8,
                align: 'center'
              }
            }
          }]
        };
        
        chart.setOption(option);
        loading.value = false;
        
        // 添加实时监听
        addRealTimeListener();
        
        // 添加点击事件
        chart.on('click', async params => {
          if (currentArea.value === '湖南省' && params.name in HUNAN_CITIES) {
            try {
              const cityCode = HUNAN_CITIES[params.name].code;
              // 加载本地市州地图数据
              try {
                const cityMapData = await import(`../map/${cityCode}.json`);
                currentArea.value = params.name;
                await initChart(params.name, cityMapData.default);
              } catch (importError) {
                console.error(`未找到${params.name}地图数据文件: ../map/${cityCode}.json`);
                error.value = `未找到${params.name}地图数据文件`;
              }
            } catch (err) {
              error.value = err.message;
              console.error(err);
            }
          }
        });
        
        // 添加视角变化事件监听
        chart.on('georoam', () => {
          requestAnimationFrame(updateCameraParams);
        });
        
      } catch (err) {
        console.error('初始化错误:', err);
        error.value = `初始化失败: ${err.message}`;
        loading.value = false;
      }
    };

    onMounted(async () => {
      // 初始加载湖南省地图
      await backToProvince();
      
      // 监听窗口大小变化
      const handleResize = () => {
        if (chart) {
          chart.resize();
          requestAnimationFrame(updateCameraParams);
        }
      };
      
      window.addEventListener('resize', handleResize);
      
      onUnmounted(() => {
        // 清理事件监听
        if (chart) {
          const zr = chart.getZr();
          ['mousedown', 'mouseup', 'mousemove', 'mousewheel', 'touchstart', 'touchmove', 'touchend'].forEach(eventName => {
            zr.off(eventName);
          });
          window.removeEventListener('resize', handleResize);
          chart.dispose();
          chart = null;
        }
      });
    });

    return {
      mapContainer,
      currentArea,
      cameraDistance,
      cameraAlpha,
      cameraBeta,
      cameraCenter,
      loading,
      error,
      backToProvince,
      resetCamera
    };
  }
};
</script>

<style scoped>
.container {
  position: relative;
  width: 100%;
  height: 960px;
}

.map-container {
  width: 100%;
  height: 100%;
}

.camera-info {
  position: absolute;
  top: 20px;
  left: 20px;
  background: rgba(1, 34, 72, 0.8);
  color: white;
  padding: 15px;
  border-radius: 5px;
  z-index: 1000;
}

.camera-info h3 {
  margin: 0 0 10px 0;
  font-size: 16px;
}

.camera-info p {
  margin: 5px 0;
  font-size: 14px;
}

.camera-info button {
  margin: 10px 0;
  padding: 5px 10px;
  background: #1E90FF;
  border: none;
  border-radius: 3px;
  color: white;
  cursor: pointer;
}

.camera-info button:hover {
  background: #4169E1;
}

.loading, .error {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 20px;
  background: rgba(1, 34, 72, 0.8);
  color: white;
  border-radius: 5px;
  z-index: 1000;
}

.error {
  color: #ff6b6b;
}

.button-group {
  display: flex;
  gap: 10px;
  margin: 10px 0;
}

.button-group button {
  padding: 5px 10px;
  background: #1E90FF;
  border: none;
  border-radius: 3px;
  color: white;
  cursor: pointer;
  flex: 1;
}

.button-group button:hover {
  background: #4169E1;
}
</style> 