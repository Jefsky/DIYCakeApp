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
            <!-- 删除&清空装饰 -->
            <div class="layer-controls">
                <button @click="deleteModel" >删除当前装饰</button>
                <button @click="deleteAllModel" >清空装饰</button>
            </div>

        </div>
    </div>
</template>

<script>
    import {
        ref,
        onMounted,
        onBeforeUnmount,
        watch
    } from 'vue';
    import axios from 'axios';
    import * as THREE from 'three';
    import {
        OrbitControls
    } from 'three/examples/jsm/controls/OrbitControls';
    import {
        GLTFLoader
    } from 'three/examples/jsm/loaders/GLTFLoader';
    import {
        randFloat
    } from 'three/src/math/MathUtils';

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
            
            let selectUuid = '';

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
                renderer = new THREE.WebGLRenderer({
                    antialias: !isMobile
                });
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
                const plateMaterial = new THREE.MeshStandardMaterial({
                    color: '#C0C0C0'
                });
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
                    const material = new THREE.MeshStandardMaterial({
                        color: '#FFCC00'
                    });
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

                    // 确保加载的模型是 THREE.Object3D 实例
                    if (!(decoration instanceof THREE.Object3D)) {
                        console.error('加载的模型不是 THREE.Object3D 实例:', decoration);
                        return;
                    }

                    const layerHeight = 0.5; // 每层的高度
                    const layerIndex = selectedLayer.value; // 当前选中的层数
                    const decorationYOffset = 0.2; // 装饰物与层的偏移量

                    // 获取装饰物的边界盒
                    const box = new THREE.Box3().setFromObject(decoration);
                    const decorationHeight = box.max.y - box.min.y;
                    const decorationWidth = box.max.x - box.min.x;

                    const baseRadius = 1.5; // 蛋糕的基础半径
                    const layerRadius = baseRadius * (1 - layerIndex * 0.1);
                    const scaleRatio = (layerRadius / (decorationWidth / 2)) * 0.2; // 等比例缩放

                    decoration.scale.set(scaleRatio, scaleRatio, scaleRatio); // 设置等比例缩放
                    decoration.position.set(0, layerIndex * layerHeight + decorationYOffset, 0);


                    scene.add(decoration);
                    loadedDecorations.push(decoration); // 确保这是 THREE.Object3D 实例
                    selectedDecoration.value = decoration;

                    // 更新装饰物的包围盒
                    decoration.boundingBox = new THREE.Box3().setFromObject(decoration);
                    console.log("装饰物已添加:", decoration);
                    selectUuid = decoration.uuid;   // 绑定当前装饰，用于删除
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
                        const model = gltf.scene; // 确保这是 THREE.Object3D 实例
                        resolve(model);
                    }, undefined, (error) => {
                        console.error('加载模型时出错:', error);
                        reject(error);
                    });
                });
            };

            // 删除模型的函数
            const deleteModel = () => {
                if(!selectUuid){
                    console.error('当前没有选中装饰');
                }
                // console.log(loadedDecorations);
                // console.log(selectUuid);
                // 在 loadedDecorations 中找到对应的模型
                loadedDecorations.forEach((decoration) => {
                    if (!(decoration instanceof THREE.Object3D)) {
                        console.error('装饰物不是 THREE.Object3D 实例:', decoration);
                        return; // 跳过这个装饰物
                    }
                    if (selectUuid == decoration.uuid){
                        console.log('se',decoration)
                        console.log(decoration.object)
                        scene.remove(decoration);
                        // 还可以选择释放模型资源，防止内存泄漏
                       decoration.traverse(function(child) {
                            if (child instanceof THREE.Mesh) {
                                child.geometry.dispose();
                                child.material.dispose();
                            }
                        });
                        // 从 loadedModels 数组中移除该模型
                        const index = loadedDecorations.indexOf(decoration);
                        loadedDecorations.splice(index, 1);
                        // console.log(loadedDecorations)
                        selectUuid = '';    // 清空选中
                    };
                });
            }
            
            // 清空装饰
            const deleteAllModel = () => {
                // 清空装饰物
                loadedDecorations.forEach(decoration => {
                    scene.remove(decoration); // 从场景中移除装饰物
                });
                loadedDecorations = []; // 清空装饰物数组
                selectedDecoration.value = null; // 清空选中的装饰物
            }


            // 获取装饰物列表
            const fetchDecorations = async () => {
                try {
                    const response = await axios.get(
                        'https://develop-wxmp-api.blacktechcake.com/3dm/list?page=1&pageSize=20&type_id=13&user_id=100'
                    );
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
                    selectUuid = selectedDecoration.value.uuid;   // 绑定当前装饰，用于删除
                    selectedDecoration.value.traverse((child) => {
                        if (child.isMesh) {
                            child.material.emissive.set(0x444444);
                        }
                    });
                    isDragging = true;
                    controls.enabled = false; // 禁用 OrbitControls 以避免冲突
                }
            };

            const onPointerMove = (event) => {
                if (!isDragging || !selectedDecoration.value) return;
                if (!threeContainer.value) return;

                const rect = threeContainer.value.getBoundingClientRect();

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
                    const cakeRadius = 1.5;

                    const clampedX = Math.max(-cakeRadius, Math.min(cakeRadius, intersection.x));
                    const clampedZ = Math.max(-cakeRadius, Math.min(cakeRadius, intersection.z));

                    selectedDecoration.value.position.set(clampedX, selectedDecoration.value.position.y, clampedZ);

                    if (!Array.isArray(loadedDecorations)) {
                        console.error('loadedDecorations 不是一个数组');
                        return; // 退出函数
                    }

                    const selectedDecorationBbox = new THREE.Box3();
                    if (selectedDecoration.value) {
                        selectedDecorationBbox.setFromObject(selectedDecoration.value);
                    } else {
                        console.error('选中的装饰物无效');
                        return; // 退出函数
                    }

                    let isIntersecting = false;

                    loadedDecorations.forEach((decoration) => {
                        if (!(decoration instanceof THREE.Object3D)) {
                            console.error('装饰物不是 THREE.Object3D 实例:', decoration);
                            return; // 跳过这个装饰物
                        }
                        if (selectedDecoration.value.uuid == decoration.uuid) return;

                        const otherDecorationBbox = new THREE.Box3().setFromObject(decoration);
                        if (selectedDecorationBbox.intersectsBox(otherDecorationBbox)) {
                            isIntersecting = true;
                            console.log(`${selectedDecoration.value.uuid} 和 ${decoration.uuid} 相交`);
                            // 将选中的装饰物变为红色
                            selectedDecoration.value.traverse((child) => {
                                if (child.isMesh) {
                                    child.material.emissive.set(0xff0000); // 设置为红色
                                }
                            });
                            // console.log("相交的装饰物位置:", decoration.position);
                        }
                    });


                    if (!isIntersecting) {
                        // 如果没有相交，选中的装饰物恢复默认颜色
                        selectedDecoration.value.traverse((child) => {
                            if (child.isMesh) {
                                child.material.emissive.set(0x444444); // 恢复默认颜色
                            }
                        });
                        console.log("没有相交的装饰物");
                    }

                    renderScene();
                }
            };


            const onPointerUp = () => {
                if (selectedDecoration.value) {
                    selectedDecoration.value.traverse((child) => {
                        if (child.isMesh) {
                            child.material.emissive.set(0x000000);
                        }
                    });

                    // 获取选中装饰物的包围盒
                    const selectedDecorationBbox = new THREE.Box3().setFromObject(selectedDecoration.value);
                    let isIntersecting = false;

                    // 存储所有相交的装饰物的包围盒
                    const intersectingBoxes = [];

                    loadedDecorations.forEach((decoration) => {
                        if (selectedDecoration.value.uuid == decoration.uuid) return; // 跳过自身

                        const otherDecorationBbox = new THREE.Box3().setFromObject(decoration);
                        if (selectedDecorationBbox.intersectsBox(otherDecorationBbox)) {
                            isIntersecting = true;
                            intersectingBoxes.push(otherDecorationBbox); // 记录相交的包围盒
                        }
                    });

                    // 如果相交，移动装饰物
                    if (isIntersecting) {
                        // 计算新的位置，直到不再与任何相交的装饰物相交
                        let movedOut = false;
                        let attempts = 0; // 用于限制尝试次数

                        while (!movedOut && attempts < 50) { // 限制尝试次数防止死循环
                            // 随机移动装饰物
                            const offsetX = randFloat(-0.5, 0.5);
                            const offsetZ = randFloat(-0.5, 0.5);
                            selectedDecoration.value.position.x += offsetX;
                            selectedDecoration.value.position.z += offsetZ;

                            // 更新选中装饰物的包围盒
                            const newBbox = new THREE.Box3().setFromObject(selectedDecoration.value);
                            movedOut = true;

                            // 检查是否与任何相交的包围盒相交
                            for (const bbox of intersectingBoxes) {
                                if (newBbox.intersectsBox(bbox)) {
                                    movedOut = false; // 如果相交，则标记为未移动成功
                                    break;
                                }
                            }

                            attempts++;
                        }

                        console.log(`${selectedDecoration.value.uuid} 被移动到新位置:`, selectedDecoration.value.position);
                    }

                    selectedDecoration.value = null; // 清空选中的装饰物
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
                window.addEventListener('touchstart', onPointerDown, {
                    passive: false
                });
                window.addEventListener('touchmove', onPointerMove, {
                    passive: false
                });
                window.addEventListener('touchend', onPointerUp, {
                    passive: false
                });
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
                selectedLayer.value = layers.value; // 设置为新的顶层
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
                deleteModel,
                deleteAllModel,
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
        font-size: 14px;
        /* 减少字体大小以适应移动设备 */
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
        overflow-x: auto;
        /* 使用 auto 确保在移动端可以滚动 */
        white-space: nowrap;
        width: 100%;
        padding: 10px 0;
    }

    .decoration-item {
        display: flex;
        flex-direction: column;
        align-items: center;
        margin-right: 10px;
        flex: 0 0 auto;
        /* Prevent items from shrinking */
        text-align: center;
    }

    .decoration-item img {
        width: 60px;
        /* 缩小装饰物图片尺寸 */
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
            font-size: 12px;
            /* 进一步缩小字体 */
            padding: 8px;
        }

        .decoration-item img {
            width: 50px;
            height: 50px;
        }
    }
</style>
