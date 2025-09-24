<!--
 * @Author: Sid Li
 * @Date: 2025-09-24 09:32:39
 * @LastEditors: Sid Li
 * @LastEditTime: 2025-09-24 13:56:10
 * @FilePath: \vue-vscode-git\src\components\ToolMenu.vue
 * @Description: 
-->
<template>
  <div class="menu-container" ref="menuContainerRef">
    <img class="logo" src="@/assets/free-logo.png" alt="" />
    <div class="menu-item" v-for="item in menuData" :key="item.id">
      <span
        @click="menuItemClick(item)"
        @mouseenter="handleMenuHover(item)"
        :class="activeMenuId === item.id ? 'menu-active' : 'menu-name'"
      >
        {{ item.name }}
      </span>

      <div v-if="activeMenuId === item.id" class="menu-child-content">
        <div class="menu-child" v-for="child in item.children" :key="child.id">
          <span class="menu-child-name">{{ child.name }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";

const menuContainerRef = ref(null);

// const props = defineProps({
//   menuData: Array,
//   default: () => [
//     {
//       id: 1,
//       name: "文件",
//       show: false,
//       children: [
//         {
//           id: 11,
//           name: "打开文件夹",
//           children: [],
//         },
//         {
//           id: 12,
//           name: "打开文件",
//           children: [],
//         },
//       ],
//     },
//     {
//       id: 2,
//       name: "编辑",
//       show: false,
//       children: [
//         {
//           id: 22,
//           name: "查找",
//           children: [],
//         },
//       ],
//     },
//   ],
// });

// 菜单数据
const menuData = ref([
  {
    id: 1,
    name: "文件",
    children: [
      { id: 11, name: "打开文件夹" },
      { id: 12, name: "打开文件" },
    ],
  },
  {
    id: 2,
    name: "编辑",
    children: [{ id: 22, name: "查找" }],
  },
]);

// 当前激活（打开）的菜单项 ID
const activeMenuId = ref(null);

// 点击菜单名称的处理函数
const menuItemClick = (item) => {
  if (activeMenuId.value === item.id) {
    console.log(1);
    // 如果点击的是当前已打开的菜单，则关闭它
    activeMenuId.value = null;
  } else {
    console.log(2);
    // 否则，关闭之前打开的（如果有），然后打开当前的
    activeMenuId.value = item.id;
  }
};

const handleMenuHover = (item) => {
  if (activeMenuId.value !== null) {
    // 只有当前已有菜单打开，hover 才触发切换
    activeMenuId.value = item.id;
  }
};

const handleClickOutside = (event) => {
  if (
    menuContainerRef.value &&
    !menuContainerRef.value.contains(event.target)
  ) {
    // 如果点击的目标不在 menu-container 内
    activeMenuId.value = null; // 关闭当前菜单
  }
};

onMounted(() => {
  document.addEventListener("click", handleClickOutside);
});

onUnmounted(() => {
  document.removeEventListener("click", handleClickOutside);
});
</script>

<style scoped lang="scss">
.menu-container {
  width: 100%;
  height: 100%;
  // background-color: red;
  color: #acacac;
  display: flex;
  flex-direction: row;
  justify-content: flex-start;
  align-items: center;
  .logo {
    width: 100px;
    height: 40px;
    background-color: #eeeeee;
    border-radius: 8px;
    margin-right: 30px;
  }
  .menu-item {
    // background-color: red;
    position: relative;
    .menu-name {
      display: block;
      height: 30px;
      line-height: 30px;
      padding: 0 20px;
      background-color: #242526;
      margin-right: 10px;
      border-radius: 8px;
      font-size: 18px;
      cursor: pointer;
    }

    .menu-active {
      display: block;
      height: 30px;
      line-height: 30px;
      padding: 0 20px;
      background-color: #353737;
      margin-right: 10px;
      border-radius: 8px;
      font-size: 18px;
      cursor: pointer;
    }

    .menu-name:hover {
      background-color: #353737;
      border-radius: 8px;
    }
  }
  .menu-child-content {
    background-color: #353737;
    position: absolute;
    top: 100%; // 在菜单项正下方
    left: 0; // 从菜单项左侧开始
    height: 200px;
    width: 150px;
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: center;
    padding: 10px 10px;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    border: 1px solid #4a4a4a;
    .menu-child {
      width: 100%;

      height: 30px;
      line-height: 30px;

      margin-bottom: 10px;
      border-radius: 8px;
      padding-left: 10px;
      cursor: pointer;
    }

    .menu-child:hover {
      background-color: #242526;
    }
  }
}
</style>
