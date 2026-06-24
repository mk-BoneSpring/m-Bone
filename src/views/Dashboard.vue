<script setup lang="ts">
import { useRouter } from 'vue-router'

const router = useRouter()

function logout() {
  localStorage.removeItem('token')
  router.push('/login')
}
</script>

<template>
  <div class="dashboard">
    <p>Dashboard — 内容待补充</p>
    <button @click="logout">退出登录</button>
  </div>
</template>

<style scoped>
.dashboard {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100vh;
  gap: 16px;
  color: #333;
  font-size: 16px;
}
button {
  padding: 8px 20px;
  border: 1px solid #ccc;
  border-radius: 6px;
  cursor: pointer;
  background: #fff;
}
</style>
