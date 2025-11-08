<template>
  <div class="chat-container">
    <!-- 头部标题 -->
    <div class="header">
      <h1>校园AI助手</h1>
      <span :class="['status', isOnline ? 'online' : 'offline']">
        {{ isOnline ? '● 在线' : '● 离线' }}
      </span>
    </div>

    <!-- 聊天消息窗口 -->
    <div class="messages-list" ref="messageList">
      <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
        <div class="avatar">{{ msg.role === 'user' ? '我' : 'AI' }}</div>
        <div class="content">{{ msg.content }}</div>
      </div>
      <!-- AI 思考中的提示 -->
      <div v-if="isLoading" class="message assistant">
        <div class="avatar">AI</div>
        <div class="content typing-indicator">
          <span></span><span></span><span></span>
        </div>
      </div>
    </div>

    <!-- 输入区域 -->
    <div class="input-area">
      <input
        v-model="userInput"
        type="text"
        placeholder="你好，有什么可以帮你的吗？"
        @keyup.enter="handleSendMessage"
        :disabled="isLoading"
      />
      <button @click="handleSendMessage" :disabled="isLoading">
        发送
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';

// --- API 配置 (从你的JS文件中提取) ---
const SILICONFLOW_API_URL = 'https://api.siliconflow.cn/v1/chat/completions';


const SILICONFLOW_TOKEN = 'sk-xmddxeqqcaeatgxwkbukioyqnoxhpaubhaojmlmfmxxwjosg';

// --- 组件状态定义 ---

// 聊天消息列表
const messages = ref([
  { role: 'assistant', content: '你好！我是你的专属校园AI助手，无论是学习问题、校园生活还是寻找活动，都可以问我哦！' }
]);

// 用户输入的内容
const userInput = ref('');

// 是否正在等待AI回复
const isLoading = ref(false);

// AI服务是否在线
const isOnline = ref(false);

// 聊天窗口的DOM引用，用于控制滚动条
const messageList = ref(null);

// --- 方法定义 ---

/**
 * 滚动聊天窗口到底部
 */
const scrollToBottom = async () => {
  // nextTick 确保DOM已经更新完毕
  await nextTick();
  if (messageList.value) {
    messageList.value.scrollTop = messageList.value.scrollHeight;
  }
};

/**
 * 发送消息给AI (使用标准的 fetch API)
 * @param {string} message 用户消息
 */
const sendMessageToAI = async (message) => {
  try {
    const response = await fetch(SILICONFLOW_API_URL, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${SILICONFLOW_TOKEN}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: "Qwen/Qwen-32B-Chat", // 注意模型名称可能需要根据API文档更新
        messages: [
          {
            role: "system",
            content: "你是一个校园AI助手，专门为大学生提供帮助。请用简洁、友好的语言回答问题，重点关注校园生活、学习、服务等相关内容。"
          },
          {
            role: "user",
            content: message
          }
        ],
        stream: false,
        max_tokens: 1000,
        temperature: 0.7,
        top_p: 0.7,
        frequency_penalty: 0.5
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP 错误! 状态: ${response.status}`);
    }

    const data = await response.json();

    if (data.choices && data.choices.length > 0) {
      const rawContent = data.choices[0].message.content || '';
      return rawContent.trim();
    } else {
      throw new Error('API返回了无效的数据格式');
    }

  } catch (error) {
    console.error('AI API调用失败:', error);
    return '抱歉，AI助手暂时无法回复，请稍后再试。';
  }
};

/**
 * 处理用户发送消息的事件
 */
const handleSendMessage = async () => {
  const message = userInput.value.trim();
  if (!message || isLoading.value) {
    return;
  }

  // 1. 将用户消息添加到聊天列表
  messages.value.push({ role: 'user', content: message });
  userInput.value = ''; // 清空输入框
  scrollToBottom();

  // 2. 设置加载状态
  isLoading.value = true;
  
  // 3. 调用API获取AI回复
  const aiReply = await sendMessageToAI(message);

  // 4. 将AI回复添加到聊天列表
  messages.value.push({ role: 'assistant', content: aiReply });
  
  // 5. 取消加载状态
  isLoading.value = false;
  scrollToBottom();
};

/**
 * 检查AI服务状态
 */
 const getAIStatus = async () => {
  try {
    // 发送一个非常简短的测试消息
    const response = await fetch(SILICONFLOW_API_URL, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${SILICONFLOW_TOKEN}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: "Qwen/Qwen-32B-Chat",
        messages: [{ role: "user", content: "你好" }],
        stream: false,
        max_tokens: 2
      })
    });
    isOnline.value = response.ok;
  } catch (error) {
    console.error('AI状态检查失败:', error);
    isOnline.value = false;
  }
};

// --- 生命周期钩子 ---

// 组件加载完成后，检查一次AI状态
onMounted(() => {
  getAIStatus();
});
</script>

<style scoped>
/* 整个容器样式 */
.chat-container {
  width: 100%;
  max-width: 600px;
  height: 80vh;
  margin: 5vh auto;
  border: 1px solid #ccc;
  border-radius: 10px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
}

/* 头部样式 */
.header {
  padding: 15px;
  background-color: #f5f5f5;
  border-bottom: 1px solid #ddd;
  text-align: center;
  position: relative;
}

.header h1 {
  margin: 0;
  font-size: 1.2em;
}

.header .status {
  position: absolute;
  right: 15px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 0.9em;
}
.status.online { color: #28a745; }
.status.offline { color: #dc3545; }

/* 消息列表区域 */
.messages-list {
  flex-grow: 1;
  padding: 20px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 15px; /* 消息之间的间距 */
  background-color: #e9ebee;
}

/* 单条消息样式 */
.message {
  display: flex;
  align-items: flex-end;
  max-width: 80%;
  gap: 10px;
}

.message .avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  font-weight: bold;
  color: white;
  flex-shrink: 0; /* 防止头像被压缩 */
}

.message .content {
  padding: 10px 15px;
  border-radius: 18px;
  word-wrap: break-word;
  white-space: pre-wrap; /* 保留换行和空格 */
  line-height: 1.5;
}

/* AI助手的消息样式 */
.message.assistant {
  align-self: flex-start;
}
.message.assistant .avatar {
  background-color: #007bff;
}
.message.assistant .content {
  background-color: #ffffff;
  border-top-left-radius: 0;
}

/* 用户的消息样式 */
.message.user {
  align-self: flex-end;
  flex-direction: row-reverse; /* 头像在右边 */
}
.message.user .avatar {
  background-color: #28a745;
}
.message.user .content {
  background-color: #dcf8c6;
  border-top-right-radius: 0;
}

/* 输入区域样式 */
.input-area {
  display: flex;
  padding: 10px;
  border-top: 1px solid #ddd;
  background-color: #f5f5f5;
}

.input-area input {
  flex-grow: 1;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 20px;
  margin-right: 10px;
  outline: none;
  font-size: 1em;
}
.input-area input:focus {
  border-color: #007bff;
}

.input-area button {
  padding: 10px 20px;
  border: none;
  background-color: #007bff;
  color: white;
  border-radius: 20px;
  cursor: pointer;
  font-size: 1em;
  transition: background-color 0.2s;
}

.input-area button:hover {
  background-color: #0056b3;
}

.input-area button:disabled {
  background-color: #a0a0a0;
  cursor: not-allowed;
}

/* AI思考中...的加载动画 */
.typing-indicator {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 3px;
}
.typing-indicator span {
  width: 8px;
  height: 8px;
  background-color: #999;
  border-radius: 50%;
  animation: bounce 1s infinite;
}
.typing-indicator span:nth-child(2) {
  animation-delay: 0.2s;
}
.typing-indicator span:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes bounce {
  0%, 60%, 100% {
    transform: translateY(0);
  }
  30% {
    transform: translateY(-5px);
  }
}
</style>
