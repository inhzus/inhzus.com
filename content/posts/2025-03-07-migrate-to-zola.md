+++
title = "从 Hugo 迁移至 Zola"
date = "2025-03-08T10:50:00+08:00"

+++
  
本博客的主题之前是基于 [futu](https://github.com/yuanji-dev/futu) 魔改而来，不是自己写的代码，条理梳理得就比较差。借这次迁移 SSG 重新梳理了代码，并记录下遇到的问题。

## Zola vs. Hugo

实际体验下来，[Zola](https://www.getzola.org/) 相比 Hugo 并没有显著的提升。

最可圈可点的性能，只有在博客数量过多时，才能感受到明显的差异。

在社区方面，Hugo 可查询到的资料和插件非常丰富，相比之下 Zola 则需要一些动手能力才能搞定。且目前看起来 zola 的 GitHub repo 并不是非常活跃，待解决的问题数量不少，很容易找到贡献的机会。

### Tera

基于 Rust 的 [Tera](https://keats.github.io/tera/) template 语言，在语法上和 Hugo 支持的 golang text/template & html/template 基本类似。

不过，Tera 对「[继承](https://keats.github.io/tera/docs/#inheritance)」的支持更加完善，支持在子页面 `extend` 父页面，定制其中一部分 block （不论是一系列变量，还是页面内容），且可以通过 `super()` 方法调用父 block 逻辑。

于是可以做到：

```html
<!-- base.html -->
{% block init %}
  {% set title = config.title %}
  {% set author = config.author %}
{% end block %}

<title>{{ title }}</title>
<meta author="{{ author }}"/>

<!-- post.html -->
{% block init %}
  {{/* super() */}}
  {% set title = page.title ~ " | " ~ config.title %}
{% endblock %}
```

在 `base.html` 中声明需要使用到的变量；在对应到不同的实现中（如 index/post/archive），调用 `super()` 接受大多数默认变量，修改当前页面需自定义的部分。

通过以上办法，不必再像大多数 Hugo 主题一样，在一份 html 中，进行所有页面类型的分类判断，将大量的代码堆在一起，之后又由于代码过长拆成若干个模板文件（如 header.html，meta.html，甚至 description.html）七零八落，维护起来相当痛苦。

## 问题

### Permalink

Zola 框架提供的绝大多数链接都是 permalink，也就是都要带有 `base_url`。举例如 `base_url = https://example.com/blog`，文章的相对链接为 `/posts/welcome.md`，透过 `get_url(path="@/posts/welcome.md")` 只能获取到 `https://example.com/blog/posts/welcome/`，而无法方便地获取到到根路径的相对路径 `/blog/posts/welcome/`。

### Syntax Highlighting

Zola 的代码高亮主题配置与 Hugo 不同，如果需要自定义主题，需要使用 textmate `.tmTheme`  或 sublime `.sublime-theme` 配置，或基于自定义的一套 css class 实现相应的高亮风格，迁移起来需要一定的成本。好在其内置了我比较喜欢的 GitHub 风格。

### Builtin Functions

Zola 缺乏诸多常用的内置函数或模板，如 google analystics、git info、TOC 等，均依赖于用户自己实现。

且仅从配置项就可看出 zola 在某些功能上缺乏自定义选项，如 markdown、goldmark、footnotes 等等。

## 总结

不过总的来说，zola 还是能够胜任我的这种比较简洁风格博客的使用。

我完全重构版本的主题 repo：[zola-futu](https://github.com/inhzus/zola-futu)
