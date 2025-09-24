<template>
  <div class="file-tree">
    <!-- #45A9F9 -->

    <div v-if="items.length === 0" class="open-btn">
      <el-button type="primary" @click="selectFolder">
        测试选择文件夹
      </el-button>
    </div>

    <div v-if="items.length > 0" class="tree-container">
      <!-- @onSelect="onItemSelected" -->
      <vue3-tree-vue
        :items="items"
        :isCheckable="false"
        :hideGuideLines="false"
        @onCheck="onItemChecked"
        @onExpand="onItemExpanded"
        @dropValidator="onBeforeItemDropped"
        class="norem-tree"
      >
        <!-- 用 FileIcon 组件渲染图标 -->
        <template v-slot:item-prepend-icon="treeItem">
          <FileIcon
            :fileName="treeItem.name || 'default_file'"
            :fileType="treeItem.type || 'file'"
            :isOpen="treeItem.expanded || false"
            :size="20"
          />
        </template>
        <template v-slot:item-name="treeItem">
          <span
            @click="handleClick(treeItem)"
            @dblclick="handleDoubleClick(treeItem)"
            class="tree-node-name"
          >
            {{ treeItem.name }}
          </span>
        </template>
      </vue3-tree-vue>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";

import FileIcon from "@/components/FileIcon.vue";
import Vue3TreeVue from "vue3-tree-vue";
import "vue3-tree-vue/dist/style.css";

const items = ref([]);
const emits = defineEmits(["fileSelected"]);
const selectedFileName = ref("");
const selectedFileContent = ref("");
// 防抖变量
const clickTimer = ref(null);
const isDoubleClick = ref(false);
// 构建树结构（
const buildTreeStructure = (files) => {
  const root = { name: "root", children: [] };

  files.forEach((file) => {
    const pathParts = file.webkitRelativePath.split("/").filter((part) => part);
    let current = root;

    pathParts.forEach((part, index) => {
      const isLast = index === pathParts.length - 1;
      const existing = current.children?.find((child) => child.name === part);

      if (!existing) {
        const ext = isLast ? part.split(".").pop().toLowerCase() : "";
        const isFirstLevel = current.name === "root";

        const newNode = {
          name: part,
          path: pathParts.slice(0, index + 1).join("/"),
          isDirectory: !isLast,
          file: isLast ? file : null,
          type: !isLast ? "folder" : "file",
          ext: ext,
          children: isLast ? undefined : [],
          expanded: isFirstLevel && !isLast ? true : false,
        };
        current.children?.push(newNode);
        current = newNode;
      } else {
        current = existing;
      }
    });
  });

  return root.children || [];
};

// 选择文件夹
const selectFolder = () => {
  const input = document.createElement("input");
  input.type = "file";
  input.webkitdirectory = true;
  input.onchange = (e) => {
    const target = e.target;
    if (target && target.files) {
      const files = Array.from(target.files);
      const tree = buildTreeStructure(files);
      items.value = tree;
    }
  };
  input.click();
};

// 选中文件(原来逻辑，备份)
const onItemSelected = (item) => {
  if (item.type === "file" && item.file) {
    selectedFileName.value = item.name;
    const file = item.file;

    const reader = new FileReader();
    reader.readAsText(file);

    reader.onload = (event) => {
      selectedFileContent.value = event.target.result;
      emits("fileSelected", {
        name: selectedFileName.value,
        content: selectedFileContent.value,
      });
    };

    reader.onerror = () => {
      selectedFileContent.value = `无法读取文件：${reader.error.message}`;
    };
  } else {
    selectedFileName.value = "";
    selectedFileContent.value = "";
  }
};

// 处理点击（单击）
const handleClick = (treeItem) => {
  if (isDoubleClick.value) {
    // 如果是双击，则忽略单击
    isDoubleClick.value = false; //  重置 isDoubleClick
    clearTimeout(clickTimer.value); //  清除残留的定时器
    return;
  }

  // 300ms 后判断是否是双击
  clickTimer.value = setTimeout(() => {
    if (!isDoubleClick.value) {
      // 300ms 内没有第二次点击，执行单击逻辑
      onSingleClick(treeItem);
    }
    clickTimer.value = null; //  清除定时器引用
    isDoubleClick.value = false; //  重置 isDoubleClick
  }, 300);
};

// 处理双击
// 处理双击
const handleDoubleClick = (treeItem) => {
  clearTimeout(clickTimer.value); // 清除单击的定时器
  isDoubleClick.value = true; // 标记为双击
  onDoubleClick(treeItem); // 执行双击逻辑

  //  双击后，确保下一次单击能正常执行
  setTimeout(() => {
    isDoubleClick.value = false; // 300ms 后重置 isDoubleClick
  }, 300);
};

// 单击逻辑（原来 onItemSelected 的逻辑）
const onSingleClick = (item) => {
  console.log("单击:", item);

  if (item.type === "file" && item.file) {
    // 单击文件：读取文件内容
    selectedFileName.value = item.name;
    const file = item.file;

    const reader = new FileReader();
    reader.readAsText(file);

    reader.onload = (event) => {
      selectedFileContent.value = event.target.result;
      emits("fileSelected", {
        name: selectedFileName.value,
        content: selectedFileContent.value,
      });
    };

    reader.onerror = () => {
      selectedFileContent.value = `无法读取文件：${reader.error.message}`;
    };
  } else if (item.type === "folder") {
    // 单击文件夹：切换展开/折叠状态
    console.log("单击文件夹，切换展开/折叠状态");
    // item.expanded = !item.expanded;
    toggleFolderExpanded(item);
  }
};

const toggleFolderExpanded = (folderItem) => {
  // 递归查找并修改当前文件夹的 expanded 状态
  const updateExpanded = (items, targetId) => {
    for (const item of items) {
      if (item.id === targetId) {
        item.expanded = !item.expanded; // 切换展开状态
        return true; // 找到并修改成功
      }
      if (item.children) {
        const found = updateExpanded(item.children, targetId);
        if (found) return true;
      }
    }
    return false;
  };

  updateExpanded(items.value, folderItem.id);
};
// 双击逻辑（原来 onItemDoubleClick 的逻辑）
const onDoubleClick = (treeItem) => {
  console.log("双击:", treeItem);

  if (treeItem.type === "file" && treeItem.file) {
    // 双击文件：读取文件内容
    selectedFileName.value = treeItem.name;
    const file = treeItem.file;

    const reader = new FileReader();
    reader.readAsText(file);

    reader.onload = (event) => {
      selectedFileContent.value = event.target.result;
      emits("fileSelected", {
        name: selectedFileName.value,
        content: selectedFileContent.value,
      });
      // 你可以在这里额外执行双击的操作（比如打开编辑器）
    };

    reader.onerror = () => {
      selectedFileContent.value = `无法读取文件：${reader.error.message}`;
    };
  } else if (treeItem.type === "folder") {
    // 双击文件夹：不执行任何操作（直接忽略）
    console.log("双击文件夹，不做任何操作");
  }
};
const onItemChecked = (checkedItems) => {
  console.log("Checked:", checkedItems);
};

const onBeforeItemDropped = (droppedItem, destinationNode) => {
  return new Promise((resolve) => {
    resolve(droppedItem !== destinationNode);
  });
};

const onItemExpanded = (expandedItem) => {
  console.log("Expanded:", expandedItem);
};
</script>

<style scoped lang="scss">
.file-tree {
  width: 100%;
  height: 100%;
  overflow-x: hidden; /* 隐藏水平滚动条 */
  overflow-y: auto; /* 保留垂直滚动 */
  background-color: #242526;
}

/* 新增树容器样式，用于限制宽度 */
.tree-container {
  margin-top: 20px;
  width: 100%;
  box-sizing: border-box;
  padding-right: 8px; /* 预留一点空间避免紧贴边缘 */
}

.open-btn {
  display: flex;
  flex-direction: row;
  justify-content: center;
  margin-top: 50px;
}
</style>

<style lang="scss">
.tiny_horizontal_margin {
  height: 30px !important;
  // border:1px solid red;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.norem-tree {
  .chevron-right {
    display: inline-block;
    // border: 1px solid blue;
    // width: 20px;
  }
  .tree-item-node-parent {
    width: 100%;
  }

  // 确保树组件宽度继承父容器
  width: 100%;

  .tree-item {
    // 限制每个树节点的最大宽度
    width: 100%;
    // width: 300px !important;
    // border: 1px solid green;
    .d-flex {
      display: flex !important;
      flex-direction: row !important;
      justify-content: flex-start !important;
      align-items: center !important;
      width: 100%;
      height: 30px;
      // border:1px solid yellow;

      // 图标容器 - 固定宽度
      .item-prepend-icon {
        flex: 0 0 24px; /* 固定宽度，不伸缩 */
        margin-right: 6px;
      }

      // 文件名容器 - 自适应剩余宽度
      span {
        display: inline-block;
        height: 30px;
        line-height: 30px;
        font-size: 16px;
        flex: 1; // 占满剩余空间
        min-width: 0; // 允许被压缩到比内容还小
        overflow: hidden; // 隐藏溢出部分
        white-space: nowrap; // 禁止换行，保证一行显示
        text-overflow: ellipsis; // 超出显示 ...
        // border: 1px solid red; // 可删除，调试用
        color: #b7b7b7;
      }
    }
  }
}

.vue3-tree-vue .guide-line {
  width: 12px;
  min-width: 12px !important;
  // border: 1px solid red;
}

.vue3-tree-vue .chevron-right {
  color: #b7b7b7;
}

//scroll
.file-tree-dark {
  border: 4px solid red;

  /* 滚动条整体 */
  ::-webkit-scrollbar {
    width: 20px;
    height: 20px;
  }

  /* 滚动条轨道 */
  ::-webkit-scrollbar-track {
    background: #292a2b;
    border-radius: 0;
  }

  /* 滚动条滑块 */
  ::-webkit-scrollbar-thumb {
    background-color: #313438;
    border-radius: 0;
    border: 2px solid transparent;
    background-clip: content-box;
    transition: background-color 0.2s ease; /* 添加过渡效果 */
  }

  /* 滚动条滑块悬停状态 */
  ::-webkit-scrollbar-thumb:hover {
    background-color: #757575 !important;
    cursor: pointer;
  }

  /* 滚动条滑块激活状态（点击时） */
  ::-webkit-scrollbar-thumb:active {
    background-color: #757575 !important; /* 使用稍暗的红色区分激活状态 */
  }

  /* 滚动条角落 */
  ::-webkit-scrollbar-corner {
    background: #292a2b; /* 与轨道颜色保持一致 */
  }
}
</style>
