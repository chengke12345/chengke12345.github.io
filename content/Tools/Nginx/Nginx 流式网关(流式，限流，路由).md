# 1. 流式

「流式网关 Streaming Gateway」是一种中间件组件，<font color="#45ce6e">「SSE, Server-Sent Events」</font>是它常用的一种流式通信方式。

它描述的是一种架构模式：<font color="orange">API 网关或类似中间件作为代理，基于 SSE 协议，将后端(尤其是大模型)持续生成的增量数据，实时、可靠地流式推送给客户端</font>。

流式网关位于客户端与推理服务之间。模型陆续生成内容时，网关会持续接受并转发，让客户一遍等待一遍可以看到回答进度。所以它的基本模式是

```
模型生成一个token，返回给网关 --> 网关立即转发 --> 页面追加显示
```

这里的「流式」指的是：<u>一次请求的响应分多次发送给客户端，不必等整段回答生成完</u>。

SSE, Server-Sent Events 意思是「服务端发送事件」。它使用普通 HTTP，响应类型是 `text/event-stream`, 所以它本质上是一个协议，它让大模型服务器端，保持响应打开，不断发送事件。例如：

```
Content-Type: text/event-stream

data: {"delta": "你"}

data: {"delta": "好"}
```

每个事件用「空行分隔」，客户端读到一条就可以处理并显示，<font color="orange">一条事件可以包含多个 token, 并不是一定严格一个 token 一个 token 的封装成事件</font>。
叫做 SSE，是因为在这条响应流中，服务端持续向客户端发送事件。客户端发起请求后，接下来主要是持续接受。Web Socket 则是支持双方随时互相发送消息。

> [!info] 分段采用 SSE 格式
> 网关成为客户端和服务器端的中间件之后，它实际是将原本的连接，分成了两段独立的连接。即客端到网关，以及网关到大模型服务端。面对客户端网关就是服务端，面对推理后端，网关就是客户端。这两段是独立的 HTTP 请求(SSE就是使用的普通HTTP), 空行分隔事件的这种 SSE 格式，可以使用到两段，即
> ```
> 推理服务 --SSE--> 网关 --SSE--> 用户客户端 
> ```
> 后端到网关这一段采用什么格式，由暴露的接口决定。也可以设计成后端使用 gRPC, 再由网关转换成 SSE。如果后端采用 SSE， 网关可以直接转发，也可以解析后重新封装。整体架构为
> 
> ![](Pasted%20image%2020261103123024.png)


> [!warning] 注意
> SSE 规定的是服务端流式响应的格式。而请求端使用普通的 HTTP POST 操作就可以了，请求服务不使用 SSE。

客户端接受响应时，需要「持续的读取并解析 SSE」，不能把整个响应数据当作一份普通 JSON，然后用 `response.json()` 来一次解析。

## 流式网关配置

```
server {
	...
	proxy_buffering off;
	proxy_cache off;
	proxy_request_buffering off;
	chunked_transfer_encoding on;
	...
}
```

- `proxy_request_buffering` 控制的是客户端请求到达 nginx 服务器后，是否提供请求体临时缓存，还是立即转发请求给 vLLM 服务。
- `proxy_buffering` 是控制从 vLLM 返回 nginx 服务器后，是否提供响应体临时缓存。 
- `proxy_cache` 是控制 vLLM 返回 Nginx 服务器后，是否提供代理缓存，可供以后请求复用。

<font color="#eb4349">「关掉 proxy_buffering 是流式网关最核心的配置」</font>。

# 2. 限流

限流，可以按 IP，API Key 或者租户进行多维度的限流。下面以 IP 限流为例，

首先，需要设置一个「限流规则的共享状态区」存放限流规则，供网关执行。`limit_req_zone $binary_remote_addr zone=api:10m rate=5r/s;` ，其中的 `limit_req_zone` 就是定义限流规则的共享状态区。<font color="orange">它只定义 “怎么统计”，不会直接拦截请求，真正启用限流的是后面的</font>`limit_seq`

`limit_req zone=api burst=10 nodelay;` 限制同一 IP 可以瞬间发送超过 5 个请求(`rate=5r/s`)；最多可使用 10 个突发额度并立即转发(nodelay)。之后如果仍然过快，就返回限流错误。随着时间流逝，额度以约 5 请求/秒的速度恢复。`nodelay` 的含义是：突发请求不排队等待，而是立即放行；<font color="#45ce6e">「代价是突发额度耗尽后，后续请求会直接被拒绝」</font>。
所以，它表示每个IP长期平均速率为 5 个请求/s, 短期允许一个 burst，但是随着时间流逝，长期来看请求平均速度要保持在 5 个以内。

> [!example] lim_req 示例
> 假设配置 `rate=2r/s`, `burst=3`, 有一个 IP 原先没有积压，现在同时来了 5 个请求
>  ```
>  配置                                        请求的处理方式
>  limit_req zone=api burst=3;                 第1个请求立即通过，第2,3,4个等待                                                    0.5,1,1.5s,第5个请求被拒绝
>  limit_req zong=api burst=3 nodelay;          前4个立即通过，第5个被拒绝。
>  ```
>  注意⚠️：rate 规定的是可以同时处理的请求数，不是允许那么多请求同时立即通过，每个请求默认间隔 500ms(1000ms / 2 req) 进入系统。只有设置了 nodelay 才会放行同时立即通过。这个默认间隔与是否超额没有关系，超额只管是否进行限额处理，是否默认间隔由 nodelay 设置决定。 默认间隔大小由 rate 决定
>  所以，<font color="orange">rate 控制持续速率。burst 控制额外容忍量，nodelay 决定超额允许范围内的请求能否立即通过。</font>

注意⚠️：限流只限制进入后端的速率，不能保证业务端到端的改善。

# 3. 路由

```
upstream vllm-active {

zone llm_backend 64k;
resolver 127.0.0.11 valid=5s ipv6=off;
resolver_timeout 2s;

server vllm-main:8000
	resolve
	max_fails=1
	fail_timeout=10s;

server vllm-backup:8000
	resolve
	backup
	max_fails=1
	fail_timeout=10s;

keepalive 32;
}

server{
	location /v1/ {...proxy_pass http://vllm-active;...}
	location = /health {... proxy_pass http://vllm-active/health;...}
}
```

通过 location, 我们可以将请求转发到不同的后端服务器上，这里设置通过 上传流量 upstream vllm-active 将转发给后端被代理的 vLLM 模型服务器。正常情况下，转发到 vllm-main 主模型服务器上，如果主模型无法访问，就转发给备用模型。 在多模型系统中，还可以根据请求的参数信息，将不同请求发送给不同的模型进行推理。

> 在前端看来，它们都是统一入口，统一流量限制规则，可以提交参数指定推理模型，或者在故障时自动切换模型。未来多模型场景下，或者加设后端服务器时，对前端用户透明。用户对系统的使用，几乎不受影响。




