<template>
  <div class="file-tree">
    <button @click="selectFolder">选择文件夹</button>

    <div v-if="items.length > 0">
      <vue3-tree-vue
        :items="items"
        :isCheckable="false"
        :hideGuideLines="false"
        @onCheck="onItemChecked"
        @onSelect="onItemSelected"
        @onExpand="onItemExpanded"
        @dropValidator="onBeforeItemDropped"
      >
        <!--   用 FileIcon 组件渲染图标 -->
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

import FileIcon from "@/components/FileIcon.vue"; // 导入自定义图标组件
import Vue3TreeVue from "vue3-tree-vue";
import "vue3-tree-vue/dist/style.css"; // ✅ 正确的官方样式文件

// 不再使用 TypeScript，items 就是一个普通数组
const items = ref([]); // 直接是数组，里面是对象，每个对象包含 text, type, children...
// 新增：保存选中的文件信息和内容

const emits = defineEmits(["fileSelected"]);
const selectedFileName = ref("");
const selectedFileContent = ref("");
// 构建树：将 File 对象数组转成树形结构
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
        // 关键：判断是否为第一层节点（root的直接子节点）
        const isFirstLevel = current.name === "root";

        const newNode = {
          name: part,
          path: pathParts.slice(0, index + 1).join("/"),
          isDirectory: !isLast,
          file: isLast ? file : null,
          type: !isLast ? "folder" : "file",
          ext: ext,
          children: isLast ? undefined : [],
          // 仅第一层文件夹默认展开，其他层级默认收起
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
const selectFolder = () => {
  const input = document.createElement("input");
  input.type = "file";
  input.webkitdirectory = true;
  input.onchange = (e) => {
    const target = e.target;
    if (target && target.files) {
      const files = Array.from(target.files); // FileList → Array<File>
      const tree = buildTreeStructure(files);
      items.value = tree;
    }
  };
  input.click();
};

//选中文件
const onItemSelected = (item) => {
  // 判断选中的是文件（非文件夹且有 File 对象）
  if (item.type === "file" && item.file) {
    selectedFileName.value = item.name; // 保存文件名
    const file = item.file; // 获取原始 File 对象

    // 用 FileReader 读取文件内容
    const reader = new FileReader();
    // 读取文本内容（适用于代码、文本文件）
    reader.readAsText(file);

    // 读取成功后更新内容
    reader.onload = (event) => {
      selectedFileContent.value = event.target.result;
      console.log(selectedFileContent.value);
      //传递给父组件
      emits("fileSelected", {
        name: selectedFileName.value,
        content: selectedFileContent.value,
      });
    };

    // 读取失败处理
    reader.onerror = () => {
      selectedFileContent.value = `无法读取文件：${reader.error.message}`;
    };
  } else {
    // 选中的是文件夹，清空内容
    selectedFileName.value = "";
    selectedFileContent.value = "";
  }
};

// 以下事件监听器你可以按需使用，目前只是占位打印
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
  border: 1px solid red;
  width: 100%;
  height: 100%;
  overflow: auto;
}
.node-icon {
  margin-right: 6px;
  font-size: 16px;
}
</style>

<style lang="scss">
.tiny_horizontal_margin {
  // border:1px solid blue;
  height: 20px !important;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.d-flex {
  display: flex !important;
  flex-direction: row !important;
  justify-content: flex-start !important;
  align-items: center !important;
  span {
    // border:1px solid blue;
    display: inline-block;
    height: 20px;
    line-height: 20px;
    font-size: 16px;
  }
}
</style>
