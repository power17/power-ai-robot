<template>
    <div class="markdown-container">
        <div v-html="renderedContent"></div>
    </div>
</template>

<script setup>
import { ref, watch } from 'vue';
import MarkdownIt from 'markdown-it';

// 定义一个 content 字段，用于父组件传入 markdown 文本
const props = defineProps({
    content: {
        type: String,
        default: '',
    },
});

// 解析后的 HTML
const renderedContent = ref('');

// 初始化 MarkdownIt
const md = new MarkdownIt({
    html: true, // 允许解析 HTML 标签
    xhtmlOut: true, // 输出符合 XHTML 规范的标签（如 `<br />` 而不是 `<br>`）。默认 false。
    linkify: true, // 自动将文本中的 URL 转换为可点击的链接
    typographer: true, // 启用排版优化
    breaks: true, // 将单个换行符 (\n) 转换为 <br>
    langPrefix: 'language-', // 代码块的语言类名前缀（默认 'language-'）。例如 ```js 会生成 <pre><code class="language-js">
});

// 监听 content 字段，流式更新处理
watch(
    () => props.content,
    (newVal) => {
        if (newVal) {
            // 渲染为 HTML
            const html = md.render(newVal);
            renderedContent.value = html;
        }
    },
    { immediate: true }
);
</script>

<style scoped></style>
