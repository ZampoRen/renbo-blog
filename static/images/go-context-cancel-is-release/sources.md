# 图片与来源说明

- `cover.svg` / `cover.png`：作者自制解释图。用途：封面图，表达 “cancel 不是 timeout 按钮，而是 release 按钮”。
- `inline-01.svg` / `inline-01.png`：作者自制解释图。用途：正文三条线排查法，创建路径 / 取消路径 / 响应路径。
- 技术事实来源：本机 Go 1.25.4 标准库源码 `/opt/homebrew/Cellar/go/1.25.4/libexec/src/context/context.go`、`go doc context.*`、`go tool vet help lostcancel`、`go doc net/http/pprof`。

正文不写图片来源说明，避免发布痕迹；如微信公众号投递需要，default 发布门可使用本文件作为来源记录。
