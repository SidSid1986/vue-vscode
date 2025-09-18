<template>
  <div class="img-container">
    <div class="toolbar">
      <div class="free-content">
        <div class="btn-free-pick">
          <button
            @click="setFreeDrawingMode"
            class="btn-free"
            :class="{ active: isDrawing }"
          >
            ✏️ 画笔
          </button>
          <!-- <el-color-picker
            @change="updateBrushColor"
            v-model="brushColor"
            size="small"
            show-alpha
            :predefine="predefineColors"
          /> -->
          <input
            class="color-picker"
            type="color"
            v-model="brushColor"
            @input="updateBrushColor"
            title="选择画笔颜色"
          />
        </div>
        <div class="free-size">
          <span>粗细: {{ brushSize }}px </span>
          <input
            class="brush-slider"
            type="range"
            min="1"
            max="20"
            v-model="brushSize"
            @input="updateBrushSize"
          />
        </div>
      </div>
      <div class="tool-mid">
        <button @click="setRectangleDragMode(true)" class="rectangle-btn">
          🔲 矩形
        </button>
        <button @click="setCircleDragMode(true)" class="circle-btn">
          ⭕ 圆
        </button>
        <button @click="setEllipseDragMode(true)" class="ellipse-btn">
          🥚 椭圆
        </button>
        <button
          @click="setArrowDragMode(true)"
          :class="{ active: isArrowDragMode }"
        >
          ​​➡️​​ 箭头
        </button>

        <button @click="setTriangleDragMode(true)" class="triangle-btn">
          🔺 三角形
        </button>
        <button @click="setTextMode(true)" class="text-btn">📝文本</button>
      </div>
      <div class="too-edit">
        <button @click="deleteSelected" class="delete-btn">🗑️ 删除选中</button>
        <button @click="exportImage" class="export-btn">📥 导出图片</button>
        <!-- <button @click="saveCanvas" class="save-btn">💾 保存画布</button>
        <button @click="loadCanvas()" class="load-btn">🔄 回显画布</button> -->
      </div>
    </div>
    <div ref="exportWrapper" class="img-wrapper export-image-wrapper">
      <!-- <img
        ref="imageElement"
        class="norem-img-content"
        :src="`${baseUrl}/get_fetch_image`"
        alt=""
        @load="onImageLoad"
        @error="onImageError"
      /> -->

      <!-- 项目暂用 -->

      <img
        ref="imageElement"
        class="norem-img-content"
        :src="imageUrl"
        alt=""
        @load="onImageLoad"
        @error="onImageError"
      />
      <!-- <img
        ref="imageElement"
        class="norem-img-content"
        :src="`${baseUrl}/api/get_fetch_image?camera_index=${valueCamera}&t=${imageCounter}`"
        alt=""
        @load="onImageLoad"
        @error="onImageError"
      /> -->
      <!-- <img
        ref="imageElement"
        class="norem-img-content"
        :src="`${baseUrl}/api/get_fetch_image?camera_index=${valueCamera}`"
        alt=""
        @load="onImageLoad"
        @error="onImageError"
      /> -->
      <!-- <img
        ref="imageElement"
        class="norem-img-content"
        src="../../assets/123.jpg"
        alt=""
        @load="onImageLoad"
      /> -->
      <canvas
        ref="canvasEl"
        class="fabric-canvas"
        style="position: absolute; top: 0; left: 0"
      ></canvas>
      <div class="noImg" v-if="noImg">暂无图像</div>
    </div>

    <div class="camera-wrapper">
      <div class="camera-border">
        <div class="camera-img-info">
          <span
            >图像获取耗时:<span class="blue">{{
              imageInfoData.latest_frame_time
            }}</span></span
          >
          <span
            >图像处理耗时:<span class="blue">{{ processTimeProp }}</span></span
          >
        </div>
        <div class="camera-local-info">
          <!-- <span>当前相机信息:测试--{{ valueCamera }}</span> -->
          <span>当前相机信息:</span>
          <el-select
            v-model="valueCamera"
            placeholder="请选择相机"
            style="width: 120px"
            @change="changeCamera"
            no-data-text="暂无数据"
          >
            <el-option
              v-for="(item, index) in optionsCamera"
              :key="index"
              :label="item.model_name"
              :value="index"
            />
          </el-select>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, onBeforeUnmount, watch } from "vue";
import { fabric } from "fabric";
import bgImage from "@/assets/123.jpg";
import pen from "@/assets/pen.png";
import {
  fetchImage,
  processImage,
  stopImg,
  cameraList,
  imageInfo,
} from "@/api/common";

import axios from "axios";

import html2canvas from "html2canvas";

const props = defineProps({
  processTimeProp: {
    type: Number,
    default: "",
  },
});

const imageInfoData = ref({});

const valueCamera = ref(null);

// const optionsCamera = [
//   {
//     value: "Option1",
//     label: "Option1",
//   },
//   {
//     value: "Option2",
//     label: "Option2",
//   },
//   {
//     value: "Option3",
//     label: "Option3",
//   },
//   {
//     value: "Option4",
//     label: "Option4",
//   },
//   {
//     value: "Option5",
//     label: "Option5",
//   },
// ];

const optionsCamera = ref([]);

const imageCounter = ref(0);

const noImg = ref(false);
const isImageReady = ref(false);
const imageElement = ref(null);
const baseUrl = ref("");
const imageUrl = ref("");

// 用于控制请求的 AbortController
let controller = null;

// const baseUrl = import.meta.env.VITE_APP_API_HOST;
// const baseUrl = import.meta.env.VITE_APP_IMG_HOST;
const predefineColors = ref([
  "#ff4500",
  "#ff8c00",
  "#ffd700",
  "#90ee90",
  "#00ced1",
  "#1e90ff",
  "#c71585",
  "rgba(255, 69, 0, 0.68)",
  "rgb(255, 120, 0)",
  "hsv(51, 100, 98)",
  "hsva(120, 40, 94, 0.5)",
  "hsl(181, 100%, 37%)",
  "hsla(209, 100%, 56%, 0.73)",
  "#c7158577",
]);

// Refs
const canvasEl = ref(null);
let canvas = null;

// 图片原始尺寸和显示尺寸
let imageNaturalWidth = 0;
let imageNaturalHeight = 0;
let imageDisplayWidth = 0;
let imageDisplayHeight = 0;

// 画笔状态
const isDrawing = ref(false);
const brushColor = ref("#ff0000");
const brushSize = ref(3);
const isArrowDragMode = ref(false);
const arrowDragStartPoint = ref(null);

// 矩形拖拽模式
const isRectangleDragMode = ref(false);
const rectangleDragStartPoint = ref(null);
let previewRect = null; //清除预览

// 圆形拖拽模式
const isCircleDragMode = ref(false);
const circleDragStartPoint = ref(null);
let previewCircle = null;

// 椭圆拖拽模式
const isEllipseDragMode = ref(false);
const ellipseDragStartPoint = ref(null);
let previewEllipse = null; // 用于清除预览

// 三角形拖拽模式
const isTriangleDragMode = ref(false);
const triangleDragStartPoint = ref(null);
let previewTriangle = null; // 用于清除预览

// 预览相关
let previewGroup = null;
let previewLine = null;
let previewArrowHead = null;

// 存储所有图片/画布状态，每个都有唯一 id 和 canvas 数据
const canvasStates = ref([
  // 初始可以为空，或者放一个默认项
]);

// 文本模式（新增！）
const isTextMode = ref(false);

//camera list
const getCameraList = async () => {
  const res = await cameraList();
  console.log(res);
  optionsCamera.value = res.devices;
};

//image info
const getImageInfo = async () => {
  const res = await imageInfo();
  console.log(res);
  imageInfoData.value = res;
};

const changeCamera = (e) => {
  console.log(e);

  // setTimeout(() => {
  //   getImageInfo();
  // }, 3000);
  //
};

// 更新画笔颜色
const updateBrushColor = () => {
  if (!canvas?.freeDrawingBrush) return;
  canvas.freeDrawingBrush.color = brushColor.value;
};

// 更新画笔粗细
const updateBrushSize = () => {
  if (!canvas?.freeDrawingBrush) return;
  canvas.freeDrawingBrush.width = Number(brushSize.value);
};

// 设置自由绘制模式
const setFreeDrawingMode = () => {
  if (!canvas) return;

  isDrawing.value = !isDrawing.value; // 反转当前状态

  if (isDrawing.value) {
    canvas.isDrawingMode = true;
    canvas.freeDrawingBrush = new fabric.PencilBrush(canvas);
    canvas.freeDrawingBrush.color = brushColor.value;
    canvas.freeDrawingBrush.width = Number(brushSize.value);
    canvas.freeDrawingCursor = `url(${pen}) 2 30, auto`;
  } else {
    canvas.isDrawingMode = false;
    canvas.freeDrawingBrush = null; // 可选：清理画笔
    canvas.defaultCursor = "default";
  }
};

// 添加矩形

// 设置矩形拖拽模式
const setRectangleDragMode = (isDragging) => {
  setMode(); // 先确保退出其他模式（比如自由绘制）
  isRectangleDragMode.value = isDragging;

  if (isDragging) {
    canvas.defaultCursor = "crosshair";
    canvas.hoverCursor = "crosshair";
    canvas.selection = false;
    clearPreviewRectangle();
  } else {
    canvas.defaultCursor = "default";
    canvas.hoverCursor = "default";
    canvas.selection = true;
    clearPreviewRectangle();
  }
};
// 矩形拖拽相关鼠标事件
const handleRectangleDragMouseDown = (opt) => {
  if (!isRectangleDragMode.value) return;
  const pointer = canvas.getPointer(opt.e);
  rectangleDragStartPoint.value = { x: pointer.x, y: pointer.y };
  clearPreviewRectangle();
};

const handleRectangleDragMouseMove = (opt) => {
  if (!isRectangleDragMode.value || !rectangleDragStartPoint.value) return;

  const pointer = canvas.getPointer(opt.e);
  const start = rectangleDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };

  // 1. 清除旧的预览
  clearPreviewRectangle();

  // 2. 创建新的预览矩形
  previewRect = createPreviewRectangle(start.x, start.y, end.x, end.y);
  if (previewRect) {
    canvas.add(previewRect);
    canvas.renderAll();
  }
};

const handleRectangleDragMouseUp = (opt) => {
  if (!isRectangleDragMode.value || !rectangleDragStartPoint.value) return;
  const pointer = canvas.getPointer(opt.e);
  const start = rectangleDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };
  clearPreviewRectangle();
  const shape = drawRectangle(start.x, start.y, end.x, end.y);
  if (shape) {
    canvas.add(shape);
    canvas.renderAll();
  }
  rectangleDragStartPoint.value = null;
  setRectangleDragMode(false);
};

// 创建矩形预览（拖的过程中显示）
const createPreviewRectangle = (startX, startY, endX, endY) => {
  const width = endX - startX;
  const height = endY - startY;
  return new fabric.Rect({
    left: startX,
    top: startY,
    width: Math.abs(width),
    height: Math.abs(height),
    fill: "rgba(0, 123, 255, 0.3)",
    stroke: "#007bff",
    strokeWidth: 2,
    selectable: false,
    evented: false,
  });
};
// 绘制正式矩形（拖完后生成）
const drawRectangle = (startX, startY, endX, endY) => {
  const width = endX - startX;
  const height = endY - startY;

  return new fabric.Rect({
    left: startX,
    top: startY,
    width: Math.abs(width),
    height: Math.abs(height),
    // fill: "rgba(0, 123, 255, 0.3)",
    fill: "transparent",
    stroke: "#007bff",
    strokeWidth: 2,
  });
};

// 清除矩形预览
const clearPreviewRectangle = () => {
  if (previewRect) {
    canvas.remove(previewRect); // ✅ 移除实际的矩形对象
    previewRect = null; // ✅ 清空引用
    canvas.renderAll(); // ✅ 刷新画布
  }
};

// 设置圆形拖拽模式
const setCircleDragMode = (isDragging) => {
  setMode(); // 确保退出其他模式，比如自由绘制
  isCircleDragMode.value = isDragging;

  if (isDragging) {
    canvas.defaultCursor = "crosshair";
    canvas.hoverCursor = "crosshair";
    canvas.selection = false;
    clearPreviewCircle();
  } else {
    canvas.defaultCursor = "default";
    canvas.hoverCursor = "default";
    canvas.selection = true;
    clearPreviewCircle();
  }
};

// 圆形拖拽 - 鼠标按下
const handleCircleDragMouseDown = (opt) => {
  if (!isCircleDragMode.value) return;
  const pointer = canvas.getPointer(opt.e);
  circleDragStartPoint.value = { x: pointer.x, y: pointer.y };
  clearPreviewCircle();
};

// 圆形拖拽 - 鼠标移动
const handleCircleDragMouseMove = (opt) => {
  if (!isCircleDragMode.value || !circleDragStartPoint.value) return;
  const pointer = canvas.getPointer(opt.e);
  const start = circleDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };
  clearPreviewCircle();
  const circle = createPreviewCircle(start.x, start.y, end.x, end.y);
  if (circle) {
    previewCircle = circle;
    canvas.add(circle);
    canvas.renderAll();
  }
};

// 圆形拖拽 - 鼠标松开
const handleCircleDragMouseUp = (opt) => {
  if (!isCircleDragMode.value || !circleDragStartPoint.value) return;
  const pointer = canvas.getPointer(opt.e);
  const start = circleDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };
  clearPreviewCircle();
  const shape = drawCircle(start.x, start.y, end.x, end.y);
  if (shape) {
    canvas.add(shape);
    canvas.renderAll();
  }
  circleDragStartPoint.value = null;
  setCircleDragMode(false);
};

// 创建圆形预览（拖的过程中显示）
const createPreviewCircle = (startX, startY, endX, endY) => {
  const radius =
    Math.sqrt(Math.pow(endX - startX, 2) + Math.pow(endY - startY, 2)) / 2;
  const cx = (startX + endX) / 2;
  const cy = (startY + endY) / 2;

  return new fabric.Circle({
    left: cx - radius,
    top: cy - radius,
    radius: radius,
    fill: "transparent", // ✅ 无填充
    stroke: brushColor.value || "#007bff", // 可使用当前画笔颜色
    strokeWidth: brushSize.value || 2,
    selectable: false,
    evented: false,
  });
};

// 绘制正式圆形（拖拽结束后生成）
const drawCircle = (startX, startY, endX, endY) => {
  const radius =
    Math.sqrt(Math.pow(endX - startX, 2) + Math.pow(endY - startY, 2)) / 2;
  const cx = (startX + endX) / 2;
  const cy = (startY + endY) / 2;

  return new fabric.Circle({
    left: cx - radius,
    top: cy - radius,
    radius: radius,
    fill: "transparent", // ✅ 无填充
    stroke: brushColor.value || "#007bff",
    strokeWidth: brushSize.value || 2,
  });
};

// 清除圆形预览
const clearPreviewCircle = () => {
  if (previewCircle) {
    canvas.remove(previewCircle);
    previewCircle = null;
    canvas.renderAll();
  }
};

// 设置椭圆拖拽模式
const setEllipseDragMode = (isDragging) => {
  setMode(); // 确保退出其他模式，比如自由绘制
  isEllipseDragMode.value = isDragging;

  if (isDragging) {
    canvas.defaultCursor = "crosshair";
    canvas.hoverCursor = "crosshair";
    canvas.selection = false;
    clearPreviewEllipse();
  } else {
    canvas.defaultCursor = "default";
    canvas.hoverCursor = "default";
    canvas.selection = true;
    clearPreviewEllipse();
  }
};

// 椭圆拖拽 - 鼠标按下
const handleEllipseDragMouseDown = (opt) => {
  if (!isEllipseDragMode.value) return;
  const pointer = canvas.getPointer(opt.e);
  ellipseDragStartPoint.value = { x: pointer.x, y: pointer.y };
  clearPreviewEllipse();
};

// 椭圆拖拽 - 鼠标移动
const handleEllipseDragMouseMove = (opt) => {
  if (!isEllipseDragMode.value || !ellipseDragStartPoint.value) return;
  const pointer = canvas.getPointer(opt.e);
  const start = ellipseDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };
  clearPreviewEllipse();
  const ellipse = createPreviewEllipse(start.x, start.y, end.x, end.y);
  if (ellipse) {
    previewEllipse = ellipse;
    canvas.add(ellipse);
    canvas.renderAll();
  }
};

// 椭圆拖拽 - 鼠标松开
const handleEllipseDragMouseUp = (opt) => {
  if (!isEllipseDragMode.value || !ellipseDragStartPoint.value) return;
  const pointer = canvas.getPointer(opt.e);
  const start = ellipseDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };
  clearPreviewEllipse();
  const shape = drawEllipse(start.x, start.y, end.x, end.y);
  if (shape) {
    canvas.add(shape);
    canvas.renderAll();
  }
  ellipseDragStartPoint.value = null;
  setEllipseDragMode(false);
};

// 创建椭圆预览（拖的过程中显示）
const createPreviewEllipse = (startX, startY, endX, endY) => {
  const rx = Math.abs(endX - startX) / 2;
  const ry = Math.abs(endY - startY) / 2;
  const cx = (startX + endX) / 2;
  const cy = (startY + endY) / 2;

  return new fabric.Ellipse({
    left: cx - rx,
    top: cy - ry,
    rx: rx,
    ry: ry,
    fill: "transparent", // ✅ 无填充
    stroke: brushColor.value || "#007bff", // 可使用当前画笔颜色
    strokeWidth: brushSize.value || 2,
    selectable: false,
    evented: false,
  });
}; // 绘制正式椭圆（拖拽结束后生成）
const drawEllipse = (startX, startY, endX, endY) => {
  const rx = Math.abs(endX - startX) / 2;
  const ry = Math.abs(endY - startY) / 2;
  const cx = (startX + endX) / 2;
  const cy = (startY + endY) / 2;

  return new fabric.Ellipse({
    left: cx - rx,
    top: cy - ry,
    rx: rx,
    ry: ry,
    fill: "transparent", // ✅ 无填充
    stroke: brushColor.value || "#007bff",
    strokeWidth: brushSize.value || 2,
  });
};

// 清除椭圆预览
const clearPreviewEllipse = () => {
  if (previewEllipse) {
    canvas.remove(previewEllipse);
    previewEllipse = null;
    canvas.renderAll();
  }
};

// 设置三角形拖拽模式
const setTriangleDragMode = (isDragging) => {
  setMode(); // 确保退出其他模式，比如自由绘制
  isTriangleDragMode.value = isDragging;

  if (isDragging) {
    canvas.defaultCursor = "crosshair";
    canvas.hoverCursor = "crosshair";
    canvas.selection = false;
    clearPreviewTriangle();
  } else {
    canvas.defaultCursor = "default";
    canvas.hoverCursor = "default";
    canvas.selection = true;
    clearPreviewTriangle();
  }
}; // 三角形拖拽 - 鼠标按下
const handleTriangleDragMouseDown = (opt) => {
  if (!isTriangleDragMode.value) return;
  const pointer = canvas.getPointer(opt.e);
  triangleDragStartPoint.value = { x: pointer.x, y: pointer.y };
  clearPreviewTriangle();
};

// 三角形拖拽 - 鼠标移动
const handleTriangleDragMouseMove = (opt) => {
  if (!isTriangleDragMode.value || !triangleDragStartPoint.value) return;
  const pointer = canvas.getPointer(opt.e);
  const start = triangleDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };
  clearPreviewTriangle();
  const triangle = createPreviewTriangle(start.x, start.y, end.x, end.y);
  if (triangle) {
    previewTriangle = triangle;
    canvas.add(triangle);
    canvas.renderAll();
  }
};

// 三角形拖拽 - 鼠标松开
const handleTriangleDragMouseUp = (opt) => {
  if (!isTriangleDragMode.value || !triangleDragStartPoint.value) return;
  const pointer = canvas.getPointer(opt.e);
  const start = triangleDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };
  clearPreviewTriangle();
  const shape = drawTriangle(start.x, start.y, end.x, end.y);
  if (shape) {
    canvas.add(shape);
    canvas.renderAll();
  }
  triangleDragStartPoint.value = null;
  setTriangleDragMode(false);
};
// 创建三角形预览（拖的过程中显示）
const createPreviewTriangle = (startX, startY, endX, endY) => {
  const size = Math.min(Math.abs(endX - startX), Math.abs(endY - startY)) / 2;
  const cx = (startX + endX) / 2;
  const cy = (startY + endY) / 2;

  return new fabric.Polygon(
    [
      { x: cx, y: cy - size }, // 顶点
      { x: cx - size, y: cy + size }, // 左下
      { x: cx + size, y: cy + size }, // 右下
    ],
    {
      fill: "transparent", // ✅ 无填充
      stroke: brushColor.value || "#007bff", // 可使用当前画笔颜色
      strokeWidth: brushSize.value || 2,
      selectable: false,
      evented: false,
    }
  );
}; // 绘制正式三角形（拖拽结束后生成）
const drawTriangle = (startX, startY, endX, endY) => {
  const size = Math.min(Math.abs(endX - startX), Math.abs(endY - startY)) / 2;
  const cx = (startX + endX) / 2;
  const cy = (startY + endY) / 2;

  return new fabric.Polygon(
    [
      { x: cx, y: cy - size },
      { x: cx - size, y: cy + size },
      { x: cx + size, y: cy + size },
    ],
    {
      fill: "transparent", // ✅ 无填充
      stroke: brushColor.value || "#007bff",
      strokeWidth: brushSize.value || 2,
    }
  );
};
// 清除三角形预览
const clearPreviewTriangle = () => {
  if (previewTriangle) {
    canvas.remove(previewTriangle);
    previewTriangle = null;
    canvas.renderAll();
  }
};

// 添加文本
// 设置文字输入模式
const setTextMode = (isTexting) => {
  isTextMode.value = isTexting;

  if (isTexting) {
    canvas.defaultCursor = "text"; // 可选：鼠标样式
    canvas.hoverCursor = "text";
    canvas.selection = false;
  } else {
    canvas.defaultCursor = "default";
    canvas.hoverCursor = "default";
    canvas.selection = true;
  }
};
// 鼠标在画布上按下（通用处理，包括文字模式）
const handleCanvasMouseDown = (opt) => {
  if (isTextMode.value) {
    // 当前处于“添加文字”模式
    const pointer = canvas.getPointer(opt.e);
    const x = pointer.x;
    const y = pointer.y;

    // 创建一个可编辑的文本框
    const text = new fabric.IText("双击编辑文字", {
      left: x,
      top: y,
      fontSize: 20,
      fill: "#333",
      fontFamily: "Arial",
      editable: true,
      selectable: true,
    });

    canvas.add(text);
    canvas.setActiveObject(text);
    canvas.renderAll();

    // 退出文字模式
    setTextMode(false);
  }

  // 注意：如果你还有其他模式的 mouse:down 处理，比如箭头/矩形，你仍然需要保留它们
  // 你可以在这里调用它们，或者用模式判断分别调用
};

// 添加图形
const addShape = (factory) => {
  if (!canvas) return;
  const shape = factory();
  canvas.add(shape);
  canvas.setActiveObject(shape);
};

// 箭头相关
const setArrowDragMode = (isDragging) => {
  setMode();
  isArrowDragMode.value = isDragging;
  if (isDragging) {
    canvas.defaultCursor = "crosshair";
    canvas.hoverCursor = "crosshair";
    canvas.selection = false;
    clearPreviewArrow();
  } else {
    canvas.defaultCursor = "default";
    canvas.hoverCursor = "default";
    canvas.selection = true;
    clearPreviewArrow();
  }
};

// 鼠标事件处理
const handleArrowDragMouseDown = (opt) => {
  if (!isArrowDragMode.value) return;
  const pointer = canvas.getPointer(opt.e);
  arrowDragStartPoint.value = { x: pointer.x, y: pointer.y };
  clearPreviewArrow();
};

const handleArrowDragMouseMove = (opt) => {
  if (!isArrowDragMode.value || !arrowDragStartPoint.value) return;
  const pointer = canvas.getPointer(opt.e);
  const start = arrowDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };
  clearPreviewArrow();
  const { line, arrowHead } = createPreviewArrow(
    start.x,
    start.y,
    end.x,
    end.y
  );
  previewLine = line;
  previewArrowHead = arrowHead;
  canvas.add(line, arrowHead);
  canvas.renderAll();
};

const handleArrowDragMouseUp = (opt) => {
  if (!isArrowDragMode.value || !arrowDragStartPoint.value) return;
  const pointer = canvas.getPointer(opt.e);
  const start = arrowDragStartPoint.value;
  const end = { x: pointer.x, y: pointer.y };
  clearPreviewArrow();
  const arrowGroup = drawArrow(start.x, start.y, end.x, end.y);
  if (arrowGroup) canvas.add(arrowGroup);
  arrowDragStartPoint.value = null;
  setArrowDragMode(false);
};

// 创建预览箭头（不加入正式图形）
const createPreviewArrow = (startX, startY, endX, endY) => {
  const line = new fabric.Line([startX, startY, endX, endY], {
    stroke: brushColor.value,
    strokeWidth: brushSize.value,
    selectable: false,
    evented: false,
  });

  const angle = Math.atan2(endY - startY, endX - startX);
  const arrowHeadLength = 15;
  const headX = endX;
  const headY = endY;
  const leftX = headX - arrowHeadLength * Math.cos(angle - Math.PI / 6);
  const leftY = headY - arrowHeadLength * Math.sin(angle - Math.PI / 6);
  const rightX = headX - arrowHeadLength * Math.cos(angle + Math.PI / 6);
  const rightY = headY - arrowHeadLength * Math.sin(angle + Math.PI / 6);

  const arrowHead = new fabric.Polygon(
    [
      { x: headX, y: headY },
      { x: leftX, y: leftY },
      { x: rightX, y: rightY },
    ],
    {
      fill: brushColor.value,
      stroke: brushColor.value,
      strokeWidth: 1,
      selectable: false,
      evented: false,
      originX: "center",
      originY: "center",
    }
  );

  return { line, arrowHead };
};

// 清除预览
const clearPreviewArrow = () => {
  if (previewLine) {
    canvas.remove(previewLine);
    previewLine = null;
  }
  if (previewArrowHead) {
    canvas.remove(previewArrowHead);
    previewArrowHead = null;
  }
  canvas.renderAll();
};

// 绘制正式箭头（line + arrowHead，组合为 Group，主线不突出）
const drawArrow = (startX, startY, endX, endY) => {
  if (!canvas) return null;

  const angle = Math.atan2(endY - startY, endX - startX);
  const arrowHeadLength = 15;
  const headX = endX;
  const headY = endY;

  // 箭头主线终点往回缩一点，避免和箭头头部重叠
  const lineEndX = headX - arrowHeadLength * 0.8 * Math.cos(angle);
  const lineEndY = headY - arrowHeadLength * 0.8 * Math.sin(angle);

  const line = new fabric.Line([startX, startY, lineEndX, lineEndY], {
    stroke: brushColor.value,
    strokeWidth: brushSize.value,
    selectable: true,
    evented: true,
  });

  const leftX = headX - arrowHeadLength * Math.cos(angle - Math.PI / 6);
  const leftY = headY - arrowHeadLength * Math.sin(angle - Math.PI / 6);
  const rightX = headX - arrowHeadLength * Math.cos(angle + Math.PI / 6);
  const rightY = headY - arrowHeadLength * Math.sin(angle + Math.PI / 6);

  const arrowHead = new fabric.Polygon(
    [
      { x: headX, y: headY },
      { x: leftX, y: leftY },
      { x: rightX, y: rightY },
    ],
    {
      fill: brushColor.value,
      stroke: brushColor.value,
      strokeWidth: 1,
      selectable: true,
      evented: true,
      originX: "center",
      originY: "center",
    }
  );

  return new fabric.Group([line, arrowHead], {
    selectable: true,
    evented: true,
  });
};

// 设置为普通选择模式（退出自由绘制 / 画笔模式）
const setMode = () => {
  if (!canvas) return;

  isDrawing.value = false; // 更新状态，用于按钮样式
  canvas.isDrawingMode = false; // 重点：真正关闭自由绘制模式
  canvas.freeDrawingBrush = null; // 可选：清理画笔对象
  canvas.defaultCursor = "default"; // 恢复默认鼠标样式

  canvas.selection = true; // ✅ 确保选中功能是开启的！
};
// 导出图片(canvas背景图是导入的img)
// const exportImage = () => {
//   if (!canvas) return;
//   const dataURL = canvas.toDataURL({ format: "png", quality: 1.0 });
//   const link = document.createElement("a");
//   link.download = `canvas-image-${Date.now()}.png`;
//   link.href = dataURL;
//   link.click();
// };

// 导出图片(canvas和img分开)静态图

// const exportImage = async () => {
//   const wrapper = document.querySelector(".export-image-wrapper");
//   console.log(wrapper);

//   if (!wrapper) {
//     alert("未找到导出区域");
//     return;
//   }

//   try {
//     const canvas = await html2canvas(wrapper, {
//       backgroundColor: null, // 透明背景，如果需要白色背景可设置为 "#ffffff"
//       useCORS: true, // 如果有跨域图片可启用
//       allowTaint: true, // 允许加载跨域图片（慎用，最好确保图片同源）
//     });

//     // 创建下载链接
//     const link = document.createElement("a");
//     link.download = `full-export-${Date.now()}.png`;
//     link.href = canvas.toDataURL("image/png");
//     link.click();
//   } catch (error) {
//     console.error("导出失败：", error);
//     alert("导出失败，请重试");
//   }
// };

// const exportImage = async () => {
//   isDrawing.value = false;
//   let data = { stop: true };

//   // await stopImg(data);
//   // fetchImage().then((res) => {});

//   try {
//     // 等待 stopImg 完成
//     await stopImg(data);

//     // 查找导出区域
//     const wrapper = document.querySelector(".export-image-wrapper");
//     if (!wrapper) {
//       alert("未找到导出区域");
//       return;
//     }
//     // 使用 html2canvas 生成 canvas
//     const canvas = await html2canvas(wrapper, {
//       backgroundColor: null,
//       useCORS: true,
//       allowTaint: true, // 谨慎使用
//     });

//     // 触发下载
//     const link = document.createElement("a");
//     link.download = `full-export-${Date.now()}.png`;
//     link.href = canvas.toDataURL("image/png");
//     link.click();

//     imageCounter.value++;

//     //启动
//     // setTimeout(() => {
//     // fetchImage().then((res) => {});
//     // }, 5000);
//   } catch (error) {
//     console.error("导出失败：", error);
//     alert("导出失败，请重试");
//   }
// };

//实时图片
//   导出按钮点击时调用的方法
const exportImage = async () => {
  isDrawing.value = false;
  let data = { stop: true };
  await stopImg(data); //停止采集

  if (!imageElement.value || !canvas) {
    alert("❌ 图片或画布未初始化，请稍后再试");
    return;
  }

  const imgEl = imageElement.value; // <img ref="imageElement">
  const fabricCanvas = canvas; //  Fabric.js canvas 实例

  if (!imgEl.complete || !imgEl.naturalWidth) {
    alert("⚠️ 当前图片尚未加载完成，请稍后重试导出");
    return;
  }

  // 1. 创建一个离屏 canvas（用于最终导出）
  const exportCanvas = document.createElement("canvas");
  const ctx = exportCanvas.getContext("2d");

  // 2. 设置导出 canvas 的尺寸为图片尺寸（或者你也可以用固定画布尺寸）
  exportCanvas.width = imgEl.naturalWidth; // 或者 imageDisplayWidth，如果你缩放过
  exportCanvas.height = imgEl.naturalHeight;

  // 3. 先绘制实时图像（当前 img 标签的帧）
  ctx.drawImage(imgEl, 0, 0, exportCanvas.width, exportCanvas.height);

  // 4. 再将 Fabric.js 的内容也绘制到该 canvas 上
  // 方法：使用 fabric.Canvas 的 lowerCanvasEl（即实际渲染的 canvas），直接 drawImage 到目标 canvas
  const fabricCanvasElement = fabricCanvas.getElement(); // 获取 fabric 底层 canvas DOM
  if (fabricCanvasElement) {
    //如果fabric画布尺寸和图片不一致，可以调整绘制位置和大小
    ctx.drawImage(
      fabricCanvasElement,
      0,
      0,
      fabricCanvasElement.width,
      fabricCanvasElement.height,
      0,
      0,
      exportCanvas.width,
      exportCanvas.height // 你可以调整位置，比如居中等
    );
  } else {
    console.warn("⚠️ 无法获取 Fabric.js 的底层 canvas，仅导出背景图");
  }

  // 5. 导出为 PNG
  exportCanvas.toBlob((blob) => {
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `exported-image-${Date.now()}.png`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
    loadCameraImage(); //
  }, "image/png");

  imageCounter.value++;

  // 或者用 toDataURL 直接下载（备选）
  // const dataURL = exportCanvas.toDataURL("image/png");
  // const link = document.createElement("a");
  // link.download = `exported-image-${Date.now()}.png`;
  // link.href = dataURL;
  // link.click();
};

// 删除当前选中的图形
const deleteSelected = () => {
  if (!canvas) return;

  // 获取当前选中的对象
  const activeObject = canvas.getActiveObject();

  if (activeObject) {
    // 如果有选中的对象，删除它
    canvas.remove(activeObject);
    canvas.discardActiveObject(); // 取消选中状态
    canvas.renderAll(); // 刷新画布（通常不需要，但可确保 UI 同步）
  } else {
    // 可选：提示用户没有选中任何对象
    console.log("请先选中要删除的图形");
    // 或者用 UI 提示，比如弹窗 / Toast：提示“请先选中一个对象”
  }
};

// 保存当前画布状态，生成一个新记录
// 保存画布内容，固定 ID = "1"，同时存到 Vue 和 localStorage
const saveCanvas = () => {
  if (!canvas) return;

  const canvasData = canvas.toJSON(); // 当前画布所有内容

  const state = {
    id: "1", // 固定 ID，简化逻辑
    name: "默认画布状态（ID=1）",
    canvasData: canvasData,
  };

  // 1. 保存到 Vue 的响应式变量（用于 UI 展示等）
  const existingIndex = canvasStates.value.findIndex((s) => s.id === "1");
  if (existingIndex >= 0) {
    canvasStates.value[existingIndex] = state;
  } else {
    canvasStates.value.push(state);
  }

  // 2. 保存到 localStorage（用于持久化，刷新不丢）
  localStorage.setItem(
    "savedCanvasState_1",
    JSON.stringify(canvasStates.value)
  );

  console.log("✅ 画布已保存（固定ID=1），并写入 localStorage");
  alert("✅ 画布已保存！（数据已存到本地，刷新页面也能回显）");
};

//只保存蒙版

// const saveCanvas = () => {
//   if (!canvas) return;

//   // 获取所有绘制的图形对象
//   const objects = canvas.getObjects(); // Fabric 对象数组

//   const state = {
//     id: "1",
//     name: "默认画布状态（ID=1）",
//     objects: objects, // 只存图形对象
//   };

//   // 保存到本地
//   const existingIndex = canvasStates.value.findIndex((s) => s.id === "1");
//   if (existingIndex >= 0) {
//     canvasStates.value[existingIndex] = state;
//   } else {
//     canvasStates.value.push(state);
//   }

//   localStorage.setItem("savedCanvasState_1", JSON.stringify(canvasStates.value));

//   console.log("✅ 蒙版（objects）已保存");
//   alert("✅ 蒙版已保存！（只存了图形，不含背景图）");
// };

// 回显画布内容：从 localStorage 恢复保存的 canvasStates 数组，并加载其中 ID=1 的记录
const loadCanvas = () => {
  if (!canvas) return;

  // 1. 从 localStorage 获取保存的数据
  const savedStatesStr = localStorage.getItem("savedCanvasState_1");

  // 2. 如果没保存过，提示用户
  if (!savedStatesStr) {
    console.log("❌ 本地存储中没有找到保存的画布状态");
    alert("❌ 没有保存过画布状态，请先点击【保存】");
    return;
  }

  let savedStates;

  // 3. 安全解析 JSON，防止解析失败
  try {
    savedStates = JSON.parse(savedStatesStr);
  } catch (e) {
    console.error("解析本地存储数据失败", e);
    alert("❌ 本地存储数据格式错误，无法加载");
    return;
  }

  // 4. 检查 savedStates 是否是数组！你存的是 canvasStates.value（数组）
  if (!Array.isArray(savedStates)) {
    console.error("❌ 本地存储的数据不是数组", savedStates);
    alert("❌ 本地存储的数据格式错误，应该是一个数组");
    return;
  }

  // 5. 从数组中查找 id === "1" 的记录
  const state = savedStates.find((s) => s && s.id === "1"); // 加 s && 避免 s 是 undefined/null

  if (!state) {
    console.log("❌ 没有找到 ID=1 的画布状态");
    alert("❌ 没有找到 ID=1 的画布状态，请确认是否已保存");
    return;
  }

  if (!state.canvasData) {
    console.log("❌ ID=1 的画布状态中没有 canvasData");
    alert("❌ 保存的数据不完整，缺少画布内容");
    return;
  }

  // 6. （可选）同步回 Vue 的 canvasStates（如果你有 UI 列表要展示）
  const existingIndex = canvasStates.value.findIndex((s) => s.id === "1");
  if (existingIndex >= 0) {
    canvasStates.value[existingIndex] = state;
  } else {
    canvasStates.value.push(state);
  }

  // 7. 加载画布数据
  canvas.clear();
  canvas.setBackgroundImage(null, canvas.renderAll.bind(canvas));

  canvas.loadFromJSON(state.canvasData, () => {
    canvas.renderAll();
    console.log("✅ 画布已成功回显（来自 localStorage，ID=1）");
    alert("✅ 画布已回显！（来自本地存储）");
  });
};

//只恢复蒙版
// const loadCanvas = () => {
//   if (!canvas) return;

//   const savedStatesStr = localStorage.getItem("savedCanvasState_1");
//   if (!savedStatesStr) {
//     console.log("❌ 没有保存的蒙版");
//     alert("❌ 没有保存过蒙版，请先点击【保存】");
//     return;
//   }

//   let savedStates;
//   try {
//     savedStates = JSON.parse(savedStatesStr);
//   } catch (e) {
//     console.error("解析失败", e);
//     alert("❌ 数据格式错误");
//     return;
//   }

//   if (!Array.isArray(savedStates)) {
//     console.error("不是数组");
//     alert("❌ 数据格式错误");
//     return;
//   }

//   const state = savedStates.find((s) => s && s.id === "1");
//   if (!state || !state.objects || !Array.isArray(state.objects)) {
//     console.log("❌ 没有找到有效的蒙版数据");
//     alert("❌ 没有找到蒙版对象，请确认是否已保存");
//     return;
//   }

//   // 清空画布并移除背景
//   canvas.clear();
//   canvas.setBackgroundImage(null);
//   canvas.renderAll();

//   // 只恢复用户画的图形（蒙版）
//   canvas.add(...state.objects);
//   canvas.renderAll();

//   console.log("✅ 蒙版已成功回显！");
//   alert("✅ 蒙版已回显！（不含背景图，只恢复图形）");
// };

// 加载摄像头图像 —— 使用 axios + AbortController
const loadCameraImage = async () => {
  // 1. 取消上一次未完成的请求
  if (controller) {
    controller.cancel("取消上一个未完成的请求");
    controller = null;
  }

  // 2. 创建新的 Axios CancelToken（通过 AbortController 适配，axios 0.22+ 支持 signal）
  // 注意：axios 直接支持 signal（Fetch-like），无需手动 CancelToken
  controller = new AbortController();

  const timeoutId = setTimeout(() => {
    console.warn("⏳ [loadCameraImage] 请求超时，主动取消");
    controller.abort(); // 超时 5 秒后取消请求
  }, 5000); // 超时时间：5秒，可调整

  try {
    // 3. 拼接请求 URL
    const url = `${baseUrl.value}/api/get_fetch_image?camera_index=${valueCamera.value}&t=${imageCounter.value}`;
    console.log("[loadCameraImage] 请求图像：", url);

    // 4. 使用 axios 发起请求，传入 signal 用于取消
    const response = await axios.get(url, {
      responseType: "blob", // 重要！告诉 axios 返回的是二进制 Blob 数据
      signal: controller.signal, // 支持取消请求
    });

    // 5. 清除超时定时器
    clearTimeout(timeoutId);

    // 6. 获取图片 Blob
    const blob = response.data;

    // 7. 生成 ObjectURL，用于 <img :src>
    const objectUrl = URL.createObjectURL(blob);
    imageUrl.value = objectUrl; // 设置给 <img :src>

    console.log("✅ 图像加载成功，已设置 ObjectURL");
  } catch (err) {
    // 8. 捕获错误：网络错误、服务无响应、超时、取消等
    clearTimeout(timeoutId);

    if (axios.isCancel(err)) {
      console.warn(
        "  [loadCameraImage] 请求被取消（超时或手动终止）:",
        err.message
      );
    } else {
      console.error("  [loadCameraImage] 加载摄像头图像失败：", err);
    }

    // 9. 加载失败时，清除 imageUrl，可触发 UI 显示“暂无图像”
    imageUrl.value = ""; // 不显示图片

    // 10. 调用你的错误处理逻辑，比如显示 noImg 提示
    onImageError(err);
  } finally {
    // 11. 清理 controller
    controller = null;
  }
};

// 图片加载完成后的处理
const onImageLoad = () => {
  imageElement.value.style.display = "block";
  if (!imageElement.value) return;
  const img = imageElement.value;

  noImg.value = false;

  // 1. 获取原始宽高
  imageNaturalWidth = img.naturalWidth;
  imageNaturalHeight = img.naturalHeight;

  console.log("原始图片尺寸:", imageNaturalWidth, "x", imageNaturalHeight);

  // 2. 计算最大允许缩放比例
  const maxWidth = 400;
  const maxHeight = 600;

  const scaleByWidth = maxWidth / imageNaturalWidth;
  const scaleByHeight = maxHeight / imageNaturalHeight;

  // 3. 取最小缩放比例
  const scale = Math.min(scaleByWidth, scaleByHeight);

  // 4. 计算最终显示尺寸
  imageDisplayWidth = imageNaturalWidth * scale;
  imageDisplayHeight = imageNaturalHeight * scale;

  console.log("约束后显示尺寸:", imageDisplayWidth, "x", imageDisplayHeight);

  // 5. 设置图片的显示尺寸（关键！）
  img.style.width = `${imageDisplayWidth}px`;
  img.style.height = `${imageDisplayHeight}px`;

  // 6. 设置 canvas 尺寸
  if (canvasEl.value) {
    canvasEl.value.width = imageDisplayWidth;
    canvasEl.value.height = imageDisplayHeight;
    canvasEl.value.style.width = `${imageDisplayWidth}px`;
    canvasEl.value.style.height = `${imageDisplayHeight}px`;
  }

  // 7. 初始化 Fabric
  initFabricCanvas();
  isDrawing.value = false;
  isImageReady.value = true;
  getImageInfo();
};

//图片加载失败
const onImageError = (event) => {
  console.error("❌ 图片加载失败！", event);
  valueCamera.value = null;

  isDrawing.value = false;
  noImg.value = true;
  isImageReady.value = false; // 加载失败也不允许截图
  imageElement.value.style.display = "none";

  //  如果 canvas 已经初始化 则禁用它
  if (canvas) {
    console.log("图片加载失败，进入了canvas");
    if (canvas) {
      canvas.dispose(); //  彻底销毁 fabric canvas 实例以及其关联的 DOM 元素
      canvas = null; // 清除引用
    }
  }
};

// 初始化 Fabric Canvas
const initFabricCanvas = () => {
  if (!canvasEl.value) return;

  if (canvas) {
    canvas.dispose(); // 销毁旧的，避免重复
  }

  canvas = new fabric.Canvas(canvasEl.value, {
    width: imageDisplayWidth,
    height: imageDisplayHeight,
    backgroundColor: "transparent", // 可选
  });

  console.log(canvasEl.value);

  console.log(
    "✅ Fabric Canvas 已初始化，尺寸:",
    imageDisplayWidth,
    "x",
    imageDisplayHeight
  );
  canvas.on("mouse:down", (opt) => {
    handleArrowDragMouseDown(opt);
    handleRectangleDragMouseDown(opt);
    handleCircleDragMouseDown(opt);
    handleEllipseDragMouseDown(opt);
    handleTriangleDragMouseDown(opt);
    handleCanvasMouseDown(opt);
  });
  canvas.on("mouse:move", (opt) => {
    handleArrowDragMouseMove(opt);
    handleRectangleDragMouseMove(opt);
    handleCircleDragMouseMove(opt);
    handleEllipseDragMouseMove(opt);
    handleTriangleDragMouseMove(opt);
  });
  canvas.on("mouse:up", (opt) => {
    handleArrowDragMouseUp(opt);
    handleRectangleDragMouseUp(opt);
    handleCircleDragMouseUp(opt);
    handleEllipseDragMouseUp(opt);
    handleTriangleDragMouseUp(opt);
  });
};

watch(valueCamera, () => {
  console.log("检测到相机切换，重新加载图像");
  imageCounter.value += 1; // 避免缓存
  loadCameraImage(); // 重新拉取图像
});

// 初始化画布
onMounted(() => {
  getCameraList();
  console.log(window.location.origin);
  baseUrl.value = window.location.origin;
  // fetchImage().then((res) => {});
  // canvas = new fabric.Canvas(canvasEl.value, { width: 600, height: 400 });
  // setArrowDragMode(false);
  // fabric.Image.fromURL(bgImage, (img) => {
  //   if (!img) return console.error("背景图加载失败");
  //   img.set({
  //     scaleX: canvas.width / img.width,
  //     scaleY: canvas.height / img.height,
  //     selectable: false,
  //     evented: false,
  //   });
  //   canvas.setBackgroundImage(img, canvas.renderAll.bind(canvas));
  // });
  // canvas.on("mouse:down", (opt) => {
  //   handleArrowDragMouseDown(opt);
  //   handleRectangleDragMouseDown(opt);
  //   handleCircleDragMouseDown(opt);
  //   handleEllipseDragMouseDown(opt);
  //   handleTriangleDragMouseDown(opt);
  //   handleCanvasMouseDown(opt);
  // });
  // canvas.on("mouse:move", (opt) => {
  //   handleArrowDragMouseMove(opt);
  //   handleRectangleDragMouseMove(opt);
  //   handleCircleDragMouseMove(opt);
  //   handleEllipseDragMouseMove(opt);
  //   handleTriangleDragMouseMove(opt);
  // });
  // canvas.on("mouse:up", (opt) => {
  //   handleArrowDragMouseUp(opt);
  //   handleRectangleDragMouseUp(opt);
  //   handleCircleDragMouseUp(opt);
  //   handleEllipseDragMouseUp(opt);
  //   handleTriangleDragMouseUp(opt);
  // });
});

onBeforeUnmount(() => {
  if (controller) {
    controller.abort();
    controller = null;
  }

  if (imageUrl.value && imageUrl.value.startsWith("blob:")) {
    console.log("组件销毁，释放 ObjectURL:", imageUrl.value);
    URL.revokeObjectURL(imageUrl.value);
  }
});
</script>

<style lang="scss" scoped>
.img-container {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;

  height: 100vh;
  width: 49vw;
  border: 1px solid pink;
  box-sizing: border-box;

  // background-color: red;;

  .img-wrapper {
    // border: 3px solid red; // 可视化边界（调试用，可删）
    position: relative;
    width: 600px;
  }

  .norem-img-content {
    z-index: 1;
    // width: 100%; // 宽度填满父容器
    // height: 100%; // 高度按比例自适应
    position: absolute;
    top: 0;
    left: 0;
    max-width: 400px;
    width: auto;
    height: auto;
    display: block;
  }

  .fabric-canvas {
    position: absolute;
    // z-index: 100 !important;
    top: 0;
    left: 0;
  }

  .toolbar {
    width: 46vw;
    height: 100px;
    padding: 5px 20px;
    display: flex;
    flex-direction: row;
    align-items: center;
    box-sizing: border-box;
    background: #f8f9fa;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    margin-bottom: 10px;

    .free-content {
      // border: 1px solid red;
      width: 200px;
      height: 80px;
      display: flex;
      flex-direction: column;
      align-items: center;

      .btn-free-pick {
        // border: 1px solid red;
        display: flex;
        flex-direction: row;
        justify-content: flex-start;
        align-items: center;
        height: 40px;
        width: 100%;
      }

      .btn-free {
        width: 100px;
        height: 40px;
        line-height: 40px;
        font-size: 18px;
        cursor: pointer;
      }

      /* 颜色选择器美化 */
      .color-picker {
        width: 80px;
        height: 40px;
        border: none;
        border-radius: 6px;
        cursor: pointer;
        // box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
      }

      .free-size {
        // border: 1px solid red;
        height: 40px;
        display: flex;
        flex-direction: row;
        width: 100%;
        justify-content: space-between;
        align-items: center;
        span {
          width: 100px;
          display: inline-block;
          font-size: 16px;
        }
      }

      /* 滑块美化 */
      .brush-slider {
        // border: 1px solid red;
        width: 100px;
        height: 6px;
        border-radius: 3px;
        background: #ddd;
        outline: none;
        cursor: pointer;

        &::-webkit-slider-thumb {
          appearance: none;
          width: 18px;
          height: 18px;
          border-radius: 50%;
          background: #007bff;
          cursor: pointer;
          box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
        }

        &::-moz-range-thumb {
          width: 18px;
          height: 18px;
          border-radius: 50%;
          background: #007bff;
          cursor: pointer;
          border: none;
          box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
        }
      }
    }

    .tool-mid {
      margin: 0 10px;
      // border: 1px solid red;
      width: 320px;
      height: 80px;
      display: flex;
      flex-direction: row;
      align-items: center;
      justify-content: space-between;
      flex-wrap: wrap;
      button {
        cursor: pointer;
        line-height: 30px;
        font-size: 16px;
        width: 100px;
        height: 30px;
      }
    }

    .too-edit {
      // border: 1px solid red;
      width: 220px;
      height: 80px;
      display: flex;
      flex-direction: row;
      justify-content: space-between;
      align-items: center;
      flex-wrap: wrap;
      button {
        width: 100px;
        height: 30px;
        line-height: 30px;
        font-size: 16px;
      }
    }

    /* 特殊按钮样式（可选，你可以进一步定制颜色） */
    .export-btn {
      cursor: pointer;
      background: #28a745;
      color: white;
      border: 1px solid #28a745;
      border-radius: 6px;

      &:hover {
        background: #218838;
      }
    }

    .delete-btn {
      cursor: pointer;
      background: #dc3545;
      color: white;
      border: 1px solid #dc3545;
      border-radius: 6px;

      &:hover {
        background: #c82333;
      }
    }

    .save-btn {
      cursor: pointer;
      background: #17a2b8;
      color: white;
      border: 1px solid #17a2b8;
      border-radius: 6px;

      &:hover {
        background: #138496;
      }
    }

    .load-btn {
      cursor: pointer;
      background: #17a2b8;
      color: white;
      border: 1px solid #17a2b8;
      border-radius: 6px;

      &:hover {
        background: #138496;
      }
    }
  }
}

.flex-row {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.flex-col {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: flex-start;
}

.active {
  background-color: #007bff;
  color: #ffffff;
  border: 1px solid #ffffff;
  border-radius: 2px;
}

.noImg {
  width: 400px;
  height: 400px;
}

/* 确保 upper-canvas 在最上层 */
.fabric-canvas.upper-canvas {
  z-index: 100 !important;
}

/* 确保 lower-canvas 在下层 */
.fabric-canvas.lower-canvas {
  z-index: 1 !important;
}

.camera-wrapper {
  width: 100%;
  box-sizing: border-box;
  // border: 1px solid red;
  padding: 20px 30px;
  .camera-border {
    // border: 1px solid red;

    background: #f8f9fa;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    padding: 0 20px;

    .camera-img-info {
      display: flex;
      flex-direction: row;
      justify-content: flex-start;
      align-items: center;
      height: 50px;
      // border: 1px solid blue;
      span {
        width: 40%;
        // border: 1px solid red;
        text-align: left;
        height: 50px;
        line-height: 50px;
        font-size: 16px;
      }

      .blue {
        color: #17a2b8;
      }
    }
    .camera-local-info {
      display: flex;
      flex-direction: row;
      justify-content: flex-start;
      align-items: center;
      height: 50px;
      // border: 1px solid blue;
      span {
        // border: 1px solid red;
        text-align: left;
        height: 50px;
        line-height: 50px;
        font-size: 16px;
      }
    }
  }
}
</style>

<style>
/* 确保 upper-canvas 在最上层 */
.fabric-canvas.upper-canvas {
  z-index: 100 !important;
}

/* 确保 lower-canvas 在下层 */
.fabric-canvas.lower-canvas {
  z-index: 1 !important;
}
</style>
