<template>
    <div class="h-screen flex flex-col overflow-y-auto" ref="chatContainer">
        <!-- 聊天记录区域 -->
        <!-- 聊天记录区域 -->
        <div class="flex-1 max-w-3xl mx-auto pt-4 px-4">
            <!-- 遍历聊天记录 -->
            <template v-for="(chat, index) in chatList" :key="index">
                <!-- 用户提问消息（靠右） -->
                <div v-if="chat.role === 'user'" class="flex justify-end mb-4">
                    <div class="quesiton-container">
                        <p>{{ chat.content }}</p>
                    </div>
                </div>

                <!-- 大模型回复消息（靠左） -->
                <div v-else class="flex mb-4">
                    <!-- 头像 -->
                    <div class="flex-shrink-0 mr-3">
                        <div
                            class="w-8 h-8 rounded-full flex items-center justify-center border border-gray-200"
                        >
                            <SvgIcon
                                name="deepseek-logo"
                                customCss="w-5 h-5"
                            ></SvgIcon>
                        </div>
                    </div>
                    <!-- 回复的内容 -->
                    <div class="p-1 max-w-[80%] mb-2">
                        <StreamMarkdownRender :content="chat.content" />
                    </div>
                </div>
            </template>
        </div>
        <!-- 提问输入框 -->
        <div class="sticky max-w-3xl mx-auto bg-white bottom-0 left-0 w-full">
            <div
                class="bg-gray-100 rounded-3xl px-4 py-3 mx-4 border border-gray-200"
            >
                <textarea
                    v-model="message"
                    @keydown.enter.prevent="sendMessage"
                    placeholder="给小哈 AI 机器人发送消息"
                    class="bg-transparent border-none outline-none w-full text-sm resize-none min-h-[24px]"
                    rows="2"
                    @input="autoResize"
                    ref="textareaRef"
                >
                </textarea>
                <!-- 发送按钮 -->
                <div class="flex justify-end">
                    <button
                        @click="sendMessage"
                        :disabled="!message.trim()"
                        class="flex items-center justify-center bg-[#4d6bfe] rounded-full w-8 h-8 border border-[#4d6bfe] hover:bg-[#3b5bef] transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
                    >
                        <SvgIcon
                            name="up-arrow"
                            customCss="w-5 h-5 text-white"
                        ></SvgIcon>
                    </button>
                </div>
            </div>
            <!-- 提示文字 -->
            <div
                class="flex items-center justify-center text-xs text-gray-400 mt-2"
            >
                内容由 AI 生成，请仔细甄别
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, onBeforeUnmount, nextTick } from 'vue';
import SvgIcon from '@/components/SvgIcon.vue';
import StreamMarkdownRender from '@/components/StreamMarkdownRender.vue';
// 输入的消息
const message = ref('');
// textarea 引用
const textareaRef = ref(null);
const chatContainer = ref(null);

// 自动调整文本域高度
const autoResize = () => {
    const textarea = textareaRef.value;
    if (textarea) {
        // 若文本域存在
        textarea.style.height = 'auto'; // 1. 先将高度重置为 'auto'
        // 计算新高度，但最大不超过 300px
        const newHeight = Math.min(textarea.scrollHeight, 300);
        textarea.style.height = newHeight + 'px';

        // 如果内容超出 300px，则启用滚动
        textarea.style.overflowY =
            textarea.scrollHeight > 300 ? 'auto' : 'hidden';
    }
};

const chatList = ref([
    {
        role: 'assistant',
        content:
            '我是小哈智能 AI 助手！✨ 我可以帮你解答各种问题，无论是学习、工作，还是日常生活中的小困惑，都可以找我聊聊。有什么我可以帮你的吗？😊',
    },
]);

// 发送消息
let eventSource = null;
const sendMessage = async (e) => {
    if (e.isComposing) return; // 中文输入时不发送
    // 校验发送的消息不能为空
    if (!message.value.trim()) return;

    // 将用户发送的消息添加到 chatList 聊天列表中
    const userMessage = message.value.trim();
    chatList.value.push({ role: 'user', content: userMessage });

    // 点击发送按钮后，清空输入框
    message.value = '';
    // 将输入框的高度重置
    if (textareaRef.value) {
        textareaRef.value.style.height = 'auto';
    }

    // 添加一个占位的回复消息
    chatList.value.push({ role: 'assistant', content: '' });

    try {
        // TODO 创建 SSE 链接
        eventSource = new EventSource(
            `http://localhost:8080/v6/ai/generateStream?message=${encodeURIComponent(
                userMessage
            )}`
        );
        let responseText = '';
        // 处理消息事件
        eventSource.onmessage = (event) => {
            console.log('接收到数据: ', event.data);
            if (event.data) {
                // 解析 json
                const response = JSON.parse(event.data);
                responseText += response.v;
                // 将回复的消息添加到 chatList 聊天列表中
                chatList.value[chatList.value.length - 1].content =
                    responseText;
                scrollToBottom();
            }
        };
        // 处理错误
        eventSource.onerror = (error) => {
            // 通常 SSE 在完成传输后会触发一次 error 事件，这是正常的
            if (error.eventPhase === EventSource.CLOSED) {
                console.log('SSE正常关闭');
            } else {
                // 提示用户 “请求出错”
                chatList.value[chatList.value.length - 1].content =
                    '抱歉，请求出错了，请稍后重试。';
            }

            // 关闭 SSE
            closeSSE();
            scrollToBottom();
        };
    } catch (error) {
        console.error('发送消息错误: ', error);
        // 提示用户 “请求出错”
        chatList.value[chatList.value.length - 1].content =
            '抱歉，请求出错了，请稍后重试。';
        scrollToBottom();
    }
};
// 关闭 SSE 连接
const closeSSE = () => {
    if (eventSource) {
        eventSource.close();
        eventSource = null;
    }
};

// 组件卸载时自动关闭连接
onBeforeUnmount(() => {
    closeSSE();
});

// 滚动到底部
const scrollToBottom = async () => {
    await nextTick(); // 等待 Vue.js 完成 DOM 更新
    if (chatContainer.value) {
        // 若容器存在
        // 将容器的滚动条位置设置到最底部
        const container = chatContainer.value;
        container.scrollTop = container.scrollHeight;
    }
};
</script>

<style scoped>
.quesiton-container {
    font-size: 16px;
    line-height: 28px;
    color: #262626;
    padding: calc((44px - 28px) / 2) 20px;
    box-sizing: border-box;
    white-space: pre-wrap;
    word-break: break-word;
    background-color: #eff6ff;
    border-radius: 14px;
    max-width: calc(100% - 48px);
    position: relative;
}
/* 聊天内容区域样式 */
.overflow-y-auto {
    scrollbar-color: rgba(0, 0, 0, 0.2) transparent; /* 自定义滚动条颜色 */
}
.markdown-container {
    width: 100%;
    line-height: 24px;
    color: rgb(64 64 64);
}

/* 第一个 p 标签的上边距设置为0 */
:deep(.markdown-container > p:first-child),
:deep(p:first-child) {
    margin-top: 0;
    margin-bottom: 0;
}

/* Markdown 转换为 HTML 的样式 */

/* 修复标题选择器 - 使用逗号分隔多个选择器 */
:deep(h1),
:deep(h2),
:deep(h3),
:deep(h4),
:deep(h5),
:deep(h6) {
    font-weight: 600;
    margin: calc(1.143 * 16px) 0 calc(1.143 * 12px) 0;
}

:deep(h1) {
    font-size: 1.5em;
    margin-top: 1.2em;
    margin-bottom: 0.7em;
    line-height: 1.5;
}

:deep(h2) {
    font-size: 1.3em;
    margin-top: 1.1em;
    margin-bottom: 0.6em;
    line-height: 1.5;
}

:deep(h3) {
    font-size: calc(1.143 * 16px);
    line-height: 1.5;
}

:deep(p) {
    line-height: 1.7;
    margin: calc(1.143 * 12px) 0;
    font-size: calc(1.143 * 14px);
}

:deep(ul) {
    list-style: disc; /* 实心圆点 */
    margin-top: 0.6em;
    margin-bottom: 0.9em;
    padding-left: 2em;
}

:deep(ol) {
    list-style: decimal;
    margin-top: 0.6em;
    margin-bottom: 0.9em;
    padding-left: 2em;
}

/* 列表项样式 */
:deep(li) {
    margin-bottom: 0.5em;
    line-height: 1.7;
}

/* 修复列表标记样式 */
:deep(ol li::marker) {
    line-height: calc(1.143 * 25px);
    color: rgb(139 139 139);
}

:deep(ul li::marker) {
    color: rgb(139 139 139);
}

/* 嵌套列表样式 */
:deep(ul ul) {
    list-style: circle;
    margin-top: 0.3em;
    margin-bottom: 0.3em;
}

:deep(ul ul ul) {
    list-style: square; /* 三级列表使用方块 */
}

:deep(pre) {
    background-color: #fafafa;
    padding: 1em;
    border-radius: 5px;
    overflow-x: auto;
    max-width: 100%; /* 确保不超过容器宽度 */
    white-space: pre; /* 保持原始格式 */
    word-wrap: normal; /* 不在单词内部换行 */
}

/* 单独的 code 标签样式 - 不在 pre 内的code */
:deep(:not(pre) > code) {
    font-size: 0.875em;
    font-weight: 600;
    background-color: #ececec;
    border-radius: 4px;
    padding: 0.15rem 0.3rem;
    margin: 0 0.2rem;
}

/* pre 内的 code 标签样式 */
:deep(pre > code) {
    font-size: 0.875em;
    background-color: transparent;
    padding: 0;
    border-radius: 0;
    /* font-family: monospace; */
    font-weight: normal;
    color: #333;
    display: block;
    width: 100%;
}

:deep(a) {
    color: #4d6bfe;
    text-decoration: none;
}

:deep(a:hover) {
    text-decoration: underline;
}

:deep(blockquote) {
    border-left: 4px solid #e5e5e5;
    padding-left: 1em;
    margin: 1em 0;
    color: #666;
}

:deep(table) {
    border-collapse: collapse;
    width: 100%;
    margin: 1em 0;
    font-size: 0.95em;
}

:deep(th),
:deep(td) {
    border: 1px solid #e5e5e5;
    padding: 0.6em;
    text-align: left;
}

:deep(th) {
    background-color: #f5f5f5;
}

:deep(hr) {
    background-color: rgb(229 229 229);
    margin: 1.5em 0;
    height: 1px;
    border: none;
}

/* 确保相邻元素之间的间距一致且适当 */
:deep(h1 + p),
:deep(h2 + p),
:deep(h3 + p) {
    margin-top: 0.5em;
}

:deep(p + ul),
:deep(p + ol) {
    margin-top: 0.5em;
}

:deep(ul + p),
:deep(ol + p) {
    margin-top: 0.7em;
}

/* 代码块包装器样式 */
:deep(.code-block-wrapper) {
    margin: 1em 0;
    border-radius: 14px;
    overflow: hidden;
    background-color: #f6f8fa;
}

/* 代码块头部样式 */
:deep(.code-header) {
    display: flex;
    justify-content: space-between;
    align-items: center;
    background-color: #f5f5f5;
    padding: 8px 12px;
}

/* 语言标签样式 */
:deep(.code-language-label) {
    color: rgb(82 82 82);
    margin-left: 8px;
    font-size: 12px;
    line-height: 18px;
}

/* 代码高亮样式优化 */
:deep(.hljs) {
    background: transparent !important;
    padding: 0 !important;
}

/* 复制按钮样式 */
:deep(.copy-code-btn) {
    display: flex;
    align-items: center;
    gap: 4px;
    background: transparent;
    border-radius: 12px;
    padding: 0 8px;
    color: #586069;
    font-size: 12px;
    height: 28px;
    cursor: pointer;
    transition: all 0.2s ease;
}

:deep(.copy-code-btn.copied .copy-icon) {
    fill: #22c55e;
}

:deep(.copy-code-btn:hover) {
    background-color: rgb(0 0 0 / 4%);
}

:deep(.copy-icon) {
    fill: currentColor;
    flex-shrink: 0;
}

:deep(.copy-text) {
    white-space: nowrap;
}
</style>
