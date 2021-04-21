---
title: Material Design 参考
toc: true
date: 2021-04-21 22:49:48
categories:
- 参考文档
tags:
- Metrial
---
## Elevation

（海拔）

### Component elevation values（组件高度值）:

| Compont    | Elevation  |
| ---------- | ---------- |
| Nav drawer | 16dp       |
| App bar    | 4dp        |
| Card       | 6dp        |
| FAB        | 6dp        |
| Button     | 2dp to 8dp |
| Dialog     | 24dp       |


<!-- more -->

### Table of default elevation values（默认高度表）：

| Component            | Defualt elevation values (dp) |
| -------------------- | ----------------------------- |
| DIalog                                          | 24 |
| Modal bottom sheet                              | 16 |
| Modal side sheet                                | 16 |
| Navigation drawer                               | 16 |
| Floatig action button (FAB -pressed)            | 12 |
| Standard bottom sheet                           | 8  |
| Standard side sheet                             | 8  |
| Bottom navigation bar                           | 8  |
| Bottom app bar                                  | 8  |
| Menus and sub menus                             | 8  |
| Card (when picked up)                           | 8  |
| Contained button (press state)                  | 8  |
| Floating action button (FAB - resting elvation) | 6  |
| Snackbar                                        | 6  |
| Top app bar (scrolled state)                    | 4  |
| Top app bar (resting elevatino)             | 0 or 4 |
| Refresh indicator                               | 3  |
| Search bar (resting elevaion)                   | 3  |
| Card (resting elevation)                        | 1  |
| Switch                                          | 1  |
| Text button                                     | 0  |
| Standard side sheet                             | 0  |


## Layout

（布局）

### Spacing methods

（间距方法）

#### Baseline grid

8dp 网格：所有组件均和手机、平板和台式机的 8dp 正方形基准网格对齐。
4dp 网格：组件中的图标、类型和某些元素可以与 4dp 网格对齐。

组件尺寸建议


#### Spacing

（间距）

- Padding

（填充）

填充指 UI 元素之间的空间。填充是基准线的另一种间距方法，以 8dp 或 4dp 的增量进行度量。

例如：组件之间具有 24dp 填充的布局。

- Dimensions

（尺寸）

某些组件（例如应用程序栏或列表）仅概述元素的高度。这些元素的高度应与 8dp 网格对齐。

Status bar height: 24dp
App bar height: 56dp
List item height: 88dp

#### Containers and ratios

（容器与比例）

- Aspect ratios

（长宽比）

建议在 UI 中使用如下纵横比：16:9，3:4，2:4，3:1，4:2，2:3。

### Touch targets 

（触摸目标）

触摸目标应至少为 48 x 48 dp，目标之间至少要有 8dp 的空间。

## Applying color to UI

（将色彩应 用于 UI）

### Top and bottom app bars

（顶部和底部应用栏）

#### Identifying app bars

（识别应用栏）

顶部和底部的应用栏使用应用程序的原色。系统栏可以使用原色的深色或浅色变体来将系统内容与顶部应用栏内容分开。

例如：顶部应用栏上使用原色（purple 500），系统栏上使用深色主色（purple 700）。

要强调应用栏和其他表面之间的区别，请在附近的组件（例如浮动操作按钮（FAB））上使用辅助颜色。

例如：底部应用栏上使用原色（blue 700），而在浮动操作按钮上使用第二色（orange 500）。

## Typography

（版式）

### The type system

（类型系统）

#### Type Scale

（类型量表）

类型量表是类型系统支持的 13 种样式的组合。

Material Design type scale:

| Scale Category  | Typeface | Font    | Size | Case     | Letter spacing |
| --------------- | -------- | ------  | ---- | -------- | -------------- |
| H1              | Roboto   | Light   | 96   | Sentence | -1.5           |
| H2              | Roboto   | Light   | 60   | Sentence | -0.5           |
| H3              | Roboto   | Regular | 48   | Sentence | 0              |
| H4              | Roboto   | Regular | 34   | Sentence | 0.25           |
| H5              | Roboto   | Regular | 24   | Sentence | 0              |
| H6              | Roboto   | Medium  | 20   | Sentence | 0.15           |
| Subtitle 1      | Roboto   | Regular | 16   | Sentence | 0.15           |
| Subtitle 2      | Roboto   | Medium  | 14   | Sentence | 0.1            |
| Body 1          | Roboto   | Regular | 16   | Sentence | 0.5            |
| Body 2          | Roboto   | Regular | 14   | Sentence | 0.25           |
| BUTTON          | Roboto   | Medium  | 14   | All caps | 1.25           |
| Caption         | Roboto   | Regular | 12   | Sentence | 0.4            |
| OVERLINE        | Roboto   | Regular | 10   | All caps | 1.5            |

## 色彩总结

material design color

primary
primary dark
primary light

secondary
secondary dark
secondary light

原则：

1. 从上至下，由深至浅。
2. 由主色彩至副色彩
3. 仅有主色彩则替换副色彩
4. 副色彩做为动作控件的色彩
5. 如果多个动作控件，那么副色彩上下要形成色彩落差。

色彩选择参考表格：

| 控件         | 只有主色彩    | 包含副色彩      |
| ------------ | ------------- | --------------- |
| 状态栏       | primary dark  | <=              |
| 标题栏       | primary       | <=              |
| 动作按钮     | primary       | secondary       |
| 滑杆球       | primary       | secondary       |
| 滑杆         | primary       | secondary       |
| 背景         | primary light | secondary light |
| tab 按钮     | primary light | <=              |
| tab 指示线   | primary       | secondary       |
| 按钮         | primary       | secondary       |
| 开关球       | primary light | secondary light |
| 开关杆       | primary       | secondary       |
| 列表点击背景 | primary       |                 |

