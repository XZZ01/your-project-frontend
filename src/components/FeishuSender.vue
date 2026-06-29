<template>
  <div class="feishu-sender">
    <h2>📨 飞书消息发送</h2>

    <!-- ===== 即时发送区域 ===== -->
    <div class="form-group">
      <label>选择接收人</label>
      <select v-model="selectedOpenIds" multiple size="5" style="width: 100%;">
        <option v-for="user in userList" :key="user.open_id" :value="user.open_id">
          {{ user.name || user.open_id }}
        </option>
      </select>
      <small>按住 Ctrl（Mac: Command）键可多选</small>
    </div>

    <div class="form-group">
        <label>选择部门（发送给部门下所有成员）</label>
        <select v-model="selectedDepartmentIds" multiple size="4" style="width: 100%;">
            <option v-for="dept in departmentList" :key="dept.department_id" :value="dept.department_id">
                {{ dept.name }}
            </option>
        </select>
        <small>按住 Ctrl（Mac: Command）键可多选，将发送给部门下所有成员</small>
    </div>

    <div class="form-group">
      <label>消息类型</label>
      <select v-model="msgType">
        <option value="text">文本 (text)</option>
        <option value="post">富文本 (post)</option>
      </select>
    </div>

    <div class="form-group" v-if="msgType === 'text'">
      <label>消息内容（支持换行和 Emoji）</label>
      <textarea v-model="textContent" rows="6" placeholder="输入消息内容，可换行"></textarea>
    </div>
    <div class="form-group" v-else>
      <label>富文本内容 (JSON 格式)</label>
      <textarea v-model="contentJson" rows="6" placeholder='{"zh_cn": {"content": [[{"tag":"text","text":"第一行"}]]}}'></textarea>
    </div>

    <button @click="sendImmediate" :disabled="isSending">
      {{ isSending ? '发送中...' : '🚀 立即发送' }}
    </button>

    <div v-if="immediateResult" class="result-box" :class="{ success: immediateResult.success, error: !immediateResult.success }">
      <pre>{{ JSON.stringify(immediateResult, null, 2) }}</pre>
    </div>

    <hr />

    <!-- ===== 定时发送区域 ===== -->
    <div class="scheduled-section">
      <h3>⏰ 定时发送</h3>
      <p class="hint">定时消息将使用当前填写的接收人、消息类型和内容</p>

      <div class="form-group">
        <label>重复方式</label>
        <select v-model="repeatType">
          <option value="once">一次性（指定日期时间）</option>
          <option value="daily">每天</option>
          <option value="workday">工作日（自动跳过节假日/周末）</option>
          <option value="weekly">每周（指定星期几）</option>
        </select>
      </div>

      <div class="form-group">
        <label>发送时间（日期+时间）</label>
        <input type="datetime-local" v-model="scheduledDateTime" style="width: 100%; padding: 8px;" />
      </div>

      <div class="form-group" v-if="repeatType === 'weekly'">
        <label>选择星期几（可多选）</label>
        <div style="display: flex; gap: 10px; flex-wrap: wrap;">
          <label v-for="(label, idx) in weekDays" :key="idx" style="display: flex; align-items: center; gap: 4px;">
            <input type="checkbox" :value="idx" v-model="selectedWeekDays" />
            {{ label }}
          </label>
        </div>
      </div>

      <div class="form-group">
        <label>
          <input type="checkbox" v-model="scheduledEnabled" />
          启用定时任务
        </label>
      </div>

      <button @click="saveScheduledTask" :disabled="isSaving">
        {{ isSaving ? '保存中...' : '💾 保存定时配置' }}
      </button>
      <button @click="stopScheduledTask" style="margin-left: 10px; background: #e74c3c;">
        ⏹️ 停止定时任务
      </button>

      <div v-if="scheduledResult" class="result-box" :class="{ success: scheduledResult.success, error: !scheduledResult.success }">
        <pre>{{ JSON.stringify(scheduledResult, null, 2) }}</pre>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';

// ---------- 用户列表 ----------
const userList = ref([]);
const selectedOpenIds = ref([]);

// ---------- 部门列表 ----------
const departmentList = ref([]);
const selectedDepartmentIds = ref([]);

// ---------- 消息内容 ----------
const msgType = ref('text');
const textContent = ref('');
const contentJson = ref('');

// 创建 axios 实例，baseURL 从环境变量读取
const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || '',
  // 如果环境变量未设置（开发环境），baseURL 为空，请求将使用相对路径（由 Vite 代理处理）
});

// 计算 content 对象（根据当前类型）
const getContent = () => {
  if (msgType.value === 'text') {
    return { text: textContent.value };
  } else {
    try {
      return JSON.parse(contentJson.value);
    } catch {
      throw new Error('富文本 JSON 格式错误');
    }
  }
};

// ---------- 即时发送 ----------
const isSending = ref(false);
const immediateResult = ref(null);

const sendImmediate = async () => {
  if (selectedOpenIds.value.length === 0 && selectedDepartmentIds.value.length === 0) {
        alert('请至少选择一个用户或部门');
        return;
    }
  let content;
  try {
    content = getContent();
  } catch (e) {
    alert(e.message);
    return;
  }

  isSending.value = true;
  immediateResult.value = null;
  try {
    const response = await apiClient.post('/api/send-batch-message', {
      openIds: selectedOpenIds.value,
      departmentIds: selectedDepartmentIds.value,
      msgType: msgType.value,
      content,
    });
    immediateResult.value = response.data;
  } catch (error) {
    immediateResult.value = {
      success: false,
      error: error.response?.data?.error || error.message,
    };
  } finally {
    isSending.value = false;
  }
};

// ---------- 定时发送 ----------
const repeatType = ref('once');
const scheduledDateTime = ref('');
const selectedWeekDays = ref([]);
const scheduledEnabled = ref(true);
const isSaving = ref(false);
const scheduledResult = ref(null);

const weekDays = ['周日', '周一', '周二', '周三', '周四', '周五', '周六'];

// 加载当前定时配置
const loadScheduledTask = async () => {
  try {
    const response = await apiClient.get('/api/scheduled-task');
    if (response.data.success && response.data.data) {
      const config = response.data.data;
      scheduledEnabled.value = config.enabled !== false;
      repeatType.value = config.repeatType || 'once';
      // 将日期时间转为 datetime-local 可用的格式
      if (config.dateTime) {
        const d = new Date(config.dateTime);
        if (!isNaN(d)) {
          const year = d.getFullYear();
          const month = String(d.getMonth() + 1).padStart(2, '0');
          const day = String(d.getDate()).padStart(2, '0');
          const hours = String(d.getHours()).padStart(2, '0');
          const minutes = String(d.getMinutes()).padStart(2, '0');
          scheduledDateTime.value = `${year}-${month}-${day}T${hours}:${minutes}`;
        }
      }
      if (config.dayOfWeek !== undefined) {
        selectedWeekDays.value = Array.isArray(config.dayOfWeek) ? config.dayOfWeek : [config.dayOfWeek];
      }
    }
  } catch (error) {
    console.error('加载定时任务配置失败:', error);
  }
};

// 保存定时任务
const saveScheduledTask = async () => {
  // 验证时间
  if (!scheduledDateTime.value) {
    alert('请选择发送时间');
    return;
  }
  if (selectedOpenIds.value.length === 0) {
    alert('请至少选择一个接收人（定时发送将使用当前选中的接收人）');
    return;
  }

  let content;
  try {
    content = getContent();
  } catch (e) {
    alert(e.message);
    return;
  }

  // 处理 dayOfWeek
  let dayOfWeek = undefined;
  if (repeatType.value === 'weekly') {
    if (selectedWeekDays.value.length === 0) {
      alert('请选择至少一个星期几');
      return;
    }
    dayOfWeek = selectedWeekDays.value; // 数组
  }

  const payload = {
    repeatType: repeatType.value,
    dateTime: new Date(scheduledDateTime.value).toISOString(),
    dayOfWeek,
    openIds: selectedOpenIds.value,
    departmentIds: selectedDepartmentIds.value,
    msgType: msgType.value,
    content,
    enabled: scheduledEnabled.value,
  };

  isSaving.value = true;
  scheduledResult.value = null;
  try {
    const response = await apiClient.post('/api/scheduled-task', payload);
    scheduledResult.value = response.data;
  } catch (error) {
    scheduledResult.value = {
      success: false,
      error: error.response?.data?.error || error.message,
    };
  } finally {
    isSaving.value = false;
  }
};

// 停止定时任务
const stopScheduledTask = async () => {
  try {
    const response = await apiClient.delete('/api/scheduled-task');
    scheduledResult.value = response.data;
    scheduledEnabled.value = false;
  } catch (error) {
    scheduledResult.value = {
      success: false,
      error: error.message,
    };
  }
};

// 获取用户列表（与之前一致）
const fetchUsers = async () => {
  try {
    const response = await apiClient.get('/api/users');
    if (response.data.success) {
      userList.value = response.data.data;
    }
  } catch (error) {
    console.error('获取用户列表失败:', error);
  }
};
// 获取部门列表
const fetchDepartments = async () => {
    try {
        const response = await apiClient.get('/api/departments');
        if (response.data.success) {
            departmentList.value = response.data.data;
        }
    } catch (error) {
        console.error('获取部门列表失败:', error);
    }
};

onMounted(() => {
  fetchUsers();
  loadScheduledTask();
  fetchDepartments();
});
</script>

<style scoped>
.feishu-sender {
  margin-top: 30px;
  padding: 20px;
  border-top: 2px dashed #ddd;
}
.feishu-sender h2 {
  color: #333;
  margin-bottom: 20px;
}
.form-group {
  margin-bottom: 15px;
}
.form-group label {
  display: block;
  font-weight: bold;
  margin-bottom: 5px;
  color: #555;
}
.form-group textarea,
.form-group select {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  font-family: monospace;
  box-sizing: border-box;
}
.form-group select {
  font-family: inherit;
}
button {
  padding: 12px 30px;
  background: #0052d9;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
}
button:hover:not(:disabled) {
  background: #003da5;
}
button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
.result-box {
  margin-top: 20px;
  padding: 15px;
  border-radius: 4px;
  background: #f5f5f5;
  border-left: 4px solid #ccc;
}
.result-box.success {
  border-left-color: #52c41a;
  background: #f6ffed;
}
.result-box.error {
  border-left-color: #ff4d4f;
  background: #fff2f0;
}
.result-box pre {
  margin: 0;
  white-space: pre-wrap;
  word-break: break-all;
  font-size: 13px;
}
.scheduled-section {
  margin-top: 30px;
  padding-top: 20px;
  border-top: 2px dashed #ddd;
}
.hint {
  color: #888;
  font-size: 14px;
  margin-bottom: 15px;
}
</style>