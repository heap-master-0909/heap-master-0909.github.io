---
title: 개발 환경 세팅 (obsidian)
date: 2026-04-12 00:00:00 +0900
categories: [Etc, obsidian-env]
tags: [Tech, Etc, obsidian-env]
pin: true
---

## 추천 세팅

+ [chrome obsidian web clipper](https://chromewebstore.google.com/detail/cnjifjpddelmedmihgijeibhnjfabmlf?utm_source=item-share-cb) 같이 쓰면 좋음.
+ 사용법 : 소스를 raw에 넣고 ingest해줘라 하면 됨.
+ lint 시 필요없는 파일등을 정리해준다
+ [참고 youtube](https://www.youtube.com/watch?v=H9Wml5xDLLY)


## 추천 플러그인

### Smart Composer

claude 용

```
obsidian://show-plugin?id=smart-composer
```

---

### Terminal

```
obsidian://show-plugin?id=terminal
```

---

### Style Settings

width 조절용

[link](https://eoureo.tistory.com/6)

```
obsidian://show-plugin?id=obsidian-style-settings
```

---

### canvas mindmap

캔버스 미니맵

obsidian://show-plugin?id=canvas-mindmap

---

### shiki highlighter

코드 라인 하이라이터

obsidian://show-plugin?id=shiki-highlighter

---

### fast-text-color

텍스트 하이라이터

---

### css style 적용

* [link](https://eoureo.tistory.com/6)

settings > apperance > CSS Snippets

```css
/*
1. 노트의 frontmatter에 추가 (Add to the Frontmatter of the notes you want)
cssclass: my_style_width_100
2. "Settings ＞ Appearance ＞ CSS Snippets"에 설정 (Set to "Settings ＞ Appearance ＞ CSS Snippets")
my_style.css에 추가 (Add to my_style.css)
*/

/* Editing View */
.my_style_width_100 .cm-sizer,
.my_style_width_100 .cm-sizer .cm-contentContainer .cm-content,
.my_style_width_100 .cm-sizer .cm-contentContainer .cm-content .cm-line,
/* Reading View */
.my_style_width_100 .markdown-preview-sizer.markdown-preview-section,
.my_style_width_100 .markdown-preview-sizer.markdown-preview-section > div
{
  max-width: 100% !important;
  width: 100% !important;
  margin-left: initial !important;
  margin-right: initial !important;
}
```
