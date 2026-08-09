# Dify 源码部署的工作流协作排障

## "同步数据中"是实时协作未就绪，不是 Celery 队列堵塞

**问题**：工作流编辑器持续显示“同步数据中，只需几秒钟”，即使 API 和 Celery Worker 进程均在线。

**做法**：先检查浏览器到 `/socket.io/` 的请求，而不是只检查 Celery。自建域名或反向代理部署时，完整配置应满足：

```text
ENABLE_COLLABORATION_MODE=true
NEXT_PUBLIC_SOCKET_URL=wss://<公开域名>
Gunicorn application: app:socketio_app
Gunicorn worker class: geventwebsocket.gunicorn.workers.GeventWebSocketWorker
```

Nginx 必须把 `/socket.io/` 转发到 API，并传递 `Upgrade` 与 `Connection: upgrade` 请求头。确认 `nginx` access log 中的 WebSocket 握手状态为 `101`，最后刷新实际工作流页面，确认提示消失。

**原因**：该提示表示工作流 CRDT 协作图尚未建立可信连接。普通 `app:app` 不会挂载 Socket.IO；普通 Gunicorn `gevent` worker 也无法处理 WebSocket 升级。两种情况都会让 API 的普通 HTTP 请求看起来正常，但协作连接持续失败。

## 不要把关闭协作模式作为首选修复

`ENABLE_COLLABORATION_MODE=false` 可以作为无法提供 WebSocket 时的功能降级开关，但应先修复协作服务配置。

**原因**：关闭后会回退到旧的草稿保存路径；在 Web 构建与 API 源码版本不一致的部署中，该路径可能再暴露参数协议不兼容，造成编辑保存失败。官方协作部署文档要求在自定义域名和反向代理场景下保持协作模式开启，并正确配置 WebSocket worker。

## 前端构建必须保持官方命令和版本一致

使用 `pnpm build` 生成线上 Web 构建。若构建报模块缺失等编译错误，应先恢复完整、匹配版本的 Web 源码树，不能以其他构建器或失败产物替代。

**原因**：部署的静态前端会固化运行时连接地址和 API 协议。源码不完整或 Web/API 版本不一致会同时导致错误的 Socket 地址、旧接口参数和兼容路由需求；失败构建产物不能安全切换到线上服务。
