---
title: about
layout: about
---

> 👋 Aloha, I'm Stuart, a Java developer. I'm here to share what I've learned, what's on my mind, and what I'm thinking about.

# Contact:

- E-Mail：stuartyangout@outlook.com

- GitHub：https://github.com/StuartYang



<script>
    // 获取图片元素
    const themeImage = document.getElementById('theme-image');

    // 根据 data-user-color-scheme 更新图片
    function updateImageBasedOnTheme() {
      const theme = document.documentElement.getAttribute('data-user-color-scheme');
      themeImage.src = theme === 'dark' ? 'https://raw.githubusercontent.com/StuartYang/Platane/output/github-contribution-grid-snake-dark.svg' : 'https://raw.githubusercontent.com/StuartYang/Platane/output/github-contribution-grid-snake.svg';
    }

    // 监听 data-user-color-scheme 的变化
    const observer = new MutationObserver((mutations) => {
      mutations.forEach((mutation) => {
        if (mutation.type === 'attributes' && mutation.attributeName === 'data-user-color-scheme') {
          updateImageBasedOnTheme(); // 更新图片
        }
      });
    });

    // 开始观察
    observer.observe(document.documentElement, {
      attributes: true, // 监听属性变化
    });

    // 初始化时检查主题
    updateImageBasedOnTheme();
  </script>