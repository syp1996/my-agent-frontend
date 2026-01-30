<template>
  <div class="chat-container">
    <div class="chat-header">DeepSeek Agent (Vue 2 版)</div>
    
    <div class="chat-box" ref="chatBox">
      <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
        <strong>{{ msg.role === 'user' ? '我' : 'Agent' }}:</strong>
        <span>{{ msg.content }}</span>
      </div>
    </div>

    <div class="input-area">
      <input 
        v-model="userInput" 
        @keyup.enter="sendMessage" 
        placeholder="输入问题..." 
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
      messages: [], // 格式: { role: 'user' | 'ai', content: '' }
      threadId: "user_" + Math.random().toString(36).substring(7),
      isStreaming: false,
    };
  },
  methods: {
    async sendMessage() {
      if (!this.userInput || this.isStreaming) return;

      const message = this.userInput;
      this.messages.push({ role: 'user', content: message });
      this.userInput = '';
      this.isStreaming = true;

      // 为 AI 回复创建一个占位对象，方便后续流式更新
      const aiMessage = { role: 'ai', content: '' };
      this.messages.push(aiMessage);

      try {
        const response = await fetch('http://localhost:8000/chat', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ 
            message: message, 
            thread_id: this.threadId 
          })
        });

        const reader = response.body.getReader();
        const decoder = new TextDecoder();

        while (true) {
          const { done, value } = await reader.read();
          if (done) break;

          const chunk = decoder.decode(value);
          const lines = chunk.split('\n');

          for (const line of lines) {
            if (line.startsWith('data: ')) {
              const dataStr = line.slice(6);
              if (dataStr === '[DONE]') break;
              
              try {
                const data = JSON.parse(dataStr);
                // 响应式更新 AI 消息内容
                aiMessage.content += data.content;
                this.scrollToBottom();
              } catch (e) {
                console.error("解析错误:", e);
              }
            }
          }
        }
      } catch (error) {
        console.error("请求失败:", error);
        aiMessage.content = "抱歉，连接服务器出错。";
      } finally {
        this.isStreaming = false;
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
.chat-container { max-width: 600px; margin: 20px auto; border: 1px solid #ddd; border-radius: 8px; }
.chat-header { background: #42b983; color: white; padding: 10px; text-align: center; }
.chat-box { height: 400px; overflow-y: auto; padding: 15px; background: #f9f9f9; }
.message { margin-bottom: 15px; line-height: 1.5; }
.user { color: #2c3e50; text-align: right; }
.ai { color: #42b983; }
.input-area { display: flex; padding: 10px; border-top: 1px solid #ddd; }
input { flex: 1; padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
button { margin-left: 10px; padding: 8px 15px; background: #42b983; color: white; border: none; border-radius: 4px; cursor: pointer; }
button:disabled { background: #ccc; }
</style>