---
title: KaTeX and Jekyll
categories: coding
tags: web
---

Github Pages are [fixed](https://help.github.com/articles/configuring-jekyll/) to kramdown and its mathjax support.
But despite the name all it does is parse the text for [math blocks](https://kramdown.gettalong.org/syntax.html#math-blocks) and put them into `<script type="math/tex">` tags: https://kramdown.gettalong.org/math_engine/mathjax.html

So if you have to include MathJax yourself anyway you may just as well use the much faster $$\KaTeX$$ instead.

## $$\KaTeX$$

Unfortunately its documentation is very misleading as the [official site](https://katex.org/) only talks about server-side rendering and how to install it in Node.
The correct page is a bit [hidden](https://katex.org/docs/browser.html).

All it takes is a few lines of code for it to work as a drop-in MathJax replacement. Usually that code can be injected into Jekyll themes via some extension point like _includes/head/custom.html:
```html
  <head>
    <link rel="stylesheet" media="none" onload="media='all'" href="https://cdn[...]/katex.min.css">
    <script defer src="https://cdn[...]/katex.min.js"></script>
    <script defer src="https://cdn[...]/contrib/mathtex-script-type.min.js"></script>
  </head>
```
The `mathtex-script-type` extension scans for the script tags created by kramdown and triggers KaTeX.
Note how everything is deferred to speed up page loading. 

## Single dollar sign inline math

By default single dollar sign inline math is not recognized.
In MathJax you can achieve that with a special [config](http://docs.mathjax.org/en/latest/tex.html#tex-and-latex-math-delimiters).
They also support `\$` to actually get a dollar sign should you ever need one (currently not available in KaTeX).

With KaTeX you need to use an extension, the following snippet should do the job:
```html
<script defer src="https://cdn[...]/contrib/auto-render.min.js"
	onload="renderMathInElement(document.body, { delimiters: [ {left: '$', right: '$', display: false} ], throwOnError: false });"></script>
```

## Side Note

Usually it is also a good idea to turn on math preview in your `_config.yml` so your page won't be crippled in rss feeds, text browsers or when javascript is disabled (or like Wikipedia when loading of images is disabled). But this doesn't seem to work nicely with KaTeX:

```yml
kramdown:
  math_engine_opts:
    preview:         true
    preview_as_code: true
```

So this remains an open item.