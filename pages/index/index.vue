
<template>
  <div class="container">
    <div ref="threeContainer" class="three-container"></div>
    <div class="controls">
      <!-- 层数选择 -->
      <div class="layer-controls">
        <label>当前选择的层数: {{ selectedLayer + 1 }}</label>
        <select v-model="selectedLayer">
          <option v-for="index in layers" :key="index" :value="index - 1">第 {{ index }} 层</option>
        </select>
      </div>

      <!-- 改变层颜色按钮 -->
      <button @click="changeSelectedLayerColor('#FF69B4')">粉红色</button>
      <button @click="changeSelectedLayerColor('#8A2BE2')">蓝紫色</button>

      <!-- 装饰物列表 -->
      <div class="decorations-list">
        <h3>装饰物列表</h3>
        <div class="decorations-container">
          <div v-for="decoration in decorations" :key="decoration.id" class="decoration-item">
            <img :src="decoration.thumb_url" alt="Decoration thumbnail" />
            <p>{{ decoration.name }}</p>
            <button @click="addDecoration(decoration.file_url)">添加装饰</button>
          </div>
        </div>
      </div>

      <!-- 增减层数按钮 -->
      <div class="layer-controls">
        <button @click="increaseLayers">增加层数</button>
        <button @click="decreaseLayers" :disabled="layers <= 1">减少层数</button>
        <label>当前层数: {{ layers }}</label>
      </div>

      <!-- 删除选中装饰物按钮 -->
      <button @click="deleteSelectedDecoration" :disabled="!selectedDecoration">删除选中装饰</button>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';
import axios from 'axios';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';

export default {
  setup() {
    // 状态变量
    const layers = ref(1); // 默认层数为1
    const selectedLayer = ref(0); // 默认选择第0层
    const threeContainer = ref(null); // 引用 Three.js 容器
    const decorations = ref([]); // 装饰物列表
    const selectedDecoration = ref(null); // 当前选中的装饰物

    // Three.js 相关变量
    let scene, camera, renderer, cake, controls;
    let cakeLayers = []; // 用于保存蛋糕层
    let loadedDecorations = []; // 用于保存加载到场景中的装饰物

    const raycaster = new THREE.Raycaster(); // 射线检测
    const mouse = new THREE.Vector2(); // 鼠标坐标
    let isDragging = false; // 是否正在拖动装饰物
 
 watch(layers, (newLayers, oldLayers) => {
   if (selectedLayer.value >= newLayers) {
     selectedLayer.value = newLayers - 1;
   }
 });

    // 初始化 Three.js 场景
    const initThreeJS = () => {
      if (!threeContainer.value) {
        console.error("Three.js 容器未找到");
        return;
      }

      const width = threeContainer.value.clientWidth;
      const height = threeContainer.value.clientHeight;

      // 创建场景
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0xffffff);

      // 创建相机
      camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);
      camera.position.set(0, 2, 5);

      // 创建渲染器，移动端禁用抗锯齿以优化性能
      const isMobile = /Mobi|Android/i.test(navigator.userAgent);
      renderer = new THREE.WebGLRenderer({ antialias: !isMobile });
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(width, height);
      threeContainer.value.appendChild(renderer.domElement);

      // 添加光源
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
      scene.add(ambientLight);

      const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
      directionalLight.position.set(5, 10, 7.5);
      scene.add(directionalLight);

      // 添加托盘
      const plateGeometry = new THREE.BoxGeometry(4, 0.01, 4);
      const plateMaterial = new THREE.MeshStandardMaterial({ color: '#C0C0C0' });
      const plate = new THREE.Mesh(plateGeometry, plateMaterial);
      plate.position.y = -0.3;
      scene.add(plate);

      // 添加控制器
      controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;
      controls.dampingFactor = 0.1;
      controls.screenSpacePanning = false;

      // 鼠标和触摸事件监听
      initEventListeners();

      // 开始渲染
      animate();
    };

    // 创建蛋糕
    const createCake = () => {
      // 清空现有蛋糕层
      if (cake) {
        scene.remove(cake);
      }

      // 清空装饰物
      loadedDecorations.forEach(decoration => {
        scene.remove(decoration); // 从场景中移除装饰物
      });
      loadedDecorations = []; // 清空装饰物数组
      selectedDecoration.value = null; // 清空选中的装饰物

      cake = new THREE.Group();
      cakeLayers = [];

      // 创建新的蛋糕层
      for (let i = 0; i < layers.value; i++) {
        const baseRadius = 1.5; // 基础半径
        const height = 0.5; // 每层高度
        const radius = baseRadius * (1 - i * 0.1); // 每层半径逐渐减小
        const geometry = new THREE.CylinderGeometry(radius, radius, height, 32);
        const material = new THREE.MeshStandardMaterial({ color: '#FFCC00' });
        const layer = new THREE.Mesh(geometry, material);
        layer.position.y = i * height;
        cake.add(layer);
        cakeLayers.push(layer);
      }

      scene.add(cake);
      renderScene();
    };

    // 更改选中层的颜色
    const changeSelectedLayerColor = (color) => {
      const layerIndex = selectedLayer.value;
      const layer = cakeLayers[layerIndex];
      if (layer) {
        layer.material.color.set(color);
        renderScene();
      }
    };

    // 添加装饰物
    const addDecoration = async (fileUrl) => {
      // 只允许在最顶层添加装饰物
      if (selectedLayer.value !== layers.value - 1) {
        alert("只能在最顶层添加装饰物");
        return;
      }

      if (!fileUrl) {
        console.error("装饰物文件 URL 无效");
        alert("装饰物文件 URL 无效");
        return;
      }

      try {
        const decoration = await loadModel(fileUrl);

        // 计算当前选中层的 y 位置
        const layerHeight = 0.5; // 每层的高度
        const layerIndex = selectedLayer.value; // 当前选中的层数
        const decorationYOffset = 0.2; // 装饰物与层的偏移量

        // 获取装饰物的边界盒
        const box = new THREE.Box3().setFromObject(decoration);
        const decorationHeight = box.max.y - box.min.y; // 装饰物的高度
        const decorationWidth = box.max.x - box.min.x; // 装饰物的宽度

        // 计算层的半径
        const baseRadius = 1.5; // 蛋糕的基础半径
        const layerRadius = baseRadius * (1 - layerIndex * 0.1); // 当前层的半径

        // 计算缩放比例，增加一个缩放因子（如0.5）来缩小装饰物
        const scaleRatio = (layerRadius / (decorationWidth / 2)) * 0.2; // 等比例缩放并乘以缩放因子
        decoration.scale.set(scaleRatio, scaleRatio, scaleRatio); // 设置等比例缩放

        // 设置装饰物初始位置在蛋糕的顶部中心
        decoration.position.set(0, layerIndex * layerHeight + decorationYOffset, 0);

        scene.add(decoration);
        loadedDecorations.push(decoration);
        selectedDecoration.value = decoration;

        // 创建包围盒(检测是否重叠)
        decoration.boundingBox = new THREE.Box3().setFromObject(decoration);
        console.log("装饰物已添加:", decoration);
        renderScene();
      } catch (error) {
        console.error("添加装饰物时出错:", error);
        alert("添加装饰物时出错，请检查控制台以获取详细信息。");
      }
    };

    // 加载 GLTF 模型
    const loadModel = (url) => {
      return new Promise((resolve, reject) => {
        const loader = new GLTFLoader();
        loader.load(url, (gltf) => {
          const model = gltf.scene;
          resolve(model);
        }, undefined, (error) => {
          console.error('加载模型时出错:', error);
          reject(error);
        });
      });
    };

    // 获取装饰物列表
    const fetchDecorations = async () => {
      try {
        const response = await axios.get('https://develop-wxmp-api.blacktechcake.com/3dm/list?page=1&pageSize=20&type_id=13&user_id=100');
        console.log('API 响应:', response.data);

        // 确保 list 存在并且是一个数组
        if (response.data && response.data.data && Array.isArray(response.data.data.list)) {
          decorations.value = response.data.data.list;
        } else {
          console.warn('装饰列表为空或格式不正确:', response.data);
        }
      } catch (error) {
        console.error('获取装饰列表失败:', error);
      }
    };

    // 处理鼠标和触摸按下事件
    const onPointerDown = (event) => {
      event.preventDefault();
      const rect = threeContainer.value.getBoundingClientRect();

      // 获取鼠标或触摸的位置
      if (event.touches) {
        mouse.x = ((event.touches[0].clientX - rect.left) / rect.width) * 2 - 1;
        mouse.y = -((event.touches[0].clientY - rect.top) / rect.height) * 2 + 1;
      } else {
        mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
        mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
      }

      raycaster.setFromCamera(mouse, camera);
      const intersects = raycaster.intersectObjects(loadedDecorations, true);

      if (intersects.length > 0) {
        // 取消之前的选中状态
        if (selectedDecoration.value && selectedDecoration.value !== intersects[0].object.parent) {
          selectedDecoration.value.traverse((child) => {
            if (child.isMesh) {
              child.material.emissive.set(0x000000);
            }
          });
        }

        // 选中新的装饰物
        selectedDecoration.value = intersects[0].object.parent; // 确保选中整个模型
        selectedDecoration.value.traverse((child) => {
          if (child.isMesh) {
            child.material.emissive.set(0x444444);
          }
        });
        isDragging = true;
        controls.enabled = false; // 禁用 OrbitControls 以避免冲突
      } else {
        // 点击空白处取消选中
        if (selectedDecoration.value) {
          selectedDecoration.value.traverse((child) => {
            if (child.isMesh) {
              child.material.emissive.set(0x000000);
            }
          });
          selectedDecoration.value = null;
        }
      }
    };

    // 处理鼠标和触摸移动事件
    const onPointerMove = (event) => {
      if (!isDragging || !selectedDecoration.value) return;
      if (!threeContainer.value) return;

      const rect = threeContainer.value.getBoundingClientRect();

      // 获取鼠标或触摸的位置
      if (event.touches) {
        mouse.x = ((event.touches[0].clientX - rect.left) / rect.width) * 2 - 1;
        mouse.y = -((event.touches[0].clientY - rect.top) / rect.height) * 2 + 1;
      } else {
        mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
        mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
      }

      raycaster.setFromCamera(mouse, camera);
      const intersects = raycaster.intersectObject(cake, true);

      if (intersects.length > 0) {
        const intersection = intersects[0].point;
        const cakeRadius = 1.5; // 蛋糕的基础半径

        // 限制装饰物的 X 和 Z 轴位置，确保它们在蛋糕边界内
        const clampedX = Math.max(-cakeRadius, Math.min(cakeRadius, intersection.x));
        const clampedZ = Math.max(-cakeRadius, Math.min(cakeRadius, intersection.z));

        // 更新装饰物的位置
        selectedDecoration.value.position.set(clampedX, selectedDecoration.value.position.y, clampedZ);

        // 检查是否超出蛋糕边界
        const isOutOfBoundary = Math.abs(selectedDecoration.value.position.x) > cakeRadius ||
                                Math.abs(selectedDecoration.value.position.z) > cakeRadius;

        console.log('装饰物位置:', selectedDecoration.value.position);
        console.log('是否超出边界:', isOutOfBoundary);

        if (isOutOfBoundary) {
          // 弹出确认对话框
          const confirmed = confirm('装饰物超出蛋糕边界，确定要删除吗？');
          if (confirmed) {
            deleteSelectedDecoration(); // 删除装饰物
          } else {
            // 如果用户选择不删除，则将装饰物移回边界内
            selectedDecoration.value.position.set(clampedX, selectedDecoration.value.position.y, clampedZ);
          }
        }

        // 检查重叠
        const selectedBoundingBox = new THREE.Box3().setFromObject(selectedDecoration.value);
        let overlap = false;

        for (const decoration of loadedDecorations) {
          if (decoration !== selectedDecoration.value) { // 跳过自己
            const boundingBox = new THREE.Box3().setFromObject(decoration);
            if (selectedBoundingBox.intersectsBox(boundingBox)) {
              overlap = true;
              break; // 找到重叠后可以退出循环
            }
          }
        }

        // 处理重叠情况
        if (overlap) {
          console.log('装饰物重叠了！');
          // 可以添加视觉提示，例如改变颜色或显示消息
        } else {
          console.log('装饰物没有重叠');
        }

        renderScene();
      }
    };

    // 处理鼠标和触摸抬起事件
    const onPointerUp = () => {
      if (selectedDecoration.value) {
        selectedDecoration.value.traverse((child) => {
          if (child.isMesh) {
            child.material.emissive.set(0x000000);
          }
        });
        selectedDecoration.value = null;
      }
      isDragging = false;
      controls.enabled = true; // 重新启用 OrbitControls
    };

    // 初始化事件监听器，包括触摸事件
    const initEventListeners = () => {
      // 鼠标事件
      window.addEventListener('mousedown', onPointerDown);
      window.addEventListener('mousemove', onPointerMove);
      window.addEventListener('mouseup', onPointerUp);

      // 触摸事件
      window.addEventListener('touchstart', onPointerDown, { passive: false });
      window.addEventListener('touchmove', onPointerMove, { passive: false });
      window.addEventListener('touchend', onPointerUp, { passive: false });
    };

    // 移除事件监听器
    const removeEventListeners = () => {
      // 鼠标事件
      window.removeEventListener('mousedown', onPointerDown);
      window.removeEventListener('mousemove', onPointerMove);
      window.removeEventListener('mouseup', onPointerUp);

      // 触摸事件
      window.removeEventListener('touchstart', onPointerDown);
      window.removeEventListener('touchmove', onPointerMove);
      window.removeEventListener('touchend', onPointerUp);
    };

    // 渲染场景
    const renderScene = () => {
      if (renderer && scene && camera) {
        renderer.render(scene, camera);
      }
    };

    // 动画循环
    const animate = () => {
      requestAnimationFrame(animate);
      if (controls) controls.update();
      renderScene();
    };

    // 生命周期钩子
    onMounted(() => {
      initThreeJS();
      createCake();
      fetchDecorations(); // 获取装饰列表
    });

    onBeforeUnmount(() => {
      removeEventListeners();
      if (renderer) {
        renderer.dispose();
        if (threeContainer.value && renderer.domElement.parentNode === threeContainer.value) {
          threeContainer.value.removeChild(renderer.domElement);
        }
      }
    });

    // 增加层数
    const increaseLayers = () => {
  layers.value = Math.floor(layers.value) + 1;
    console.log(`增加层数，当前层数: ${layers.value}`);
    createCake(); // 重新创建蛋糕
    selectedLayer.value = layers.value - 1; // 设置为新的顶层
    console.log(`当前选择的层数: ${selectedLayer.value}`);
    };

    // 减少层数
    const decreaseLayers = () => {
      if (layers.value > 1) {
  layers.value = Math.floor(layers.value) - 1;
        console.log(`减少层数，当前层数: ${layers.value}`);
        createCake(); // 重新创建蛋糕
        // 如果当前选中的层超过新的层数，调整为新的顶层
        if (selectedLayer.value >= layers.value) {
          selectedLayer.value = layers.value - 1;
        }
      }
    };
 
 const deleteSelectedDecoration = () => {
   console.log('正在删除选中的装饰物', selectedDecoration.value);
 
   if (selectedDecoration.value) {
     scene.remove(selectedDecoration.value);
     loadedDecorations = loadedDecorations.filter(dec => dec !== selectedDecoration.value);
     selectedDecoration.value = null; // 清空选中的装饰物
 
     renderScene(); // 确保场景更新
   } else {
     console.log('没有选中的装饰物，无法删除');
   }
 };

    return {
      layers,
      selectedLayer,
      threeContainer,
      decorations,
      selectedDecoration,
      addDecoration,
      changeSelectedLayerColor,
      increaseLayers,
      decreaseLayers,
      deleteSelectedDecoration, // 确保这里有
    };
  },
};
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  height: 100vh;
  padding: 10px;
}

.three-container {
  width: 100%;
  height: 50vh;
  border: 1px solid #ccc;
}

.controls {
  margin-top: 10px;
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.layer-controls {
  margin: 10px 0;
}

button {
  margin: 5px;
  padding: 10px;
  font-size: 14px; /* 减少字体大小以适应移动设备 */
  cursor: pointer;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.decorations-list {
  margin-top: 20px;
  width: 100%;
}

.decorations-container {
  display: flex;
  overflow-x: auto; /* 使用 auto 确保在移动端可以滚动 */
  white-space: nowrap;
  width: 100%;
  padding: 10px 0;
}

.decoration-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-right: 10px;
  flex: 0 0 auto; /* Prevent items from shrinking */
  text-align: center;
}

.decoration-item img {
  width: 60px; /* 缩小装饰物图片尺寸 */
  height: 60px;
  object-fit: cover;
  margin-bottom: 5px;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .three-container {
    height: 40vh;
  }

  .controls {
    width: 100%;
  }

  .decorations-container {
    flex-direction: row;
    overflow-x: scroll;
  }

  button {
    font-size: 12px; /* 进一步缩小字体 */
    padding: 8px;
  }

  .decoration-item img {
    width: 50px;
    height: 50px;
  }
}
</style>