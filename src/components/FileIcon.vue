<template>
  <img
    :src="iconSrc"
    alt="File icon"
    class="file-icon" 
  />
</template>

<script setup lang="ts">
import { computed } from "vue";
import {
  getIconForFile,
  getIconForFolder,
  getIconForOpenFolder,
} from "vscode-icons-js";

const props = defineProps<{
  fileName: string;
  fileType: string;
  isOpen?: boolean;
  size?: number;
}>();

 
const iconSrc = computed(() => {
  let iconName: string = "";
  try {
    if (props.fileType === "folder") {
      iconName = props.isOpen
        ? getIconForOpenFolder(props.fileName) ?? "default_folder_opened.svg"
        : getIconForFolder(props.fileName) ?? "default_folder.svg";
    } else {
      iconName = getIconForFile(props.fileName) ?? "default_file.svg";
    }
    return new URL(`/icons/${iconName}`, window.location.origin).href;
  } catch (error) {
    iconName = props.fileType === "folder" ? "default_folder.svg" : "default_file.svg";
    console.error("图标处理异常:", error);
    return new URL(`/icons/${iconName}`, window.location.origin).href;
  }
});
</script>

<style scoped>
/* CSS 直接引用带单位的变量，无需拼接 */
.file-icon {
  /* 正确写法：直接用 v-bind 引用带单位的计算属性 */
  width: 20px;
  height: 20px;
  
  /* border: 1px solid red; */
  margin-right: 6px;
   
}
</style>