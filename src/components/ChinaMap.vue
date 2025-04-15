<template>
  <div class="container">
    <div v-if="loading" class="loading">加载中...</div>
    <div v-if="error" class="error">{{ error }}</div>
    <div ref="mapContainer" class="map-container"></div>
    <div v-if="!loading && !error" class="camera-info">
      <h3>当前区域: {{ currentArea }}</h3>
      <div class="button-group">
        <button v-if="currentArea !== '中国'" @click="backToChina">返回全国</button>
        <button v-if="currentArea !== '湖南省' && currentArea !== '中国'" @click="backToProvince">返回湖南省</button>
        <button @click="resetCamera">复位视角</button>
        <button @click="toggleProjection">
          切换{{ projectionMode === 'perspective' ? '正交' : '透视' }}视图
        </button>
      </div>
      <div class="camera-params">
        <template v-if="projectionMode === 'perspective'">
          <p>视图模式: 透视</p>
          <p>距离: {{ cameraDistance }}</p>
        </template>
        <template v-else>
          <p>视图模式: 正交</p>
          <p>视野大小: {{ orthographicSize }}</p>
        </template>
        <p>俯仰角: {{ cameraAlpha }}°</p>
        <p>方位角: {{ cameraBeta }}°</p>
        <p>中心坐标: [{{ cameraCenter.join(', ') }}]</p>
      </div>
    </div>
    <div class="performance-panel">
      <div class="panel-header">
        <h3>性能监控</h3>
        <button @click="clearPerformanceHistory">清除</button>
      </div>
      <div class="metrics-list">
        <div v-for="(record, index) in performanceHistory" :key="index" class="metric-item">
          <span>{{ record.operation }}</span>
          <span>{{ record.duration }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as echarts from 'echarts';
import 'echarts-gl';
import { onMounted, onUnmounted, ref, nextTick, watch } from 'vue';
import hunanMapData from '../map/430000.json';
import chinaMapData from '../map/100000.json';
import HNImg from '../assets/textures/fn.png';
import CNImg from '../assets/textures/zg.png';

// 使用数组形式存储湖南市州坐标数据
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

// 使用数组形式存储全国所有省份、自治区、直辖市坐标数据
const provinceCoordinates = [
  // 华北地区
  ['北京市', 116.405285, 39.904989, '110000'],
  ['天津市', 117.190182, 39.125596, '120000'],
  ['河北省', 114.502461, 38.045474, '130000'],
  ['山西省', 112.549248, 37.857014, '140000'],
  ['内蒙古自治区', 111.670801, 40.818311, '150000'],

  // 东北地区
  ['辽宁省', 123.429096, 41.796767, '210000'],
  ['吉林省', 125.324501, 43.886841, '220000'],
  ['黑龙江省', 126.642464, 45.756967, '230000'],

  // 华东地区
  ['上海市', 121.472644, 31.231706, '310000'],
  ['江苏省', 118.767413, 32.041544, '320000'],
  ['浙江省', 120.153576, 30.287459, '330000'],
  ['安徽省', 117.283042, 31.86119, '340000'],
  ['福建省', 119.306239, 26.075302, '350000'],
  ['江西省', 115.892151, 28.676493, '360000'],
  ['山东省', 117.000923, 36.675807, '370000'],

  // 华中地区
  ['河南省', 113.665412, 34.757975, '410000'],
  ['湖北省', 114.298572, 30.584355, '420000'],
  ['湖南省', 112.982279, 28.19409, '430000'],

  // 华南地区
  ['广东省', 113.280637, 23.125178, '440000'],
  ['广西壮族自治区', 108.320004, 22.82402, '450000'],
  ['海南省', 110.33119, 20.031971, '460000'],

  // 西南地区
  ['重庆市', 106.504962, 29.533155, '500000'],
  ['四川省', 104.065735, 30.659462, '510000'],
  ['贵州省', 106.713478, 26.578343, '520000'],
  ['云南省', 102.712251, 25.040609, '530000'],
  ['西藏自治区', 91.132212, 29.660361, '540000'],

  // 西北地区
  ['陕西省', 108.948024, 34.263161, '610000'],
  ['甘肃省', 103.823557, 36.058039, '620000'],
  ['青海省', 101.778916, 36.623178, '630000'],
  ['宁夏回族自治区', 106.278179, 38.46637, '640000'],
  ['新疆维吾尔自治区', 87.617733, 43.792818, '650000'],

  // 特别行政区
  ['香港特别行政区', 114.173355, 22.320048, '810000'],
  ['澳门特别行政区', 113.54909, 22.198951, '820000'],
  ['台湾省', 121.509062, 25.044332, '710000']
];

// 修改默认视角参数配置
const DEFAULT_CAMERA = {
  china: {
    perspective: {
      distance: 200,
      alpha: 70,
      beta: 0,
      center: [2.786219, 29.710574, 6.20325]
    },
    orthographic: {
      orthographicSize: 220, 
      alpha: 80,
      beta: 0,
      center: [2.786219, 29.710574, 6.20325]
    }
  },
  province: {
    perspective: {
      distance: 160,
      alpha: 70,
      beta: 0,
      center: [0.735583, 23.998643, 11.526896]
    },
    orthographic: {
      orthographicSize: 80,   // 省级地图下的默认视野大小
      alpha: 70,
      beta: 0,
      center: [0.735583, 23.998643, 11.526896]
    }
  }
};

export default {
  name: 'ChinaMap',
  setup() {
    const mapContainer = ref(null);
    let chart = null;
    const loading = ref(true);
    const error = ref('');
    const currentArea = ref('中国');
    
    // 地图样式配置
    const mapConfig = {
      boxHeight: 8,
      regionHeight: 6,
      itemStyle: {
        color: '#fff',
        opacity: 0.8,
        borderWidth: 1,
        borderColor: '#fff'
      }
    };

    // 柱状图配置
    const barConfig = {
      barSize: 2,
      minHeight: 10,
      itemStyle: {
        color: '#FFD700',
        opacity: 0.6,
        bevelSize: 1,           // 添加斜角大小
        bevelSmoothness: 2      // 添加斜角平滑度
      }
    };
    
    const cameraDistance = ref(DEFAULT_CAMERA.china.perspective.distance);
    const cameraAlpha = ref(DEFAULT_CAMERA.china.perspective.alpha);
    const cameraBeta = ref(DEFAULT_CAMERA.china.perspective.beta);
    const cameraCenter = ref([...DEFAULT_CAMERA.china.perspective.center]);

    // 添加投影模式的响应式变量
    const projectionMode = ref('perspective'); // 默认透视投影

    // 添加正交大小的响应式变量
    const orthographicSize = ref(DEFAULT_CAMERA.china.orthographic.orthographicSize);

    // 将 HUNAN_CITIES 定义移到这里
    const HUNAN_CITIES = ref(cityCoordinates.reduce((acc, [name, lng, lat, code]) => {
      acc[name] = {
        center: [lng, lat],
        code: code
      };
      return acc;
    }, {}));

    // 添加性能监控相关的响应式数据
    const performanceHistory = ref([]);
    
    // 性能监控工具
    const performanceMonitor = {
      metrics: new Map(),
      
      start(operation) {
        this.metrics.set(operation, performance.now());
      },
      
      end(operation) {
        const startTime = this.metrics.get(operation);
        if (startTime) {
          const duration = performance.now() - startTime;
          performanceHistory.value.push({
            operation,
            duration: `${duration.toFixed(2)}ms`,
            timestamp: new Date().toISOString()
          });
          console.log(`${operation}: ${duration.toFixed(2)}ms`);
          this.metrics.delete(operation);
        }
      }
    };
    
    // 清除性能历史记录
    const clearPerformanceHistory = () => {
      performanceHistory.value = [];
    };

    // 生成随机数据函数修改
    const generateRandomData = (mapData, level = 'country') => {
      if (level === 'country') {
        return provinceCoordinates.map(([name, lng, lat]) => ({
          name,
          value: [lng, lat, Math.random() * 100]
        }));
      } else if (level === 'province') {
        return cityCoordinates.map(([name, lng, lat]) => ({
          name,
          value: [lng, lat, Math.random() * 100]
        }));
      } else {
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

    // 修改更新摄像头参数的函数
    const updateCameraParams = () => {
      if (!chart) return;
      try {
        const option = chart.getOption();
        if (option?.geo3D?.[0]?.viewControl) {
          const viewControl = option.geo3D[0].viewControl;
          
          // 更新投影模式
          projectionMode.value = viewControl.projection;
          
          // 更新相机参数
          if (viewControl.projection === 'perspective') {
            // 透视模式下更新距离
            cameraDistance.value = Math.round(viewControl.distance);
          }
          
          // 更新共同参数
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

    // 修改 addRealTimeListener 函数
    const addRealTimeListener = () => {
      if (!chart) return;
      
      // 在正交模式下跟踪缩放比例
      let scaleRatio = 1;
      
      // 监听鼠标滚轮事件
      chart.getZr().on('mousewheel', (e) => {
        if (projectionMode.value === 'orthographic') {
          // 根据滚轮方向调整缩放比例
          const delta = e.wheelDelta > 0 ? 0.9 : 1.1;
          scaleRatio *= delta;
          
          // 计算新的视野大小
          const baseSize = DEFAULT_CAMERA.china.orthographic.orthographicSize;
          const newSize = Math.round(baseSize * scaleRatio);
          
          // 限制范围
          orthographicSize.value = Math.max(10, Math.min(300, newSize));
        }
      });
      
      // 监听所有可能导致视图变化的事件
      ['mousedown', 'mouseup', 'mousemove', 'mousewheel'].forEach(eventName => {
        chart.getZr().on(eventName, () => {
          requestAnimationFrame(() => {
            const option = chart.getOption();
            if (option?.geo3D?.[0]?.viewControl) {
              const viewControl = option.geo3D[0].viewControl;
              
              // 更新共同参数
              cameraAlpha.value = Math.round(viewControl.alpha);
              cameraBeta.value = Math.round(viewControl.beta % 360);
              if (viewControl.center) {
                cameraCenter.value = viewControl.center.map(v => Number(v.toFixed(6)));
              }
              
              // 更新特定模式的参数
              if (viewControl.projection === 'perspective') {
                cameraDistance.value = Math.round(viewControl.distance);
              }
            }
          });
        });
      });
      
      // 监听触摸事件
      ['touchstart', 'touchmove', 'touchend'].forEach(eventName => {
        chart.getZr().on(eventName, () => {
          requestAnimationFrame(() => {
            const option = chart.getOption();
            if (option?.geo3D?.[0]?.viewControl) {
              const viewControl = option.geo3D[0].viewControl;
              
              // 更新共同参数
              cameraAlpha.value = Math.round(viewControl.alpha);
              cameraBeta.value = Math.round(viewControl.beta % 360);
              if (viewControl.center) {
                cameraCenter.value = viewControl.center.map(v => Number(v.toFixed(6)));
              }
              
              // 更新特定模式的参数
              if (viewControl.projection === 'perspective') {
                cameraDistance.value = Math.round(viewControl.distance);
              }
            }
          });
        });
      });
      
      // 监听视图变化
      chart.on('georoam', () => {
        requestAnimationFrame(() => {
          const option = chart.getOption();
          if (option?.geo3D?.[0]?.viewControl) {
            const viewControl = option.geo3D[0].viewControl;
            
            // 更新共同参数
            cameraAlpha.value = Math.round(viewControl.alpha);
            cameraBeta.value = Math.round(viewControl.beta % 360);
            if (viewControl.center) {
              cameraCenter.value = viewControl.center.map(v => Number(v.toFixed(6)));
            }
            
            // 更新特定模式的参数
            if (viewControl.projection === 'perspective') {
              cameraDistance.value = Math.round(viewControl.distance);
            }
          }
        });
      });
      
      // 重置缩放比例的函数
      const resetScale = () => {
        scaleRatio = 1;
        if (projectionMode.value === 'orthographic') {
          orthographicSize.value = DEFAULT_CAMERA.china.orthographic.orthographicSize;
        }
      };
      
      // 切换投影模式时重置缩放
      watch(projectionMode, () => {
        resetScale();
      });
      
      // 切换区域时重置缩放
      watch(currentArea, () => {
        resetScale();
      });
    };

    // 返回全国地图
    const backToChina = async () => {
      currentArea.value = '中国';
      try {
        await initChart('中国', chinaMapData, 'country');
      } catch (err) {
        error.value = err.message;
        console.error(err);
      }
    };

    // 返回湖南省地图
    const backToProvince = async () => {
      currentArea.value = '湖南省';
      try {
        await initChart('湖南', hunanMapData, 'province');
      } catch (err) {
        error.value = err.message;
        console.error(err);
      }
    };

    // 修改 getViewControlConfig 函数
    const getViewControlConfig = (level) => {
      const isCountry = level === 'country';
      const defaultConfig = isCountry 
        ? DEFAULT_CAMERA.china[projectionMode.value]
        : DEFAULT_CAMERA.province[projectionMode.value];
      
      return {
        projection: projectionMode.value,
        alpha: cameraAlpha.value || defaultConfig.alpha,
        beta: cameraBeta.value || defaultConfig.beta,
        center: cameraCenter.value || defaultConfig.center,
        minAlpha: 10,
        maxAlpha: 90,
        minBeta: -180,
        maxBeta: 180,
        animation: false,
        roam: true,
        zoomSensitivity: 1.5,
        ...(projectionMode.value === 'orthographic' ? {
          orthographicSize: orthographicSize.value,
          minOrthographicSize: 10,
          maxOrthographicSize: isCountry ? 300 : 150,
          zoomToMouseLocation: true
        } : {
          distance: cameraDistance.value || defaultConfig.distance,
          minDistance: 40,
          maxDistance: isCountry ? 400 : 200
        })
      };
    };

    // 修改 resetCamera 函数
    const resetCamera = () => {
      if (!chart) return;
      
      const isCountry = currentArea.value === '中国';
      const defaultConfig = isCountry 
        ? DEFAULT_CAMERA.china[projectionMode.value]
        : DEFAULT_CAMERA.province[projectionMode.value];
      
      // 计算地图尺寸
      const mapSize = calculateMapBounds(isCountry ? chinaMapData : hunanMapData);
      
      // 构建视图控制参数
      const viewControl = {
        projection: projectionMode.value,
        alpha: defaultConfig.alpha,
        beta: defaultConfig.beta,
        center: defaultConfig.center,
        animation: true,
        animationDurationUpdate: 500  // 缩短复位动画时间
      };
      
      // 根据模式添加特定参数
      if (projectionMode.value === 'perspective') {
        Object.assign(viewControl, {
          distance: defaultConfig.distance,
          minDistance: 40,
          maxDistance: isCountry ? 400 : 200
        });
      } else {
        Object.assign(viewControl, {
          orthographicSize: defaultConfig.orthographicSize,
          minOrthographicSize: 10,
          maxOrthographicSize: isCountry ? 300 : 150
        });
      }
      
      // 设置完整配置
      chart.setOption({
        geo3D: {
          boxWidth: isCountry ? 200 : mapSize.width,
          boxHeight: mapConfig.boxHeight,
          boxDepth: isCountry ? 160 : mapSize.depth,
          viewControl
        }
      });
      
      // 更新响应式数据
      if (projectionMode.value === 'perspective') {
        cameraDistance.value = defaultConfig.distance;
      } else {
        orthographicSize.value = defaultConfig.orthographicSize;
      }
      cameraAlpha.value = defaultConfig.alpha;
      cameraBeta.value = defaultConfig.beta;
      cameraCenter.value = [...defaultConfig.center];
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

    // 修改 initChart 函数
    const initChart = async (mapName, mapData, level = 'country') => {
      performanceMonitor.start('初始化图表');
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
        
        performanceMonitor.start('创建图表实例');
        chart = echarts.init(mapContainer.value);
        performanceMonitor.end('创建图表实例');
        
        if (mapData) {
          performanceMonitor.start('注册地图数据');
          echarts.registerMap(mapName, mapData);
          performanceMonitor.end('注册地图数据');
        }
        
        performanceMonitor.start('计算地图边界');
        const mapSize = calculateMapBounds(mapData);
        performanceMonitor.end('计算地图边界');
        
        // 获取并设置默认相机参数
        const defaultConfig = level === 'country' ? DEFAULT_CAMERA.china.perspective : DEFAULT_CAMERA.province.perspective;
        cameraDistance.value = defaultConfig.distance;
        cameraAlpha.value = defaultConfig.alpha;
        cameraBeta.value = defaultConfig.beta;
        cameraCenter.value = [...defaultConfig.center];
        
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
              detailTexture: level === 'country' ? CNImg : HNImg,
              textureTiling: 1
            },
            environment: '#012248',
            boxWidth: level === 'country' ? 200 : mapSize.width,
            boxHeight: mapConfig.boxHeight,
            boxDepth: level === 'country' ? 160 : mapSize.depth,
            regionHeight: mapConfig.regionHeight,
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
            viewControl: getViewControlConfig(level),
            itemStyle: mapConfig.itemStyle,
            emphasis: {
              itemStyle: {
                color: '#00EAFF',
                opacity: 1
              }
            }
          },
          series: [{
            type: 'bar3D',
            coordinateSystem: 'geo3D',
            data: generateRandomData(mapData, level),
            barSize: level === 'country' ? barConfig.barSize * 1.2 : barConfig.barSize,
            minHeight: barConfig.minHeight,
            silent: false,
            shading: 'realistic',  // 使用真实感渲染
            itemStyle: {
              ...barConfig.itemStyle,
              opacity: 0.6
            },
            shape: 'cylinder',  // 设置为圆柱体形状
            emphasis: {
              itemStyle: {
                color: '#00FF00',
                opacity: 0.8
              }
            },
            label: {
              show: true,
              formatter: function(params) {
                return Math.round(params.value[2]);
              },
              position: 'top',
              distance: 2,
              textStyle: {
                fontSize: level === 'country' ? 16 : 20,
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
            data: generateRandomData(mapData, level).map(item => ({
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
                fontSize: level === 'country' ? 14 : 12,
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
        
        performanceMonitor.start('设置图表选项');
        chart.setOption(option);
        performanceMonitor.end('设置图表选项');
        
        loading.value = false;
        addRealTimeListener();
        
        performanceMonitor.end('初始化图表');

        chart.on('click', async params => {
          if (currentArea.value === '中国') {
            // 检查是否点击了省份
            const clickedProvince = provinceCoordinates.find(([name]) => name === params.name);
            if (clickedProvince) {
              if (params.name === '湖南省') {
                await backToProvince();
              } else {
                error.value = '目前只支持查看湖南省的详细信息';
              }
            }
          } else if (currentArea.value === '湖南省' && params.name in HUNAN_CITIES.value) {
            try {
              const cityCode = HUNAN_CITIES.value[params.name].code;
              try {
                const cityMapData = await import(`../map/${cityCode}.json`);
                currentArea.value = params.name;
                await initChart(params.name, cityMapData.default, 'city');
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
      } catch (err) {
        performanceMonitor.end('初始化图表');
        console.error('初始化错误:', err);
        error.value = `初始化失败: ${err.message}`;
        loading.value = false;
      }
    };

    // 修改切换投影模式的函数
    const toggleProjection = () => {
      if (!chart) return;
      
      const newMode = projectionMode.value === 'perspective' ? 'orthographic' : 'perspective';
      const isCountry = currentArea.value === '中国';
      const defaultConfig = isCountry 
        ? DEFAULT_CAMERA.china[newMode]
        : DEFAULT_CAMERA.province[newMode];
      
      // 计算地图尺寸
      const mapSize = calculateMapBounds(isCountry ? chinaMapData : hunanMapData);
      
      // 1. 先只切换投影模式，不带动画
      chart.setOption({
        geo3D: {
          viewControl: {
            projection: newMode,
            animation: false
          }
        }
      });
      
      // 2. 立即设置完整参数
      const viewControl = {
        projection: newMode,
        alpha: defaultConfig.alpha,
        beta: defaultConfig.beta,
        center: defaultConfig.center,
        animation: true,
        animationDurationUpdate: 1000
      };
      
      // 根据模式添加特定参数
      if (newMode === 'perspective') {
        Object.assign(viewControl, {
          distance: defaultConfig.distance,
          minDistance: 40,
          maxDistance: isCountry ? 400 : 200
        });
      } else {
        Object.assign(viewControl, {
          orthographicSize: defaultConfig.orthographicSize,
          minOrthographicSize: 10,
          maxOrthographicSize: isCountry ? 300 : 150
        });
      }
      
      // 设置完整配置
      chart.setOption({
        geo3D: {
          boxWidth: isCountry ? 200 : mapSize.width,
          boxHeight: mapConfig.boxHeight,
          boxDepth: isCountry ? 160 : mapSize.depth,
          viewControl
        }
      });
      
      // 3. 更新投影模式
      projectionMode.value = newMode;
      
      // 4. 延迟执行复位
      setTimeout(() => {
        resetCamera();
      }, 1000);
    };

    // 添加 watch 以监控视野大小变化
    watch(orthographicSize, (newValue) => {
      console.log('orthographicSize changed:', newValue);
    });

    onMounted(async () => {
      await backToChina();
      
      const handleResize = () => {
        if (chart) {
          chart.resize();
          requestAnimationFrame(updateCameraParams);
        }
      };
      
      window.addEventListener('resize', handleResize);
      
      onUnmounted(() => {
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
      backToChina,
      backToProvince,
      resetCamera,
      projectionMode,
      toggleProjection,
      orthographicSize,
      HUNAN_CITIES,
      performanceHistory,
      clearPerformanceHistory
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

.camera-params {
  margin-top: 10px;
}

.performance-panel {
  position: fixed;
  right: 20px;
  bottom: 20px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 15px;
  border-radius: 5px;
  z-index: 1000;
  min-width: 300px;
  max-height: 400px;
  overflow-y: auto;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
  padding-bottom: 10px;
}

.panel-header h3 {
  margin: 0;
  font-size: 16px;
}

.panel-header button {
  padding: 4px 8px;
  background: #1E90FF;
  border: none;
  border-radius: 3px;
  color: white;
  cursor: pointer;
}

.panel-header button:hover {
  background: #4169E1;
}

.metrics-list {
  font-size: 14px;
}

.metric-item {
  display: flex;
  justify-content: space-between;
  padding: 5px 0;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.metric-item:last-child {
  border-bottom: none;
}
</style> 