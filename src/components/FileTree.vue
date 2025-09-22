<template>
  <div class="file-tree">
    <button style="color: red" @click="selectFolder">测试选择文件夹</button>

    <div v-if="items.length > 0" class="tree-container">
      <vue3-tree-vue
        :items="items"
        :isCheckable="false"
        :hideGuideLines="false"
        @onCheck="onItemChecked"
        @onSelect="onItemSelected"
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

// 构建树结构（保持不变）
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

// 选择文件夹（保持不变）
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

// 选中文件（保持不变）
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

<style scoped>
.file-tree {
  width: 100%;
  height: 100%;
  overflow-x: hidden; /* 隐藏水平滚动条 */
  overflow-y: auto; /* 保留垂直滚动 */
  background-color: #1e1e1e;
}

/* 新增树容器样式，用于限制宽度 */
.tree-container {
  width: 100%;
  box-sizing: border-box;
  padding-right: 8px; /* 预留一点空间避免紧贴边缘 */
}
</style>

<style lang="scss">
.tiny_horizontal_margin {
  height: 20px !important;
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

      // 图标容器 - 固定宽度
      .item-prepend-icon {
        flex: 0 0 24px; /* 固定宽度，不伸缩 */
        margin-right: 6px;
      }

      // 文件名容器 - 自适应剩余宽度
      span {
        display: inline-block;
        height: 20px;
        line-height: 20px;
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
</style>
