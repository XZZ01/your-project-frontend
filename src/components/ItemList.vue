<template>
  <div>
    <!-- 新增表单 -->
    <div class="form-row">
      <input 
        v-model="newItemName" 
        @keyup.enter="addItem"
        type="text" 
        placeholder="输入新事项..."
      />
      <button @click="addItem">➕ 添加</button>
    </div>

    <!-- 数据列表 -->
    <ul v-if="items.length > 0">
      <li v-for="item in items" :key="item.id">
        <span>{{ item.name }}</span>
        <button @click="deleteItem(item.id)" class="delete-btn">✖</button>
      </li>
    </ul>
    <p v-else class="empty-tip">暂无数据，添加一条吧！</p>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

// 数据状态
const items = ref([])
const newItemName = ref('')

// 获取列表
const fetchItems = async () => {
  try {
    const res = await axios.get('/api/items')
    items.value = res.data
  } catch (error) {
    console.error('获取数据失败:', error)
  }
}

// 添加数据
const addItem = async () => {
  const name = newItemName.value.trim()
  if (!name) return alert('请输入内容')

  try {
    const res = await axios.post('/api/items', { name })
    items.value.push(res.data)   // 更新列表
    newItemName.value = ''       // 清空输入框
  } catch (error) {
    console.error('添加失败:', error)
    alert(error.response?.data?.error || '添加失败')
  }
}

// 删除数据
const deleteItem = async (id) => {
  if (!confirm('确认删除？')) return
  try {
    await axios.delete(`/api/items/${id}`)
    items.value = items.value.filter(item => item.id !== id)
  } catch (error) {
    console.error('删除失败:', error)
    alert('删除失败')
  }
}

// 组件挂载时自动加载数据
onMounted(() => {
  fetchItems()
})
</script>

<style scoped>
.form-row {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}
input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
}
button {
  padding: 10px 20px;
  background: #42b883;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}
button:hover {
  background: #359f6e;
}
ul {
  list-style: none;
  padding: 0;
}
li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 15px;
  background: white;
  margin-bottom: 8px;
  border-radius: 4px;
  border-left: 4px solid #42b883;
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
}
.delete-btn {
  background: #e74c3c;
  padding: 4px 10px;
  font-size: 14px;
  border-radius: 4px;
}
.delete-btn:hover {
  background: #c0392b;
}
.empty-tip {
  color: #999;
  text-align: center;
  padding: 20px 0;
}
</style>