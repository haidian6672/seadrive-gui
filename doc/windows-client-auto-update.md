# Seadrive Windows 客户端自动升级

## 目标

使 seadrive windows 客户端能够定期检查是否有更新。 如果有更新, 则弹出对话框询问用户是否下载更新。

## 交互实现

最后要达到的效果是:

- 在设置对话框中添加一个选项 “自动检查版本更新”， 如果勾选上则定期检查更新。 默认勾选上。
- 如果有更新, 则询问用户是否下载更新, 如果用户点 “是” 则自动下载并安装更新

## 实现

使用 WinSparkle 这个库: https://github.com/vslavik/winsparkle

(它的接口是模仿 mac 下著名的 [Sparkle 框架](https://sparkle-project.org/) 来定义的, 后面 seadrive 的 mac 客户端自动升级可能会使用 sparkle 框架)

上面提到的定期检查更新、提示用户更新的功能，其实大部分已经被 winsparkle 这个库实现了，我们要做的就是调用它提供的函数就行了, 函数列表在这个[头文件里](https://github.com/vslavik/winsparkle/blob/master/include/winsparkle.h#L265)。

### 版本检查

sparkle/winsparkle 的版本检查其实是客户端向服务器的某个 URL 发一个请求, 服务器返回一个 XML 文档。具体的 XML 文档格式见 [这里](https://github.com/vslavik/winsparkle/wiki/Appcast-Feeds)。

客户端解析的部分也已经被 winsparkle 处理了，我们需要做的是在服务器端准备这个 XML 文档。在开发中可能需要在本地启动一个简单的 http 服务器来做测试用，比如搭一个简单的 nginx 服务器。