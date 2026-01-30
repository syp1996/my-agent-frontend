<template>
  <div class="chat-container">
    <div class="chat-header">杭州地铁智能客服 (AI Agent)</div>
    
    <div class="chat-box" ref="chatBox">
      <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
        <div class="message-content">
          <strong>{{ msg.role === 'user' ? '我' : 'AI' }}:</strong>
          <span style="white-space: pre-wrap;">{{ msg.content }}</span>
        </div>
      </div>
      
      <div v-if="currentAgent" class="agent-status">
        <span class="status-dot"></span>
        ⚡ 正在处理: {{ currentAgent }}...
      </div>
    </div>

    <div class="input-area">
      <input 
        v-model="userInput" 
        @keyup.enter="sendMessage" 
        placeholder="输入问题 (例如: 查一下杭州东到西湖的票价)..." 
        :disabled="isStreaming"
      />
      <button @click="sendMessage" :disabled="isStreaming">发送</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      userInput: '',
      messages: [], 
      // 保持会话 ID，实现多轮对话记忆
      threadId: "user_" + Math.random().toString(36).substring(7),
      isStreaming: false,
      currentAgent: '', // 用于显示当前工作的智能体名称
    };
  },
  methods: {
    async sendMessage() {
      if (!this.userInput || this.isStreaming) return;

      const query = this.userInput; // 暂存输入
      this.messages.push({ role: 'user', content: query });
      this.userInput = '';
      this.isStreaming = true;
      this.currentAgent = 'Supervisor (调度中)'; // 初始状态

      // 创建 AI 消息占位
      const aiMessage = { role: 'ai', content: '' };
      this.messages.push(aiMessage);

      try {
        const response = await fetch('http://localhost:8000/chat/stream', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ 
            query: query, 
            thread_id: this.threadId 
          })
        });

        const reader = response.body.getReader();
        const decoder = new TextDecoder();
        let buffer = ''; // 缓冲区，处理断包问题

        // eslint-disable-next-line no-constant-condition
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;

          buffer += decoder.decode(value, { stream: true });
          // SSE 事件通常以双换行符分隔
          const events = buffer.split('\n\n');
          // 保留最后一个可能不完整的块
          buffer = events.pop();

          for (const eventStr of events) {
            if (!eventStr.trim()) continue;
            
            // 解析 event 和 data
            const lines = eventStr.split('\n');
            let eventType = null;
            let eventData = null;

            lines.forEach(line => {
              if (line.startsWith('event: ')) {
                eventType = line.substring(7).trim();
              } else if (line.startsWith('data: ')) {
                const dataStr = line.substring(6).trim();
                if (dataStr === '[DONE]') {
                  this.currentAgent = ''; // 结束
                } else {
                  try {
                    eventData = JSON.parse(dataStr);
                  } catch (e) { console.error('JSON解析失败', e); }
                }
              }
            });

            // 根据不同的事件类型更新 UI
            if (eventType === 'agent_start' && eventData) {
              // 更新状态栏
              this.currentAgent = eventData.agent;
              this.scrollToBottom();
            } else if (eventType === 'message' && eventData) {
              // 更新回复内容
              aiMessage.content += eventData.content;
              this.scrollToBottom();
            } else if (eventType === 'error' && eventData) {
              aiMessage.content += `\n[系统错误: ${eventData.error}]`;
            }
          }
        }
      } catch (error) {
        console.error("请求失败:", error);
        aiMessage.content += "\n[网络连接失败，请检查后端服务]";
      } finally {
        this.isStreaming = false;
        this.currentAgent = ''; // 清除状态
      }
    },
    scrollToBottom() {
      this.$nextTick(() => {
        const container = this.$refs.chatBox;
        container.scrollTop = container.scrollHeight;
      });
    }
  }
};
</script>

<style scoped>
.chat-container { max-width: 800px; margin: 20px auto; border: 1px solid #e0e0e0; border-radius: 12px; font-family: 'Helvetica Neue', Arial, sans-serif; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
.chat-header { background: #2c3e50; color: white; padding: 15px; text-align: center; font-weight: bold; border-radius: 12px 12px 0 0; }
.chat-box { height: 500px; overflow-y: auto; padding: 20px; background: #f5f7fa; display: flex; flex-direction: column; }
.message { margin-bottom: 20px; display: flex; flex-direction: column; }
.message-content { max-width: 80%; padding: 12px 16px; border-radius: 8px; font-size: 15px; }
.user { align-items: flex-end; }
.user .message-content { background: #42b983; color: white; border-radius: 12px 12px 0 12px; }
.ai { align-items: flex-start; }
.ai .message-content { background: white; color: #333; border: 1px solid #ddd; border-radius: 12px 12px 12px 0; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }

/* 智能体状态样式 */
.agent-status { 
  align-self: center; 
  background: #e8f5e9; 
  color: #2e7d32; 
  padding: 4px 12px; 
  border-radius: 20px; 
  font-size: 12px; 
  margin-bottom: 10px; 
  display: flex; 
  align-items: center; 
  animation: fadeIn 0.5s;
}
.status-dot {
  width: 8px; height: 8px; background: #2e7d32; border-radius: 50%; margin-right: 6px;
  animation: pulse 1.5s infinite;
}

@keyframes pulse { 0% { opacity: 0.4; } 50% { opacity: 1; } 100% { opacity: 0.4; } }
@keyframes fadeIn { from { opacity: 0; transform: translateY(5px); } to { opacity: 1; transform: translateY(0); } }

.input-area { display: flex; padding: 15px; border-top: 1px solid #eee; background: white; border-radius: 0 0 12px 12px; }
input { flex: 1; padding: 12px; border: 1px solid #ddd; border-radius: 6px; outline: none; font-size: 16px; transition: border 0.3s; }
input:focus { border-color: #42b983; }
button { margin-left: 12px; padding: 0 24px; background: #42b983; color: white; border: none; border-radius: 6px; font-size: 16px; cursor: pointer; transition: background 0.3s; }
button:hover { background: #3aa876; }
button:disabled { background: #a8d5c2; cursor: not-allowed; }
</style>