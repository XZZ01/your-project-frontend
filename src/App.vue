<template>
  <div id="app">
    <h1>📋 机器人定时批量发送消息</h1>
    <ItemList />
    <FeishuSender />
  </div>
</template>

<script setup>
import FeishuSender from './components/FeishuSender.vue'
</script>

<style>
#app {
  font-family: Arial, Helvetica, sans-serif;
  max-width: 600px;
  margin: 40px auto;
  padding: 20px;
  background: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}
h1 {
  text-align: center;
  color: #333;
}
</style>