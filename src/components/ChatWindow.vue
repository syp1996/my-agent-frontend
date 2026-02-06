<template>
  <div class="chat-container">
    <div class="chat-header"></div> 
    
    <div class="chat-box" ref="chatBox">
      <div class="messages-wrapper">
        <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
          <div v-if="msg.role === 'ai' && (msg.content || msg.hasThought)" class="ai-label-container">
            <span class="ai-label">Agent</span>
          </div>

          <div class="message-content" v-if="msg.content || msg.hasThought">
            
            <div v-if="msg.role === 'ai' && msg.hasThought" class="thought-process">
              <div class="thought-header" @click="msg.isThoughtExpanded = !msg.isThoughtExpanded">
                <span v-if="!msg.isDoneThinking" class="thinking-spinner">🔄</span>
                <span v-else class="thinking-icon">💡</span>
                <span class="thought-title">
                  {{ msg.isDoneThinking ? '思考已完成' : '深度思考中...' }}
                </span>
                <span :class="['toggle-arrow', { 'is-expanded': msg.isThoughtExpanded }]">
                  ▼
                </span>
              </div>

              <div v-if="msg.isThoughtExpanded" class="thought-body">
                <div v-for="(item, tIndex) in msg.timeline" :key="tIndex">
                  
                  <div v-if="item.type === 'step'" class="step-item">
                    <span class="step-icon">{{ item.status === 'loading' ? '⏳' : '✅' }}</span>
                    <span class="step-text">{{ item.title }}</span>
                  </div>

                  <div 
                    v-if="item.type === 'thought'" 
                    class="thought-segment markdown-body" 
                    v-html="renderMarkdown(item.content)"
                  ></div>
                  
                </div>
              </div>
            </div>

            <div 
              v-if="msg.role === 'ai'" 
              class="markdown-body" 
              v-html="renderMarkdown(msg.content)"
            ></div>
            <span v-else style="white-space: pre-wrap;">{{ msg.content }}</span>
          </div>
        </div>
        
        <div v-if="isLoadingHistory" class="loading-tip">正在拉取历史记录...</div>
        </div>
      <div class="scroll-anchor"></div>
    </div>

    <div v-if="messages.length > 0 || isLoadingHistory" class="bottom-mask"></div>

    <div :class="['input-section', { 'at-bottom': messages.length > 0 || isLoadingHistory }]">
      <div class="custom-input-box">
        <textarea 
          v-model="userInput" 
          @keyup.enter.exact.prevent="sendMessage" 
          placeholder="问问我关于杭州地铁的一切..." 
          :disabled="isStreaming"
        ></textarea>
        <img src="~@/assets/play.png" class="play-icon" @click="sendMessage" alt="send" />
      </div>
    </div>
  </div>
</template>

<script>
import DOMPurify from 'dompurify';
import hljs from 'highlight.js';
import 'highlight.js/styles/github.css';
import MarkdownIt from 'markdown-it';

const md = new MarkdownIt({
  html: true,
  linkify: true,
  typographer: true,
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return '<pre class="hljs"><code>' +
               hljs.highlight(str, { language: lang, ignoreIllegals: true }).value +
               '</code></pre>';
      } catch (__) { console.error(__); }
    }
    return '<pre class="hljs"><code>' + md.utils.escapeHtml(str) + '</code></pre>';
  }
});

export default {
  props: {
    threadId: { type: String, required: true }
  },
  data() {
    return {
      userInput: '',
      messages: [], 
      isStreaming: false,
      isLoadingHistory: false,
    };
  },
  watch: {
    threadId: {
      immediate: true,
      handler(newVal) {
        if (newVal) this.fetchHistory(newVal);
      }
    }
  },
  methods: {
    renderMarkdown(content) {
      if (!content) return '';
      const rawHtml = md.render(content);
      return DOMPurify.sanitize(rawHtml);
    },
    async fetchHistory(id) {
      this.isLoadingHistory = true;
      this.messages = [];
      this.userInput = ''; 
      try {
        const res = await fetch(`http://localhost:8000/threads/${id}/history`);
        if (res.ok) {
          const historyData = await res.json();
          const rawMessages = historyData.history || [];
          
          this.messages = rawMessages.map(msg => {
            const isAi = msg.role === 'assistant';
            let timeline = [];
            
            if (isAi) {
               if (msg.thoughts) {
                 timeline.push({ type: 'thought', content: msg.thoughts });
               }
               if (msg.steps && msg.steps.length) {
                 msg.steps.forEach(s => timeline.push({ type: 'step', ...s }));
               }
            }

            return {
              ...msg,
              role: isAi ? 'ai' : msg.role,
              timeline: timeline,
              hasThought: timeline.length > 0,
              isDoneThinking: true,
              isThoughtExpanded: false // 历史记录默认也是折叠的
            };
          });
          this.scrollToBottom();
        }
      } catch (e) { 
        console.error("获取历史记录失败:", e); 
      } finally { 
        this.isLoadingHistory = false; 
      }
    },
    async sendMessage() {
      if (!this.userInput || this.isStreaming) return;
      const isFirstMessage = this.messages.length === 0;
      const query = this.userInput;
      this.messages.push({ role: 'user', content: query });
      this.userInput = ''; 
      this.isStreaming = true;
      
      const aiMessage = { 
        role: 'ai', 
        content: '', 
        timeline: [], 
        hasThought: false,     
        isDoneThinking: false, 
        // ✅ 核心修改点：默认设为 false，即默认折叠
        isThoughtExpanded: false 
      };
      this.messages.push(aiMessage);
      this.scrollToBottom();

      try {
        const response = await fetch('http://localhost:8000/chat/stream', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ query, thread_id: this.threadId })
        });
        const reader = response.body.getReader();
        const decoder = new TextDecoder();
        let buffer = '';
        
        while (this.isStreaming) {
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
                if (dataStr === '[DONE]') {
                   // 结束信号
                } else { 
                   try { eventData = JSON.parse(dataStr); } catch (e) { console.error(e); } 
                }
              }
            });

            // ================== 处理思考流 (Thinking) ==================
            if (eventType === 'thought' && eventData) {
              aiMessage.hasThought = true;
              const lastItem = aiMessage.timeline[aiMessage.timeline.length - 1];
              
              const SEPARATOR = "=====FINAL_ANSWER=====";
              let newContent = eventData.content;

              if (lastItem && lastItem.type === 'thought') {
                lastItem.content += newContent;
                if (lastItem.content.includes(SEPARATOR)) {
                  lastItem.content = lastItem.content.split(SEPARATOR)[0].trim();
                }
              } else {
                if (newContent.includes(SEPARATOR)) {
                  newContent = newContent.split(SEPARATOR)[0].trim();
                }
                if (newContent) {
                  aiMessage.timeline.push({ type: 'thought', content: newContent });
                }
              }
              // 如果用户手动展开了，才滚动到底部；否则保持不动或者只滚到正文
              if (aiMessage.isThoughtExpanded) {
                  this.scrollToBottom();
              }
            }
            
            // ================== 处理步骤 (Step) ==================
            else if (eventType === 'step' && eventData) {
              aiMessage.hasThought = true;
              
              let lastStep = null;
              for (let i = aiMessage.timeline.length - 1; i >= 0; i--) {
                if (aiMessage.timeline[i].type === 'step') {
                  lastStep = aiMessage.timeline[i];
                  break;
                }
              }

              if (lastStep && lastStep.status === 'loading' && eventData.status === 'done') {
                 lastStep.status = 'done';
                 if (eventData.title) lastStep.title = eventData.title;
              } else {
                 aiMessage.timeline.push({
                   type: 'step',
                   status: eventData.status || 'loading',
                   title: eventData.title
                 });
              }
              if (aiMessage.isThoughtExpanded) {
                  this.scrollToBottom();
              }
            }

            // ================== 处理回复 (Message) ==================
            else if (eventType === 'message' && eventData) {
              if (!aiMessage.isDoneThinking) {
                aiMessage.isDoneThinking = true;
                // 这里原本有 isThoughtExpanded = false 的逻辑，现在默认已经是 false 了，
                // 但保留着也无妨，确保回答开始时一定是折叠的（如果用户中间手动展开了）
                aiMessage.isThoughtExpanded = false; 
              }
              aiMessage.content += eventData.content;
              this.scrollToBottom();
            }
            
            else if (eventType === 'title_generated' && eventData) {
              this.$emit('title-generated', {
                thread_id: eventData.thread_id,
                title: eventData.title
              });
            }
          }
        }
      } catch (error) { aiMessage.content += "\n[连接失败]"; }
      finally { 
        this.isStreaming = false; 
        if (aiMessage) aiMessage.isDoneThinking = true;
        if (isFirstMessage) {
          this.$emit('first-message-sent');
        }
      }
    },
    scrollToBottom() {
      this.$nextTick(() => {
        setTimeout(() => {
          const container = this.$refs.chatBox;
          if (container) container.scrollTo({ top: container.scrollHeight, behavior: 'smooth' });
        }, 100);
      });
    }
  }
};
</script>

<style scoped>
/* ================= 布局样式 (严格保持不变) ================= */
.chat-container { display: flex; flex-direction: column; height: 100%; width: 100%; position: relative; background: transparent; }
.chat-header { height: 56px; }
.chat-box { flex: 1; overflow-y: auto; padding: 20px 40px; padding-bottom: 240px; display: flex; flex-direction: column; align-items: center; z-index: 1; outline: none; }
.messages-wrapper { width: 80%; max-width: 733px; display: flex; flex-direction: column; }
.bottom-mask { position: absolute; bottom: 0; left: 0; width: 100%; height: 200px; background: #f8f8f7; z-index: 8; }
.input-section { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 80%; max-width: 733px; z-index: 10; transition: all 0.5s cubic-bezier(0.2, 1, 0.3, 1); outline: none; }
.input-section.at-bottom { top: calc(100% - 104px); }
.custom-input-box { position: relative; width: 100%; height: 164px; background: #ffffff; border-radius: 20px; box-shadow: 0 4px 24px rgba(0,0,0,0.08); padding: 20px; box-sizing: border-box; outline: none; }
textarea { width: 100%; height: calc(100% - 30px); border: none; outline: none; resize: none; font-size: 16px; font-family: 'PingFang SC', sans-serif; color: #333; background: transparent; box-shadow: none; }
.play-icon { position: absolute; bottom: 20px; right: 20px; width: 24px; height: 24px; cursor: pointer; }

/* 消息气泡 */
.message { margin-bottom: 24px; display: flex; width: 100%; }
.message.user { justify-content: flex-end; }
.user .message-content { max-width: 85%; background: #f0f0f0; color: #333; padding: 12px 18px; border-radius: 18px; }
.message.ai { flex-direction: column; align-items: flex-start; } 
.ai .message-content { 
  width: 100%; max-width: 100%; background: #fff; border: 1px solid #f0f0f0; 
  box-shadow: 0 2px 8px rgba(0,0,0,0.02); padding: 12px 18px; border-radius: 18px; 
  border-top-left-radius: 0; line-height: 1.6; font-size: 15px; box-sizing: border-box; 
}

.ai-label-container { width: 100%; display: flex; justify-content: flex-start; margin-bottom: -1px; position: relative; z-index: 2; }
.ai-label { color: #666; font-size: 14px; padding: 12px 12px; font-weight: 500; }

/* ================= Gemini 风格思考过程 ================= */
.thought-process {
  background-color: #f8f9fa;
  border-radius: 12px;
  border: 1px solid #e9ecef;
  margin: 16px 0;
  overflow: hidden;
  transition: all 0.3s ease;
}

.thought-header {
  display: flex;
  align-items: center;
  padding: 14px 16px;
  cursor: pointer;
  background: transparent;
  font-size: 14px;
  font-weight: 500;
  color: #495057;
  user-select: none;
  transition: background-color 0.2s ease;
}
.thought-header:hover { background-color: rgba(0, 0, 0, 0.03); }

.thinking-spinner { animation: spin 1s linear infinite; margin-right: 12px; font-size: 16px; color: #1a73e8; }
.thinking-icon { margin-right: 12px; font-size: 16px; color: #1a73e8; }
.thought-title { flex: 1; letter-spacing: 0.01em; }

.toggle-arrow {
  font-size: 12px; color: #adb5bd; transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  margin-left: 8px; display: inline-block;
}
.toggle-arrow.is-expanded { transform: rotate(180deg); }

.thought-body {
  padding: 0 16px 16px 16px;
  background: transparent;
  color: #3c4043;
  animation: fadeIn 0.3s ease-in-out;
}

.step-item {
  display: flex; align-items: center; margin-top: 12px; margin-bottom: 12px;
  font-size: 13px; font-weight: 500; color: #1f2937;
}
.step-icon { width: 20px; margin-right: 8px; text-align: center; display: inline-block; }

/* ✅ 修改点：思考片段支持 Markdown 样式 */
.thought-segment {
  margin-top: 8px; margin-bottom: 8px; padding-left: 28px;
  font-size: 14px; line-height: 1.6; color: #3c4043;
  position: relative;
}

/* 针对 Markdown 内容的微调，去掉默认 margin */
.thought-segment.markdown-body >>> p { margin: 0 0 8px 0; }
.thought-segment.markdown-body >>> p:last-child { margin-bottom: 0; }

.thought-body > div:first-child .thought-segment,
.thought-body > div:first-child .step-item {
  margin-top: 0;
}

/* 动画 */
@keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
@keyframes fadeIn { from { opacity: 0; transform: translateY(-5px); } to { opacity: 1; transform: translateY(0); } }

/* ================= Markdown (保持不变) ================= */
.markdown-body { word-break: break-word; }
.markdown-body >>> p { margin: 0 0 10px 0; }
.markdown-body >>> p:last-child { margin-bottom: 0; }
.markdown-body >>> code { background-color: rgba(175, 184, 193, 0.2); padding: 0.2em 0.4em; border-radius: 6px; font-family: monospace; }
.markdown-body >>> pre code { background-color: transparent; padding: 0; }
.markdown-body >>> .hljs { padding: 12px; border-radius: 8px; margin: 10px 0; overflow-x: auto; background: #f6f8fa; }
.markdown-body >>> ul, .markdown-body >>> ol { padding-left: 2em; margin-bottom: 10px; }
.loading-tip { text-align: center; color: #bbb; font-size: 13px; padding: 20px; }
.scroll-anchor { height: 1px; width: 100%; flex-shrink: 0; }
</style>