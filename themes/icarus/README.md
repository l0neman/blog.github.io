# icarus 主题配置

## 改变内容宽高

- 更改每种屏幕的相对尺寸配置

themes/icaus/include/style/responsive.styl
themes/icaus/include/style/base.styl

- 根据栏目数量更改中间内容宽度或旁边的挂件

themes/icarus/layout/layout.jsx

```html
 <!-- 更改中间内容宽度 -->
 <div class={classname({
     column: true,
     'order-2': true,
     'column-main': true,
     'is-12': columnCount === 1,
-    'is-8-tablet is-8-desktop is-8-widescreen': columnCount === 2,
+    'is-8-tablet is-8-desktop is-9-widescreen': columnCount === 2,
     'is-8-tablet is-8-desktop is-6-widescreen': columnCount === 3
```

themes/icarus/layout/common/widgets.jsx

```html
 // 更改组件宽度
 function getColumnSizeClass(columnCount) {
     switch (columnCount) {
         case 2:
-            return 'is-4-tablet is-4-desktop is-4-widescreen';
+            return 'is-4-tablet is-4-desktop is-3-widescreen';
         case 3:
             return 'is-4-tablet is-4-desktop is-3-widescreen';
```

尺寸更改参考：https://bulma.io/documentation/columns/sizes/



## 添加文章目录

在文章头部添加 `toc: true`。

```post.md
title: 一篇有目录的文章
toc: true
---
文章内容...
```



## 添加 read more 按钮

在文章合适位置插入 `<!-- more -->`