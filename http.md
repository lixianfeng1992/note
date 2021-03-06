# HTTP 1.1
- 支持 `connection: keep-alive`

## 链接建立过程
1. 把请求的URL放入队列
2. `DNS`解析域名的`IP`地址
3. 建立与目标主机的`TCP`连接
4. 如果是`HTTPS`, 初始化并完成TLS握手
5. 向对应的`URL`发送请求
6. 接受响应
7. 如果是主题`HTML`, 解析, 并针对页面中资源触发优先获取机制
8. 如果页面关键资源已经接收到, 开始渲染页面
9. 接收其他资源继续渲染直到结束

# 问题
1. 队头阻塞
2. 浏览器一般只支持6个并发连接
3. TCP利用率低(拥塞窗口调节)
4. 臃肿的消息首部
5. 受限制的优先级设置

# SPDY
- HTTP 的替代方案
- 多路复用
- 帧和首部压缩

# HTTP 2.0
- 基于SPDY

## 优势
1. 二进制协议
2. 首部压缩
3. 多路复用
4. 加密传输
5. 服务端推送
6. 支持优先级
