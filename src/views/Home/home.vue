<!--
 * @Author: Sid Li
 * @Date: 2025-09-18 11:07:27
 * @LastEditors: Sid Li
 * @LastEditTime: 2025-09-18 16:08:05
 * @FilePath: \vue-Monaco-Editor\src\views\Home\home.vue
 * @Description: 
-->
<template>
  <div class="container" ref="containerRef">
    <MonacoCom
      ref="jsonComponents"
      :model-value="strJson"
      @update:model-value="handleChangeResponseJson"
    />
    <!-- <MonacoCom
      ref="jsonComponents"
      :model-value="strJson"
      @update:model-value="handleChangeResponseJson"
      language="python"
    /> -->
  </div>
</template>

<script setup>
import MonacoCom from "@/components/MonacoCom.vue";
import { ref, onMounted, onUnmounted } from "vue";

const jsonComponents = ref(null);
// const strJson = ref("");
const strJson = ref("# Python 示例代码\nprint('Hello Python!')\nresult = 1 + 2\nprint('计算结果：', result)");
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
</style>
