<template>
  <!-- 最外层布局容器，100vw / 100vh，flex column -->
  <div class="app-wrapper">
    <!--  顶部工具栏：黄色，固定 50px -->
    <div class="global-toolbar">
      顶部工具栏{{ leftPanelWidth }}--{{ rightPanelWidth }}
    </div>

    <!--  container（占满剩余所有高度） -->
    <div class="container" ref="containerRef">
      <!-- 左侧粉色区域（文件操作图标区） -->
      <div class="left-icon-area">
        <i class="iconfont icon-icon-test1" @click="openFileContent"></i>
        <i class="iconfont icon-icon-test"></i>
      </div>

      <!-- 左侧文件树 -->
      <div
        class="file-content"
        ref="fileContentRef"
        :style="{ width: leftPanelWidth }"
      >
        <FileTree @fileSelected="fileSelected" />
      </div>

      <!--  左右拖拽手柄 -->
      <div
        class="drag-container-horizontal"
        @mousedown="startHorizontalDrag"
        :class="{ dragging: isHorizontalDragging }"
      ></div>

      <!--  右侧代码 + Terminal 区域 -->
      <div
        class="code-content"
        ref="codeContentRef"
        :style="{ width: rightPanelWidth }"
      >
        <!-- 上方：代码展示区，高度动态变化 -->
        <div class="editor-content" :style="{ height: editorHeight }">
          <MonacoCom
            ref="jsonComponents"
            :model-value="strJson"
            @update:model-value="handleChangeResponseJson"
          />
        </div>

        <!-- 中间：拖拽手柄 -->
        <div class="drag-container">
          <div
            class="resize-handle"
            @mousedown="startDrag"
            :class="{ dragging: isDragging }"
          ></div>

          <!-- 下方：Terminal 组件，高度动态变化 -->
          <div class="terminal-container" :style="{ height: terminalHeight }">
            <Terminal
              :terminalContentHeight="terminalContentHeight"
              @closeTerminalFunc="closeTerminalFunc"
            />
          </div>
        </div>
      </div>
    </div>

    <!-- ✅ 3. 新增：底部工具栏：蓝色，固定 30px -->
    <div class="global-status-bar">
      🧩 底部工具栏（蓝色，比如状态/日志/Git/行号）
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, computed } from "vue";
import FileTree from "@/components/FileTree.vue";
import MonacoCom from "@/components/MonacoCom.vue";
import Terminal from "@/components/Terminal.vue";

//  （leftPanel + rightPanel）总宽度 = 100 - 3 - 0.4 = 96.6vw
const DYNAMIC_TOTAL_WIDTH_VW = 96.6;

//  文件内容
const oneFileName = ref("");
const strJson = ref(
  "# Python 示例代码\nprint('Hello Python!')\nresult = 1 + 2\nprint('计算结果：', result)"
);

// 高度相关
const codeContentRef = ref(null);
const editorHeight = ref(""); // 动态高度，单位 px
const terminalHeight = ref(""); // 动态高度，单位 px
const terminalContentHeight = ref("");
const isDragging = ref(false);

// 拖拽相关变量（垂直拖拽）
const startY = ref(0);
const startTerminalHeight = ref(0); // 拖拽起始时 terminal 高度（px）

// 宽度相关
const containerRef = ref(null);
const fileContentRef = ref(null);
const leftPanelWidth = ref("15vw"); // 左侧面板宽度
const rightPanelWidth = ref("81.6vw"); // 右侧面板宽度
const isHorizontalDragging = ref(false);

// 水平拖拽相关变量
const startX = ref(0); //鼠标按下时的 X 坐标
const startLeftWidthVw = ref(0); //拖拽开始时，左侧面板的宽度

// 文件选择  ========
const fileSelected = (file) => {
  console.log("File selected:", file);
  oneFileName.value = file.name;
  strJson.value = file.content;
};

// 开始垂直拖拽（编辑器和终端之间的拖拽）
const startDrag = (e) => {
  if (!codeContentRef.value) return;

  isDragging.value = true;

  const container = codeContentRef.value;
  const containerHeight = container.clientHeight; // 容器总高度（px）

  startY.value = e.clientY;
  startTerminalHeight.value = parseHeight(
    terminalHeight.value,
    containerHeight
  );
  document.body.style.cursor = "ns-resize";
  document.addEventListener("mousemove", handleMouseMove);
  document.addEventListener("mouseup", stopDrag);

  e.preventDefault(); // 防止文本选中
};

// 垂直拖拽中：计算新的 Terminal 和 Editor 高度
const handleMouseMove = (e) => {
  if (!isDragging.value) return;

  const container = codeContentRef.value;
  if (!container) return;

  const containerHeight = container.clientHeight;
  const deltaY = startY.value - e.clientY; //  startY - e.clientY

  // newTerminalHeight = 初始 Terminal 高度 + deltaY
  // newEditorHeight = 初始总高度 - newTerminalHeight
  let newTerminalHeightPx = startTerminalHeight.value + deltaY;

  // 容器总高度
  const totalHeight = containerHeight;

  // 限制最小高度
  const minTerminalHeight = 100;
  const minEditorHeight = 100;

  newTerminalHeightPx = Math.max(
    minTerminalHeight,
    Math.min(newTerminalHeightPx, totalHeight - minEditorHeight)
  );

  const newEditorHeightPx = totalHeight - newTerminalHeightPx;

  // 设置高度（px）
  terminalHeight.value = `${newTerminalHeightPx}px`;

  terminalContentHeight.value = `${newTerminalHeightPx - 40}px`;
  editorHeight.value = `${newEditorHeightPx}px`;
};

// 停止垂直拖拽
const stopDrag = () => {
  isDragging.value = false;
  document.body.style.cursor = "";

  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopDrag);
};

// 开始水平拖拽（左右面板的拖拽）
const startHorizontalDrag = (e) => {
  // 如果 containerRef（即 .container）还未挂载或不存在，直接退出函数，避免报错
  if (!containerRef.value) return;

  // 标记当前正在水平拖拽中，用于控制状态和样式
  isHorizontalDragging.value = true;

  // 获取容器 DOM 元素（即 .container，也就是整体 100vw 的父容器）
  const container = containerRef.value;

  // 获取容器的 clientWidth（当前视口下容器的实际像素宽度，用于后续计算比例）
  const containerWidth = container.clientWidth;

  // 记录鼠标按下的 X 坐标，用于后续计算鼠标移动的距离（deltaX）
  startX.value = e.clientX;

  // ✅ 解析当前 leftPanelWidth 的值（比如 "15vw"）→ 转为数字 15
  // 这个函数会根据 "15vw" 这种格式，提取出数字部分 15，用作拖拽的起始宽度
  startLeftWidthVw.value = parseWidth(leftPanelWidth.value, containerWidth);

  // 设置鼠标样式为 ew-resize（左右箭头），提示用户当前可以左右拖拽
  document.body.style.cursor = "ew-resize";

  // 添加鼠标移动事件监听：拖拽过程中会持续触发 onHorizontalMouseMove
  document.addEventListener("mousemove", onHorizontalMouseMove);

  // 添加鼠标释放事件监听：鼠标松开时触发 stopHorizontalDrag，结束拖拽
  document.addEventListener("mouseup", stopHorizontalDrag);

  // 阻止默认行为，比如避免拖拽时选中文本或触发其他浏览器默认拖拽行为
  e.preventDefault();
};
// 水平拖拽中：计算新的左右面板宽度
const onHorizontalMouseMove = (e) => {
  // 如果当前没有在拖拽中（比如鼠标移出后误触发），直接退出，不做任何处理
  if (!isHorizontalDragging.value) return;

  // 计算鼠标从按下到当前移动了多少像素（deltaX > 0 表示向右，deltaX < 0 表示向左）
  const deltaX = e.clientX - startX.value;

  // 再次获取容器，以及它的实际宽度（像素），用于将鼠标移动距离换算成百分比宽度
  const container = containerRef.value;
  const containerWidth = container.clientWidth;

  //  计算新的左侧面板宽度（newLeftWidthVw）：
  // 从起始宽度 startLeftWidthVw.value（比如 15）开始，
  // 加上鼠标移动距离 deltaX 所对应的百分比变化：
  //   (deltaX / containerWidth) * 100 → 是鼠标横向移动的百分比
  let newLeftWidthVw = startLeftWidthVw.value + (deltaX / containerWidth) * 100;

  //  限制 newLeftWidthVw 的范围在 [0, 96.6] 之间：
  // 为什么是 96.6？
  // 因为整个容器是 100vw，但其中 3vw 是左侧图标，0.4vw 是拖拽把手 → 剩下动态分配的宽度只有 96.6vw
  // 所以 leftPanelWidth 最小可以是 0vw，最大只能是 96.6vw
  newLeftWidthVw = Math.max(0, Math.min(96.6, newLeftWidthVw));

  //  更新左侧面板宽度（动态部分）：使用 vw 单位，保留 4 位小数避免频繁触发更新
  leftPanelWidth.value = `${newLeftWidthVw.toFixed(4)}vw`;

  //   更新右侧面板宽度（动态部分）：
  // 因为 left + right 必须严格等于 96.6vw，所以：
  // right = 96.6 - left
  rightPanelWidth.value = `${(DYNAMIC_TOTAL_WIDTH_VW - newLeftWidthVw).toFixed(
    4
  )}vw`;
};
// 停止水平拖拽
const stopHorizontalDrag = () => {
  // 标记拖拽状态结束，用于 UI 状态控制和后续逻辑判断
  isHorizontalDragging.value = false;

  // 恢复鼠标指针为默认样式，不再显示 ew-resize（左右箭头）
  document.body.style.cursor = "";

  // 移除鼠标移动事件监听，避免拖拽结束后仍持续响应 mousemove
  document.removeEventListener("mousemove", onHorizontalMouseMove);

  // 移除鼠标释放事件监听（其实这一步可以省略，因为 mouseup 只触发一次，但为了规范，保留它）
  document.removeEventListener("mouseup", stopHorizontalDrag);
};

// 转为 px 数字（高度）
const parseHeight = (heightStr, containerHeight) => {
  if (heightStr.endsWith("px")) {
    return parseFloat(heightStr);
  } else if (heightStr.endsWith("vh")) {
    return (parseFloat(heightStr) / 100) * containerHeight;
  }
  return containerHeight / 2; // 默认值，备用
};

// 转为数值（宽度，vw单位）
const parseWidth = (widthStr, containerWidth) => {
  if (widthStr.endsWith("vw")) {
    return parseFloat(widthStr);
  } else if (widthStr.endsWith("px")) {
    return (parseFloat(widthStr) / containerWidth) * 100;
  }
  return 25; // 默认25%
};

const closeTerminalFunc = () => {
  terminalHeight.value = 0;
  editorHeight.value = "100%";
};

const openFileContent = () => {
  leftPanelWidth.value == "15vw"
    ? (leftPanelWidth.value = "0vw")
    : (leftPanelWidth.value = "15vw");
  if (leftPanelWidth.value == "0vw") {
    rightPanelWidth.value = "96.6vw";
  } else if (leftPanelWidth.value == "15vw") {
    rightPanelWidth.value = "81.6vw";
  }
};

const handleChangeResponseJson = () => {
  // 返回内容值，根据业务增加
};

onMounted(() => {
  nextTick(() => {
    // 初始化高度
    const containerHeight = codeContentRef.value?.clientHeight || 600;
    const initialTerminalHeight = containerHeight * 0.3; // 30%
    const initialEditorHeight = containerHeight - initialTerminalHeight;

    terminalHeight.value = `${initialTerminalHeight}px`;
    terminalContentHeight.value = `${initialTerminalHeight - 40}px`;
    editorHeight.value = `${initialEditorHeight}px`;
  });
});

onUnmounted(() => {
  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopDrag);
  document.removeEventListener("mousemove", handleHorizontalMouseMove);
  document.removeEventListener("mouseup", stopHorizontalDrag);
});
</script>

<style scoped lang="scss">
/* 最外层：100vw / 100vh，column 布局 */
.app-wrapper {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: sans-serif;
  overflow: hidden;
}

/* 1. 顶部工具栏：黄色，固定 50px */
.global-toolbar {
  height: 50px;
  background-color: blue;
  color: #ffffff;
  display: flex;
  align-items: center;
  padding: 0 16px;
  font-weight: bold;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  z-index: 1000;
}

/* 2. 你原来的 .container，占满剩余高度 */
.container {
  display: flex;
  flex-direction: row;
  width: 100%;
  height: 100%; /* 占满除去顶部和底部之后的所有高度 */
  overflow: hidden;
  user-select: none;
  -webkit-user-select: none;
}

/* 3. 你新增的：左侧粉色区域（文件操作图标区） */
.left-icon-area {
  width: 3vw;
  background-color: #222223;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  padding: 10px 0;

  i {
    color: #707071;
    font-size: 30px;
    margin-bottom: 30px;
    cursor: pointer;
    font-weight: bold;
  }
  i:hover {
    color: #e6e6e6;
  }
}

/* 以下是你原来的内部样式，全部原样保留，未改动 */
.file-content {
  height: 100%;
  overflow-y: auto;
  overflow-x: hidden;
}

.code-content {
  height: 100%;
  position: relative;
  overflow: hidden;
}

.editor-content {
  width: 100%;
  overflow: auto;
  box-sizing: border-box;
  background-color: #292a2b;
}

.drag-container-horizontal {
  width: 0.4vw;
  background-color: #242526;
  cursor: ew-resize;
  user-select: none;
  position: relative;
  z-index: 20;
  transition: background-color 0.2s;
  flex-shrink: 0;
  height: 100%;
  align-self: stretch;
}

.drag-container-horizontal:hover {
  background-color: #007fd4;
}

.drag-container-horizontal.dragging {
  background-color: #007fd4;
  width: 0.4vw;
}

.drag-container {
  width: 100%;
  position: absolute;
  bottom: 0;
}

.terminal-container {
  width: 100%;
  box-sizing: border-box;
  overflow: hidden;
  background-color: #292a2b;
}

.resize-handle {
  height: 8px;
  background-color: transparent;
  cursor: ns-resize;
  user-select: none;
  transition: background-color 0.2s;
  flex-shrink: 0;
  position: relative;
  z-index: 10;
  border: 1px solid transparent;
  box-sizing: border-box;
}

.resize-handle:hover {
  background-color: #007fd4;
  border: 1px solid transparent;
}

.resize-handle.dragging {
  background-color: #007fd4;
  border: 1px solid transparent;
  height: 8px;
}

* {
  -webkit-user-drag: none;
  touch-action: none;
}

/* ✅ 4. 新增：底部工具栏：蓝色，固定 30px */
.global-status-bar {
  height: 30px;
  background-color: blue; /* 你要求的蓝色 */
  color: white;
  display: flex;
  align-items: center;
  padding: 0 16px;
  font-size: 12px;
  z-index: 1000;
}
</style>
