# DIYCakeApp
基于**uni-app** 和 **Three.js** 开发一款DIY蛋糕应用。将涵盖项目设置、3D模型创建、用户交互实现、界面设计以及部署等各个方面。

## 目录

1. [项目准备](#项目准备)
2. [环境搭建](#环境搭建)
3. [创建uni-app项目](#创建uni-app项目)
4. [集成Three.js](#集成threejs)
5. [设计3D蛋糕模型](#设计3d蛋糕模型)
6. [实现用户交互](#实现用户交互)
7. [UI设计与集成](#ui设计与集成)
8. [测试与调试](#测试与调试)
9. [部署与发布](#部署与发布)
10. [资源与参考](#资源与参考)

---

## 项目准备

在开始开发之前，确保具备以下知识和工具：

- **基本知识**：
  - JavaScript、HTML、CSS
  - Vue.js（因为uni-app基于Vue）
  - Three.js的基础知识

- **工具**：
  - 代码编辑器（如VS Code）
  - Node.js 和 npm
  - HBuilderX（官方推荐的uni-app开发工具）
  
## 环境搭建

### 1. 安装Node.js和npm

首先，确保的开发环境中安装了Node.js和npm。可以从 [Node.js官网](https://nodejs.org/) 下载并安装最新的LTS版本。

```bash
node -v
npm -v
```

### 2. 安装HBuilderX

HBuilderX 是由DCloud推出的支持uni-app开发的IDE。下载并安装HBuilderX：

- [HBuilderX下载链接](https://www.dcloud.io/hbuilderx.html)

### 3. 安装uni-app CLI（可选）

使用命令行，可以安装uni-app的CLI工具：

```bash
npm install -g @vue/cli
npm install -g @dcloudio/uni-app
```

## 创建uni-app项目

### 1. 使用HBuilderX创建项目

1. 打开HBuilderX。
2. 点击“文件” > “新建” > “项目”。
3. 选择“uni-app”模板，填写项目名称（如`DIYCakeApp`），选择项目保存路径，点击“创建”。

### 2. 使用命令行创建项目（可选）

如果使用命令行：

```bash
vue create -p dcloudio/uni-preset-vue DIYCakeApp
cd DIYCakeApp
```

## 集成Three.js

### 1. 安装Three.js

在项目根目录下，通过npm安装Three.js：

```bash
npm install three
```

### 2. 引入Three.js

在需要使用Three.js的页面或组件中引入：

```javascript
import * as THREE from 'three';
```

## 设计3D蛋糕模型

### 1. 准备3D模型

使用Three.js自带的几何体（如圆柱体、球体等）组合成蛋糕，也可以使用3D建模软件（如Blender）创建复杂模型，并导出为Three.js支持的格式（如GLTF）。

### 2. 使用Three.js创建基本蛋糕模型

以下是一个简单的蛋糕模型示例：

```javascript
// 创建场景
const scene = new THREE.Scene();

// 创建相机
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.z = 5;

// 创建渲染器
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);

// 将渲染器的DOM添加到页面
document.getElementById('three-container').appendChild(renderer.domElement);

// 添加蛋糕主体（圆柱体）
const cakeGeometry = new THREE.CylinderGeometry(1, 1, 0.5, 32);
const cakeMaterial = new THREE.MeshBasicMaterial({ color: 0xff69b4 });
const cake = new THREE.Mesh(cakeGeometry, cakeMaterial);
scene.add(cake);

// 添加蛋糕装饰（例如球体表示水果）
const decorationGeometry = new THREE.SphereGeometry(0.1, 16, 16);
const decorationMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
const decoration = new THREE.Mesh(decorationGeometry, decorationMaterial);
decoration.position.set(0.5, 0.3, 0);
scene.add(decoration);

// 渲染循环
function animate() {
  requestAnimationFrame(animate);
  
  // 旋转蛋糕
  cake.rotation.y += 0.01;
  decoration.rotation.y += 0.02;
  
  renderer.render(scene, camera);
}
animate();
```

### 3. 在uni-app中集成3D场景

在uni-app页面中，添加一个用于渲染Three.js的容器。例如，在`pages/index/index.vue`中：

```html
<template>
  <view class="container">
    <view id="three-container" class="three-container"></view>
  </view>
</template>

<script>
import * as THREE from 'three';

export default {
  onReady() {
    this.initThree();
  },
  methods: {
    initThree() {
      // 与上述示例相同的Three.js初始化代码
      const scene = new THREE.Scene();
      const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
      camera.position.z = 5;

      const renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      document.getElementById('three-container').appendChild(renderer.domElement);

      const cakeGeometry = new THREE.CylinderGeometry(1, 1, 0.5, 32);
      const cakeMaterial = new THREE.MeshBasicMaterial({ color: 0xff69b4 });
      const cake = new THREE.Mesh(cakeGeometry, cakeMaterial);
      scene.add(cake);

      const decorationGeometry = new THREE.SphereGeometry(0.1, 16, 16);
      const decorationMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
      const decoration = new THREE.Mesh(decorationGeometry, decorationMaterial);
      decoration.position.set(0.5, 0.3, 0);
      scene.add(decoration);

      function animate() {
        requestAnimationFrame(animate);
        cake.rotation.y += 0.01;
        decoration.rotation.y += 0.02;
        renderer.render(scene, camera);
      }
      animate();
    }
  }
}
</script>

<style>
.container {
  width: 100%;
  height: 100vh;
}
.three-container {
  width: 100%;
  height: 100%;
}
</style>
```

## 实现用户交互

为了让用户能够DIY蛋糕，需要实现以下交互功能：

1. **选择蛋糕基底**：不同的蛋糕形状或口味。
2. **添加装饰**：添加水果、奶油、糖霜等装饰。
3. **颜色选择**：更改蛋糕和装饰的颜色。
4. **视角控制**：允许用户旋转、缩放和移动视角。

### 1. 添加控制器

使用**OrbitControls**来实现视角控制。首先，引入OrbitControls：

```javascript
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
```

然后，在初始化Three.js时添加控制器：

```javascript
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true; // 使动画更流畅
```

在渲染循环中更新控制器：

```javascript
function animate() {
  requestAnimationFrame(animate);
  
  controls.update(); // 更新控制器

  cake.rotation.y += 0.01;
  decoration.rotation.y += 0.02;
  
  renderer.render(scene, camera);
}
```

### 2. 添加UI控件

在uni-app中使用`<button>`或自定义组件来作为用户交互的控件。例如，添加颜色选择按钮：

```html
<template>
  <view class="container">
    <view id="three-container" class="three-container"></view>
    <view class="controls">
      <button @click="changeCakeColor('#ff69b4')">粉红色</button>
      <button @click="changeCakeColor('#8a2be2')">蓝紫色</button>
      <!-- 更多颜色按钮 -->
    </view>
  </view>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export default {
  data() {
    return {
      cakeMaterial: null,
      decorationMaterial: null,
    }
  },
  onReady() {
    this.initThree();
  },
  methods: {
    initThree() {
      const scene = new THREE.Scene();
      const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
      camera.position.z = 5;

      const renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      document.getElementById('three-container').appendChild(renderer.domElement);

      const cakeGeometry = new THREE.CylinderGeometry(1, 1, 0.5, 32);
      this.cakeMaterial = new THREE.MeshBasicMaterial({ color: 0xff69b4 });
      const cake = new THREE.Mesh(cakeGeometry, this.cakeMaterial);
      scene.add(cake);

      const decorationGeometry = new THREE.SphereGeometry(0.1, 16, 16);
      this.decorationMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
      const decoration = new THREE.Mesh(decorationGeometry, this.decorationMaterial);
      decoration.position.set(0.5, 0.3, 0);
      scene.add(decoration);

      const controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;

      const animate = () => {
        requestAnimationFrame(animate);
        controls.update();
        cake.rotation.y += 0.01;
        decoration.rotation.y += 0.02;
        renderer.render(scene, camera);
      };
      animate();
    },
    changeCakeColor(color) {
      if (this.cakeMaterial) {
        this.cakeMaterial.color.set(color);
      }
    }
  }
}
</script>

<style>
.container {
  position: relative;
  width: 100%;
  height: 100vh;
}
.three-container {
  width: 100%;
  height: 100%;
}
.controls {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 10px;
}
button {
  padding: 10px 20px;
  background-color: #ffffffaa;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}
button:hover {
  background-color: #ffffff;
}
</style>
```

### 3. 动态添加装饰

允许用户从预设的装饰列表中选择并添加到蛋糕上。以下是一个简单的实现示例：

```javascript
methods: {
  // ...之前的代码
  addDecoration(color) {
    const decorationGeometry = new THREE.SphereGeometry(0.1, 16, 16);
    const decorationMaterial = new THREE.MeshBasicMaterial({ color });
    const decoration = new THREE.Mesh(decorationGeometry, decorationMaterial);
    decoration.position.set(Math.random() - 0.5, Math.random() * 0.5, Math.random() - 0.5);
    this.$refs.scene.add(decoration);
  }
}
```

在UI中添加按钮调用`addDecoration`方法：

```html
<button @click="addDecoration('#00ff00')">添加绿色装饰</button>
<button @click="addDecoration('#0000ff')">添加蓝色装饰</button>
```

**注意**：确保在Three.js的初始化代码中将场景对象存储在`this.$refs.scene`或其他可访问的位置。

## UI设计与集成

为了提供良好的用户体验，UI设计至关重要。以下是一些建议：

### 1. 布局设计

- **3D视图区域**：占据大部分屏幕，用于展示和交互3D蛋糕。
- **控制面板**：放置在侧边或底部，用于选择蛋糕基底、添加装饰、颜色选择等。
- **预览与保存**：提供预览功能和保存定制蛋糕的选项。

### 2. 使用uni-app的组件

利用uni-app丰富的组件库来构建界面，例如：

- `uView` 或 `Vant Weapp` 等第三方UI库，可以加快开发速度。
- 使用`<button>`、`<picker>`、`<slider>`等组件实现用户交互。

### 3. 响应式设计

确保应用在不同设备（手机、平板等）上都有良好的显示效果。利用CSS的Flex布局或Grid布局来实现响应式设计。

```css
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
}
.three-container {
  flex: 1;
}
.controls {
  padding: 10px;
  background-color: #f0f0f0;
}
```

## 测试与调试

### 1. 使用HBuilderX的预览功能

HBuilderX提供了丰富的预览选项，可以在手机、浏览器中实时预览应用效果。

- **手机预览**：通过扫码预览在真实设备上的效果。
- **浏览器预览**：在浏览器中打开，方便调试和测试。

### 2. 调试Three.js

Three.js的渲染问题可以通过浏览器的开发者工具进行调试。检查控制台错误、查看渲染效果是否符合预期。

### 3. 性能优化

- **减少多边形数量**：复杂的3D模型会影响性能，尽量优化模型。
- **使用纹理贴图**：合理使用纹理可以减少几何体的复杂度。
- **启用渲染器的性能选项**：如抗锯齿、阴影等根据需要开启或关闭。

## 部署与发布

### 1. 打包应用

使用HBuilderX的打包功能，将应用打包成Web应用、小程序、APP等格式。

- 点击“发行” > 选择目标平台（如Web、微信小程序、APP等）。
- 按照指引完成打包流程。

### 2. 发布到应用商店或服务器

根据目标平台，将应用发布到相应的应用商店或部署到Web服务器。

- **Web应用**：将打包后的文件上传到服务器，配置域名和SSL等。
- **移动应用**：通过相应的应用商店（如Apple App Store、Google Play）发布。
- **小程序**：通过微信公众平台等发布。

### 3. 持续更新与维护

根据用户反馈和需求，不断优化和更新应用，修复bug，添加新功能。

## 资源与参考

### 官方文档

- **uni-app**：
  - 官方网站：[https://uniapp.dcloud.io/](https://uniapp.dcloud.io/)
  - 文档：[https://uniapp.dcloud.io/docs](https://uniapp.dcloud.io/docs)

- **Three.js**：
  - 官方网站：[https://threejs.org/](https://threejs.org/)
  - 文档：[https://threejs.org/docs/](https://threejs.org/docs/)

### 教程与示例

- **uni-app教程**：
  - [uni-app快速入门教程](https://www.dcloud.io/tutorial.html)
  
- **Three.js教程**：
  - [Three.js入门教程](https://threejsfundamentals.org/)
  - [Three.js官方示例](https://threejs.org/examples/)

### 资源素材

- **3D模型资源**：
  - [Sketchfab](https://sketchfab.com/)
  - [TurboSquid](https://www.turbosquid.com/)

- **纹理贴图**：
  - [CC0 Textures](https://cc0textures.com/)
  - [Textures.com](https://www.textures.com/)

### 社区与支持

- **Stack Overflow**：提问和搜索相关问题。
- **GitHub**：查找开源项目和示例代码。
- **Three.js论坛**：[https://discourse.threejs.org/](https://discourse.threejs.org/)

---
