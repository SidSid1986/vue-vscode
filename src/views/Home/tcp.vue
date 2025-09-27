<template>
  <div class="tcp-demo">
    <h3>TCP 通信示例</h3>
    <button @click="connect">建立连接</button>
    <button @click="sendMessage" :disabled="!isConnected">发送消息</button>
    <button @click="disconnect" :disabled="!isConnected">断开连接</button>
    <div class="log">
      <p>日志:</p>
      <pre>{{ log }}</pre>
    </div>
  </div>
</template>

<script setup>
import { ref, onUnmounted } from 'vue'
import { createStack } from 'tcpip'

const log = ref('')
const isConnected = ref(false)
let stack = null
let connection = null

// 记录日志
const addLog = (message) => {
  log.value += `[${new Date().toLocaleTimeString()}] ${message}\n`
}

// 建立 TCP 连接
const connect = async () => {
  try {
    // 创建网络栈
    stack = await createStack()
    console.log(stack);
    addLog('网络栈初始化完成')

    // 连接到 TCP 服务器（示例地址，需替换为实际服务地址）
    connection = await stack.connectTcp({
      host: '192.168.3.160', // 目标 IP
      port: 9000, // 目标端口
    })
    isConnected.value = true
    addLog('已连接到 TCP 服务器')

    // 监听服务器发送的数据
    listenForData()
  } catch (error) {
    addLog(`连接失败: ${error.message}`)
  }
}

// 监听服务器数据
const listenForData = async () => {
  try {
    for await (const chunk of connection) {
      const data = new TextDecoder().decode(chunk)
      addLog(`收到服务器数据: ${data}`)
    }
  } catch (error) {
    addLog(`数据接收错误: ${error.message}`)
    isConnected.value = false
  }
}

// 发送消息到服务器
const sendMessage = async () => {
  if (!connection) return
  try {
    const message = 'Hello from Vue!'
    const writer = connection.writable.getWriter()
    await writer.write(new TextEncoder().encode(message))
    await writer.releaseLock()
    addLog(`已发送: ${message}`)
  } catch (error) {
    addLog(`发送失败: ${error.message}`)
  }
}

// 断开连接
const disconnect = async () => {
  if (connection) {
    await connection.close()
    isConnected.value = false
    addLog('已断开连接')
    connection = null
  }
}

// 组件卸载时清理资源
onUnmounted(() => {
  if (connection) {
    connection.close()
  }
  if (stack) {
    stack.close()
  }
})
</script>

<style scoped>
.tcp-demo {
  padding: 20px;
}
button {
  margin: 0 5px 10px;
  padding: 5px 10px;
}
.log {
  margin-top: 10px;
  padding: 10px;
  border: 1px solid #ccc;
  height: 300px;
  overflow: auto;
}
</style>