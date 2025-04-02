---
# general config
title: Typecho中代码高亮并有Mac风格
subtitle: 
date: 2023-12-10
authors:
  - viogami: author.png

# # card specific config
# summary: A summary of the blog post
# cardimage: photo1_card.jpeg

# # post specific config
# featureimage: photo1.jpeg
# caption: Some image caption
# toc: true
---
## 三步

- 下载prism文件。
- 在head中引用，并添加行号类。
- 修改CSS。

<!--more-->

### 第一步：官网下载，选择语言和需要的插件

易得

### 第二步：header.php文件的head块中添加如下代码

```php
    <!-- 为pre块添加 line-numbers 类 -->
    <script>
        document.addEventListener('DOMContentLoaded', function () {
            var codeBlocks = document.querySelectorAll('.post-content pre:not(.line-numbers)');
            codeBlocks.forEach(function (codeBlock) {
                codeBlock.classList.add('line-numbers');
            });
        });
    </script>
    <!-- prism 代码高亮&主题选择 -->
    <!-- 注意prismjs_theme 是我在function文件中定义的 -->
    <?php if ($this->options->prismjs_theme == 'tomorrow') : ?>
    <link rel="stylesheet" href="<?php $this->options->themeUrl('替换为你的prism.css的路径，下同'); ?>" />
    <script src="<?php $this->options->themeUrl('替换为你的prism.js的路径，下同'); ?>"></script>
    <?php elseif ($this->options->prismjs_theme == 'coy') : ?>
    <link rel="stylesheet" href="<?php $this->options->themeUrl('prismjs/coy/prism.css'); ?>" />
    <script src="<?php $this->options->themeUrl('prismjs/coy/prism.js'); ?>"></script>
    <?php endif; ?>
    <!-- 修改为mac样式,请替换你的mac.css文件路径-->
    <link rel="stylesheet" href="<?php $this->options->themeUrl('css/mac.css'); ?>" />
```

### 第三步：修改满意的CSS

下面只写了 `mac.css`文件。
**注意，我的效果也修改了下载了 `prism.css`文件。**
更具体的css可以参考我的github仓库，该主题是开源的。

```css
/* macOS 风格的代码框样式 */
pre[class*=language-] {
    position: relative;
    padding: 20px;
    border-radius: 8px; 
    box-shadow: rgba(0, 0, 0, 0.2) 0px 5px 20px;
    transition: box-shadow 0.3s ease-in-out;
    max-height: 100%;
    max-width: 85%;
    padding-top: 50px;
}

/* macOS 风格的图标 */
pre[class*=language-]::before {
    content: "";
    position: absolute;
    background: url("../img/icon_mac.svg");
    background-position-y: center;
    top: 15px;
    left: 20px;
    height: 14px;
    width: 54px;
    margin-left: 5px;
    display: block;
}

/* 悬浮 */
pre[class*=language-]:hover {
    box-shadow: 0 25px 15px rgba(0, 0, 0, 0.2);
  }
```

## 你也可以直接从我的仓库拉取源代码！👉👉**[Ayakin](https://github.com/viogami/Ayakin-TypechoTheme)**

<script src="https://giscus.app/client.js"
        data-repo="viogami/blog"
        data-repo-id="R_kgDOORWDyA"
        data-category="Announcements"
        data-category-id="DIC_kwDOORWDyM4Conxc"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="top"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
        crossorigin="anonymous"
        async>
</script>
