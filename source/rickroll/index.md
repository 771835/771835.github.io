---
title: 正在加载...
date: 0721-13-14 00:00:00
comments: false
---

<div style="text-align: center; margin-top: 50px;">
    <h2>正在为您跳转到精彩内容...</h2>
    <p>如果没有自动跳转，请刷新页面。</p>
</div>

<script>
setTimeout(function() {
    try {
        // 方法1：检查时区是否为中国时区 (东八区)
        const timezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
        
        // 方法2：检查浏览器语言
        const language = navigator.language || navigator.userLanguage;

        // 判断逻辑：如果是中国时区，或者浏览器语言是中文，则判定为国内用户
        const isChina = (timezone === 'Asia/Shanghai' || timezone === 'Asia/Chongqing' || timezone === 'Asia/Urumqi') || 
                        (language && language.toLowerCase().startsWith('zh'));

        if (isChina) {
            // 国内跳转 B站
            window.location.href = 'https://www.bilibili.com/video/BV1GJ411x7h7';
        } else {
            // 国外跳转 YouTube
            window.location.href = 'https://www.youtube.com/watch?v=dQw4w9WgXcQ';
        }
    } catch (e) {
        // 极端情况兜底：默认去 YouTube
        window.location.href = 'https://www.youtube.com/watch?v=dQw4w9WgXcQ';
    }
}, 800);
</script>

