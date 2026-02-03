<template>
  <div class="chat-container">
    <div class="chat-header"></div>
    
    <div class="chat-box" ref="chatBox">
      <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
        <div class="message-content">
          <span style="white-space: pre-wrap;">{{ msg.content }}</span>
        </div>
      </div>
      
      <div v-if="currentAgent" class="agent-status">
        <span class="status-dot"></span>
        ⚡ {{ currentAgent }}...
      </div>
    </div>

    <div class="input-section">
      <div class="custom-input-box">
        <textarea 
          v-model="userInput" 
          @keyup.enter.exact.prevent="sendMessage" 
          placeholder="问问我关于杭州地铁的一切..." 
          :disabled="isStreaming"
        ></textarea>
        
        <img 
          src="~@/assets/play.png" 
          class="play-icon" 
          @click="sendMessage"
          alt="play"
        />
      </div>
    </div>
  </div>
</template>

<script>
// 逻辑部分保持不变，包含 ESLint 修复
export default {
  data() {
    return {
      userInput: '',
      messages: [], 
      threadId: "user_" + Math.random().toString(36).substring(7),
      isStreaming: false,
      currentAgent: '',
    };
  },
  methods: {
    async sendMessage() {
      if (!this.userInput || this.isStreaming) return;
      const query = this.userInput;
      this.messages.push({ role: 'user', content: query });
      this.userInput = '';
      this.isStreaming = true;
      const aiMessage = { role: 'ai', content: '' };
      this.messages.push(aiMessage);

      try {
        const response = await fetch('http://localhost:8000/chat/stream', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ query: query, thread_id: this.threadId })
        });
        const reader = response.body.getReader();
        const decoder = new TextDecoder();
        let buffer = '';
        
        // eslint-disable-next-line no-constant-condition
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          buffer += decoder.decode(value, { stream: true });
          const events = buffer.split('\n\n');
          buffer = events.pop();
          for (const eventStr of events) {
            if (!eventStr.trim()) continue;
            const lines = eventStr.split('\n');
            let eventType = null;
            let eventData = null;
            lines.forEach(line => {
              if (line.startsWith('event: ')) eventType = line.substring(7).trim();
              else if (line.startsWith('data: ')) {
                const dataStr = line.substring(6).trim();
                if (dataStr !== '[DONE]') {
                  try { eventData = JSON.parse(dataStr); } catch (e) { console.error(e); }
                }
              }
            });
            if (eventType === 'agent_start' && eventData) {
              this.currentAgent = eventData.agent;
              this.scrollToBottom();
            } else if (eventType === 'message' && eventData) {
              aiMessage.content += eventData.content;
              this.scrollToBottom();
            }
          }
        }
      } catch (error) {
        aiMessage.content += "\n[网络连接失败]";
      } finally {
        this.isStreaming = false;
        this.currentAgent = '';
      }
    },
    scrollToBottom() {
      this.$nextTick(() => {
        const container = this.$refs.chatBox;
        if (container) container.scrollTop = container.scrollHeight;
      });
    }
  }
};
</script>

<style scoped>
.chat-container { 
  display: flex; 
  flex-direction: column; 
  height: 100%; 
  width: 100%; 
  position: relative; /* 必须为 relative 供内部绝对定位参考 */
  background: transparent;
}

/* 保持 Header 高度但移除文字 */
.chat-header { 
  height: 56px; 
}

.chat-box { 
  flex: 1; 
  overflow-y: auto; 
  padding: 20px 40px; 
}

/* 核心：动态居中定位 */
.input-section { 
  position: absolute; 
  top: 50%;         /* 距离顶端 50% */
  left: 50%;        /* 距离左侧 50% */
  /* 通过 transform 实现真正的中心对齐 */
  transform: translate(-50%, -50%); 
  
  width: 80%;       /* 响应式宽度 */
  max-width: 733px; /* 最大宽度限制 */
  z-index: 10; 
  
  /* 确保位置变动时平滑过渡，与侧边栏动画同步 */
  transition: all 0.3s ease; 
}

.custom-input-box {
  position: relative;
  width: 100%; 
  height: 164px;
  background: #ffffff;
  border: 1px solid rgba(0,0,0,0.00);
  border-radius: 20px;
  box-shadow: 0 4px 24px rgba(0,0,0,0.08); 
  padding: 20px;
  box-sizing: border-box;
}

textarea {
  width: 100%;
  height: calc(100% - 30px);
  border: none;
  outline: none;
  resize: none;
  font-size: 16px;
  font-family: 'PingFang SC', sans-serif;
  color: #333;
  background: transparent;
}

.play-icon {
  position: absolute;
  bottom: 20px;
  right: 20px;
  width: 24px;
  height: 24px;
  cursor: pointer;
  transition: opacity 0.2s;
}

.play-icon:hover { opacity: 0.8; }

/* 气泡样式微调 */
.message { margin-bottom: 24px; display: flex; }
.message.user { justify-content: flex-end; }
.message.ai { justify-content: flex-start; }
.message-content { max-width: 70%; padding: 12px 16px; border-radius: 12px; line-height: 1.6; }
.user .message-content { background: #e0e0e0; color: #333; }
.ai .message-content { background: #fff; border: 1px solid #eee; }

/* 状态样式 */
.agent-status { align-self: center; background: rgba(232, 245, 233, 0.8); color: #2e7d32; padding: 4px 12px; border-radius: 20px; font-size: 12px; margin-bottom: 10px; display: flex; align-items: center; }
.status-dot { width: 8px; height: 8px; background: #2e7d32; border-radius: 50%; margin-right: 6px; }
</style>