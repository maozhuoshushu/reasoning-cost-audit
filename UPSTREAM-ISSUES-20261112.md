# 上游问题单：推理参数、超时计费与用量字段语义

三份可直接提交的问题，均已脱敏（不含账号、密钥、内网地址、金额）。复现命令用占位符 <BASE_URL> / <KEY> / <MODEL>。

---

## 1｜同一个 reasoning_effort 字段，不同上游枚举互斥；不支持时不报错而是静默忽略

**现象**

同一网关后面的三家上游对 reasoning_effort 的合法值互不相同，且互相排斥：

| 上游 | 接受 | 拒绝 | 拒绝时表现 |
|---|---|---|---|
| A | off, low, medium, high, xhigh, max | none | 400 invalid_request |
| B | none, minimal, low, medium, high, xhigh, ultra, max | off | 400 unknown variant |
| C | low, high, max | 其余 | 400 |

交集只有 low / medium / high / xhigh / max —— **没有任何一个值能同时表达「关闭思考」**。

更严重的是 B 分支上 off 的行为：

- 请求 reasoning_effort=off → **HTTP 200**
- 返回 reasoning_tokens = 8，content 为空字符串
- 既不报错也不提示降级

**为什么这是问题**

静默忽略比 400 危险。用户以为关掉了思考，实际拿到空回复，只能靠下游解析失败才发现；表现为一批任务「成功但无内容」，定位花了两天。

**复现**

    curl -s <BASE_URL>/v1/chat/completions \
      -H 'Authorization: Bearer <KEY>' \
      -H 'Content-Type: application/json' \
      -d '{"model":"<MODEL>","messages":[{"role":"user","content":"ping"}],"reasoning_effort":"off"}'

对三家分别跑一次，记录 status 与 usage.reasoning_tokens。

**期望**

1. 文档列全每家支持的枚举。
2. 不支持的取值返回 4xx，不要 200 + 空 content。
3. 若需兼容旧值，请显式降级并在响应里回显最终生效的 effort。

---

## 2｜504 upstream_timeout 附带 billing_uncertain: true，却没有查询途径

**现象**

长请求被上游静默超过 180 秒后中断，返回：

    Error code: 504
    {'error': {'billing_uncertain': True, 'code': 'upstream_timeout',
               'message': '上游静默超过 180 秒，已中断且不自动重放（可能已计费）',
               'retryable': False}}

**问题**

- 明说可能已计费，但没有任何接口能确认这笔扣没扣。
- retryable: false 意味着不能自动重放；若已计费，等于为失败请求付费。
- 一次这样的失败，wall time 329 秒，占住一个并发槽近 5 分半。

**期望**

1. 明确：这类请求是否计费？按什么口径（发出即计 / 收到首字节计 / 不计）？
2. 若计费，请在用量接口里可查（哪怕只给 request_id 维度的标记）。
3. 若不计费，请把该字段去掉或置 false。

---

## 3｜usage.input 是否包含 cache_read，各家语义不一致

**现象**

按 usage 复算成本时，同一公式在不同上游算出不同结果，差别全在 usage.input 的语义：

- 解释一：input = 未缓存输入（cache_read 另计）
- 解释二：input = 总输入（含 cache_read）

命中率高的场景下，两种解释能差 30% 以上金额。我们为此改判过两次成本结论。

**期望**

1. 文档写明 input 是否含 cache_read（一句话即可）。
2. 或提供 uncached_input 与 cached_input 两个独立字段。

---

## 附：不算 bug 但值得写进文档的一条

缓存读单价相对输入单价的折扣率，各家差一个数量级：

| 模型 | 输入单价 | 缓存读单价 | 折扣 |
|---|---|---|---|
| 模型 X | 0.15 | 0.003 | 2% |
| 模型 Y | 0.15 | 0.03 | 20% |

结论：**换更便宜的模型之前，先看缓存折扣。** 命中率高的工作负载换到折扣差 10 倍的模型上会更贵——我们实测踩过这个坑。