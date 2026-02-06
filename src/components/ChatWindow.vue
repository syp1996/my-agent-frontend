<template>
  <div class="chat-container">
    <div class="chat-header"></div> 
    
    <div class="chat-box" ref="chatBox">
      <div class="messages-wrapper">
        <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
          <div v-if="msg.role === 'ai'" class="ai-label-container">
            <span class="ai-label">Agent</span>
          </div>

          <div class="message-content" v-if="msg.role === 'ai' || msg.content || msg.hasThought">
            
            <div v-if="msg.role === 'ai' && !msg.hasThought && !msg.content" class="preparing-state">
              <span class="dot-flashing"></span>
              <span class="preparing-text">正在接入神经系统...</span>
            </div>

            <template v-else>
              <div 
                v-if="msg.role === 'ai' && msg.hasThought" 
                :class="['thought-process', { 'thinking-active': !msg.isDoneThinking }]"
              >
                <div class="thought-header" @click="msg.isThoughtExpanded = !msg.isThoughtExpanded">
                  <div class="status-indicator">
                    <span v-if="!msg.isDoneThinking" class="thinking-spinner-modern"></span>
                    <span v-else class="thinking-icon-done">✨</span>
                  </div>
                  
                  <div class="status-text">
                    <span class="status-title">
                      {{ msg.isDoneThinking ? '深度思考已完成' : '正在思考' }}
                    </span>
                    <span class="status-timer" v-if="msg.thinkingDuration > 0">
                      {{ msg.thinkingDuration }}s
                    </span>
                  </div>

                  <span :class="['toggle-arrow', { 'is-expanded': msg.isThoughtExpanded }]">
                    <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round"><polyline points="6 9 12 15 18 9"></polyline></svg>
                  </span>
                </div>

                <div v-if="msg.isThoughtExpanded" class="thought-body">
                  <div class="timeline-container">
                    <div v-for="(item, tIndex) in msg.timeline" :key="tIndex" class="timeline-item">
                      
                      <div v-if="item.type === 'step'" class="step-card">
                        <div class="step-status">
                          <span v-if="item.status === 'loading'" class="pulse-dot"></span>
                          <span v-else class="check-icon">✓</span>
                        </div>
                        <span class="step-text">{{ item.title }}</span>
                      </div>

                      <div 
                        v-if="item.type === 'thought'" 
                        :class="['thought-segment', 'markdown-body', { 'is-streaming': !msg.isDoneThinking && tIndex === msg.timeline.length - 1 }]" 
                        v-html="renderMarkdown(item.content)"
                      ></div>
                      
                    </div>
                  </div>
                </div>
              </div>

              <div 
                v-if="msg.role === 'ai'" 
                class="markdown-body" 
                v-html="renderMarkdown(msg.content)"
              ></div>
              <span v-else style="white-space: pre-wrap;">{{ msg.content }}</span>
            </template>

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
      timerInterval: null, // ✅ 新增：用于计时器的 Interval
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
              isThoughtExpanded: false,
              thinkingDuration: 0 // 历史记录不显示具体耗时，或默认为0
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
        isThoughtExpanded: true, // ⚠️ 流式输出时，建议默认展开，让用户看到“心流”
        thinkingDuration: 0.0,   // ✅ 新增：计时器
        startTime: Date.now()    // ✅ 新增：记录开始时间
      };
      this.messages.push(aiMessage);
      this.scrollToBottom();

      // ✅ 开启计时器
      this.timerInterval = setInterval(() => {
        if (!aiMessage.isDoneThinking) {
          const now = Date.now();
          aiMessage.thinkingDuration = ((now - aiMessage.startTime) / 1000).toFixed(1);
        }
      }, 100);

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
              if (aiMessage.isThoughtExpanded) this.scrollToBottom();
            }
            
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
              if (aiMessage.isThoughtExpanded) this.scrollToBottom();
            }

            else if (eventType === 'message' && eventData) {
              if (!aiMessage.isDoneThinking) {
                aiMessage.isDoneThinking = true;
                clearInterval(this.timerInterval); // ✅ 停止计时
                
                // 思考结束时，自动收起（模仿 Manus/R1 的交互）
                // 延迟一小会儿收起，让用户看到“完成”的状态
                setTimeout(() => {
                  aiMessage.isThoughtExpanded = false; 
                }, 800);
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
      } catch (error) { 
        aiMessage.content += "\n[连接失败]"; 
        aiMessage.isDoneThinking = true;
        clearInterval(this.timerInterval);
      } finally { 
        this.isStreaming = false; 
        if (aiMessage) aiMessage.isDoneThinking = true;
        clearInterval(this.timerInterval); // 确保最后一定停止
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
/* ================= 布局样式 (保持不变) ================= */
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
.message { margin-bottom: 24px; display: flex; width: 100%; }
.message.user { justify-content: flex-end; }
.user .message-content { max-width: 85%; background: #f0f0f0; color: #333; padding: 12px 18px; border-radius: 18px; }
.message.ai { flex-direction: column; align-items: flex-start; } 
.ai .message-content { width: 100%; max-width: 100%; background: #fff; border: 1px solid #f0f0f0; box-shadow: 0 2px 8px rgba(0,0,0,0.02); padding: 12px 18px; border-radius: 18px; border-top-left-radius: 0; line-height: 1.6; font-size: 15px; box-sizing: border-box; }
.ai-label-container { width: 100%; display: flex; justify-content: flex-start; margin-bottom: -1px; position: relative; z-index: 2; }
.ai-label { color: #666; font-size: 14px; padding: 12px 12px; font-weight: 500; }

/* ================= ✅ Manus Style 思考过程样式重构 ================= */

/* 1. 容器：更现代的圆角和背景 */
.thought-process {
  background-color: #f7f9fb; /* 极淡的冷灰色 */
  border-radius: 12px;
  border: 1px solid #e2e8f0;
  margin: 12px 0 16px 0;
  overflow: hidden;
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
}

/* 2. 呼吸态：当正在思考时，边框有微弱的蓝色脉冲 */
.thought-process.thinking-active {
  border-color: #bfdbfe;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.05);
}

/* 3. 头部：状态栏设计 */
.thought-header {
  display: flex;
  align-items: center;
  padding: 12px 16px;
  cursor: pointer;
  background: transparent;
  user-select: none;
  min-height: 24px;
}
.thought-header:hover { background-color: rgba(0,0,0,0.02); }

.status-indicator { display: flex; align-items: center; margin-right: 12px; }

/* 现代加载圈 (类似 iOS) */
.thinking-spinner-modern {
  width: 16px; height: 16px;
  border: 2px solid #e2e8f0;
  border-top-color: #3b82f6; /* 科技蓝 */
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

.thinking-icon-done { font-size: 14px; line-height: 1; }

.status-text { flex: 1; display: flex; align-items: center; font-family: 'SF Mono', 'Roboto Mono', monospace; }
.status-title { font-size: 13px; font-weight: 600; color: #475569; letter-spacing: -0.01em; }
.status-timer { 
  margin-left: 10px; font-size: 12px; color: #94a3b8; background: rgba(0,0,0,0.04); 
  padding: 2px 6px; border-radius: 4px;
}

.toggle-arrow { color: #cbd5e1; transition: transform 0.3s ease; display: flex; }
.toggle-arrow.is-expanded { transform: rotate(180deg); }

/* 4. 内容区域：时间轴样式 */
.thought-body {
  padding: 8px 16px 16px 16px;
  border-top: 1px solid #f1f5f9;
  background: #ffffff; /* 展开后内部偏白 */
}

.timeline-container {
  position: relative;
  padding-left: 8px; /* 给左边框留空间 */
}
/* 左侧时间轴线 */
.timeline-container::before {
  content: ''; position: absolute; left: 0; top: 8px; bottom: 8px;
  width: 2px; background: #f1f5f9; border-radius: 2px;
}

.timeline-item { position: relative; margin-bottom: 12px; padding-left: 16px; }

/* 步骤卡片 (Search, Code etc.) */
.step-card {
  display: inline-flex; align-items: center;
  background: #fff; border: 1px solid #e2e8f0;
  padding: 6px 12px; border-radius: 6px;
  box-shadow: 0 1px 2px rgba(0,0,0,0.03);
  margin-bottom: 8px;
}
.step-status { margin-right: 8px; display: flex; align-items: center; }
.pulse-dot { 
  width: 8px; height: 8px; background: #3b82f6; border-radius: 50%; 
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.2);
  animation: pulse 1.5s infinite;
}
.check-icon { color: #10b981; font-size: 12px; font-weight: bold; }
.step-text { font-size: 12px; font-weight: 500; color: #334155; }

/* 思考文本 */
.thought-segment {
  font-size: 13px; line-height: 1.7; color: #64748b; /* 思考内容颜色偏淡 */
}

/* 打字机光标效果：仅在最后一段且正在生成时显示 */
.thought-segment.is-streaming::after {
  content: '▋';
  display: inline-block;
  color: #3b82f6;
  font-size: 12px;
  margin-left: 2px;
  animation: blink 1s step-end infinite;
}

/* 动画定义 */
@keyframes spin { to { transform: rotate(360deg); } }
@keyframes pulse { 0% { opacity: 1; transform: scale(1); } 50% { opacity: 0.5; transform: scale(1.2); } 100% { opacity: 1; transform: scale(1); } }
@keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0; } }

/* Markdown 微调 */
.markdown-body >>> p { margin: 0 0 6px 0; }
.markdown-body >>> code { font-size: 85%; background: rgba(0,0,0,0.03); }

/* 初始化状态 */
.preparing-state { display: flex; align-items: center; padding: 4px 0; color: #94a3b8; font-size: 13px; }
.preparing-text { margin-left: 8px; font-family: monospace; }
.dot-flashing { position: relative; width: 6px; height: 6px; border-radius: 5px; background-color: #cbd5e1; animation: dot-flashing 1s infinite linear alternate; animation-delay: 0.5s; }
.dot-flashing::before, .dot-flashing::after { content: ''; display: inline-block; position: absolute; top: 0; width: 6px; height: 6px; border-radius: 5px; background-color: #cbd5e1; animation: dot-flashing 1s infinite alternate; }
.dot-flashing::before { left: -10px; animation-delay: 0s; }
.dot-flashing::after { left: 10px; animation-delay: 1s; }
@keyframes dot-flashing { 0% { background-color: #cbd5e1; } 50%, 100% { background-color: #94a3b8; } }

/* 通用样式 */
.loading-tip { text-align: center; color: #bbb; font-size: 13px; padding: 20px; }
.scroll-anchor { height: 1px; width: 100%; flex-shrink: 0; }
</style>