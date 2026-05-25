# 图片来源记录：go-context-not-a-cleaner

## 头图

- 文件：`/Users/zampo/Documents/renbo-blog/static/images/go-context-not-a-cleaner/cover.png`
- 源文件：`/Users/zampo/Documents/renbo-blog/static/images/go-context-not-a-cleaner/cover.svg`
- 类型：作者自制技术解释图
- 用途：文章封面/正文首图
- 说明：监控曲线 + cancel 通知隐喻，用于表达“超时已发生，但 goroutine 是否退出取决于业务代码是否响应”。

## 正文图

- 文件：`/Users/zampo/Documents/renbo-blog/static/images/go-context-not-a-cleaner/inline-01.png`
- 源文件：`/Users/zampo/Documents/renbo-blog/static/images/go-context-not-a-cleaner/inline-01.svg`
- 类型：作者自制技术解释图
- 用途：解释 cancelCtx 取消树
- 说明：父节点向下通知子节点，子节点取消不反向取消父节点；业务代码决定是否退场。

## 转换与检查

- SVG 转 PNG：`sips -s format png ...`
- 尺寸：cover 900x383，inline-01 1200x760
- 视觉检查：已检查无水印、无乱码、无明显文字越界或裁切。

图片来源说明只放在本文件和交付说明中，不写入正文。
