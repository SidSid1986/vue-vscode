<!--
 * @Author: Sid Li
 * @Date: 2025-09-18 11:07:27
 * @LastEditors: Sid Li
 * @LastEditTime: 2025-09-19 10:19:51
 * @FilePath: \vue-vscode-git\src\views\Home\home.vue
 * @Description: 
-->
<template>
  <div class="container" ref="containerRef">
    <div class="file-content">
      <FileTree @fileSelected="fileSelected" />
    </div>

    <div class="code-content">
      <!-- <MonacoCom
        ref="jsonComponents"
        :model-value="strJson"
        @update:model-value="handleChangeResponseJson"
      /> -->

      <Terminal />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import MonacoCom from "@/components/MonacoCom.vue";

import FileTree from "@/components/FileTree.vue";
import Terminal from "@/components/Terminal.vue";

const oneFileName = ref("");

const jsonComponents = ref(null);
// const strJson = ref("");
const strJson = ref(
  "# Python 示例代码\nprint('Hello Python!')\nresult = 1 + 2\nprint('计算结果：', result)"
);

const fileSelected = (file) => {
  console.log(file);
  oneFileName.value = file.name;
  strJson.value = file.content;
};

//更改返回json数据
const handleChangeResponseJson = () => {
  // 返回内容值，根据业务增加后面的逻辑
};

// 生命周期：添加/移除事件监听
onMounted(() => {});

onUnmounted(() => {});
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: row;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  justify-content: space-between;
}

.file-content {
  width: 25vw;
  height: 100vh;
  border: 1px solid green;
}
</style>
