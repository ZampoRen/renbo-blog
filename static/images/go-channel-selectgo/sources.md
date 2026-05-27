# go-channel-selectgo 图片来源

本文图片为作者自制 SVG/PNG 示意图，用于解释 Go runtime/select.go 中 selectgo 的执行流程。

## 文件

- cover.svg / cover.png：作者自制封面图
- inline-01.svg / inline-01.png：作者自制正文解释图

## 依据

图中概念来自本地 Go 1.25.4 源码：

- /opt/homebrew/Cellar/go/1.25.4/libexec/src/runtime/select.go

关键事实：

- nil channel case 会被从 poll order 和 lock order 中省略
- poll order 使用 cheaprandn 生成随机化扫描顺序
- lock order 按 hchan 地址排序
- 阻塞 select 会为每个 case 创建 sudog 并挂入 sendq / recvq
- 唤醒后会清理未命中的 sudog

## 授权/署名风险

无外部图片素材；无第三方版权署名要求。
