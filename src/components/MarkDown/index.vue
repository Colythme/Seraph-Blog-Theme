<!-- Source: Chanshiyu -->
<!-- Modified -->

<template>
  <!-- <highlightjs language = "python" code="var x = 5;" /> -->
  <div class="markdown" v-html="html"></div>
</template>

<script>
import marked from 'marked'
import Zooming from 'zooming'
import ClipboardJS from 'clipboard'
import katex from 'katex'
import 'katex/dist/katex.css'
import hljs from '/public/lib/highlight'
import { fileCDN, handleImg } from '/src/utils/index.js'

function mathsExpression(expr) {
  if (expr[0] != '$') return ;
  if (expr.match(/^\$\$[\s\S]*\$\$$/)) {
    expr = expr.substr(2, expr.length - 4)
    return katex.renderToString(expr, { displayMode: true })
  } else if (expr.match(/^\$[\s\S]*\$$/)) {
    expr = expr.substr(1, expr.length - 2)
    return katex.renderToString(expr, { isplayMode: true })
  }
}

const zooming = new Zooming({
  bgOpacity: 0.8,
  zIndex: 100,
})

let IMG_ID = 0
let CODE_ID = 0
const CODE_COPY_LIST = []

const renderer = new marked.Renderer()

renderer.heading = function (text, level, raw, slugger) {
  return `<h${level} id="h-${slugger.slug(raw)}"><span class = "pound">＃</span>${text}</h${level}>`
}

renderer.image = function (href, title, text) {
  IMG_ID++
  const id = `img-${IMG_ID}`
  const cdnHref = fileCDN(href)

  const img = new Image()
  img.src = cdnHref
  const compelete = (isSuccess) => {
    const dom = document.getElementById(id)
    dom.src = isSuccess ? cdnHref : href
    dom.style.opacity = 1
  }
  img.onload = () => compelete(true)
  img.onerror = () => compelete(false)

  const { style, isFull } = handleImg(cdnHref)

  return `<span class="img-box ${isFull ? 'full' : ''}">
  <span class="bg" style="${style}">
    <span class="loading">
      <span class="dot"></span><span class="dot"></span><span class="dot"></span><span class="dot"></span><span class="dot"></span>
    </span>
  </span>
  <img id="${id}" class="img-zoomable cursor" style="${style}"  loading="lazy" alt="${text}" />${
    text ? `<span class="title">◭ ${text}</span>` : ''
  }</span>`
}

renderer.table = function (header, body) {
  if (body) body = `<tbody>${body}</tbody>`
  return `<div class="table-wrapper">\n<table>\n<thead>\n${header}</thead>\n${body}</table>\n</div>\n`
}

const rendererCodespan = renderer.codespan
renderer.codespan = function (text) {
  const math = mathsExpression(text)

  if (math) {
    return math
  }

  return rendererCodespan(text)
}

const rendererCode = renderer.code
renderer.code = function code(_code, infostring, escaped) {
  // katex 支持， https://github.com/markedjs/marked/issues/1538#issuecomment-526189561
  // if (!lang) {
    const math = mathsExpression(_code)
    // console.log ("AAA" + math + _code);
    if (math) {
      return math
    }
  // }

  CODE_ID++
  const id = `code-${CODE_ID}`
  CODE_COPY_LIST.push({ id, code: _code })

  const lang = (infostring || '').match(/\S*/)[0]

  // if (this.options.highlight) {
  //   const out = this.options.highlight(_code, lang)
  //   if (out != null && out !== _code) {
  //     escaped = true
  //     _code = out
  //   }
  // }
  console.log ("lang", _code);
  if (!lang) {
    return `<pre><code>${_code}</code><i id = "${id}" class = "far fa-copy"></i></pre>`
  }

  // return `<pre><div class = "gutter"><div class = "light"></div><div class = "language">${lang}</div></div><div class = "code_wrapper"><code class = "${
  //   this.options.langPrefix + lang
  // }">${_code}<i id = "${id}" class = "far fa-copy"></i></code></div></pre>\n`

  return `<pre v-highlightjs><code>${_code}</code></pre>`
  // return `<highlightjs language = "python" code="var x = 5;" />`
  // return `<pre><code class = "language-python">${_code}</code></pre>`
}

marked.setOptions({
  renderer,
  highlight: (code) => hljs.highlightAuto(code).value,
})

export default {
  name: 'MarkDown',
  props: {
    content: {
      type: String,
      default: '',
    },
    target: {
      type: String,
      default: '',
    },
  },
  data() {
    return {
      html: '',
      clipboards: [],
    }
  },
  created() {
    this.marked()
  },
  watch: {
    content() {
      this.marked()
    },
  },
  methods: {
    marked() {
      this.CODE_COPY_LIST = []
      this.html = marked(this.content)

      // 对于只是纯解析文字不需要代码高亮和灯箱
      if (!this.target) return
      // console.log ("AAA" + this.target);
      this.$nextTick(() => {
        hljs.highlightAll({ target: this.target })
        zooming.listen('.img-zoomable')
        this.bindCodeCopy()
      })
    },
    bindCodeCopy() {
      CODE_COPY_LIST.forEach((o) => {
        const clipboard = new ClipboardJS('#' + o.id, {
          text(trigger) {
            return o.code
          },
        })
        this.clipboards.push(clipboard)
      })
    },
    unBindCodeCopy() {
      this.clipboards.forEach((o) => {
        o.destroy()
      })
    },
  },
  beforeDestroy() {
    zooming.close()
    this.unBindCodeCopy()
  },
}
</script>

<style lang="scss" scope>
@import './index.scss';
</style>
