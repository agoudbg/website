# 关注点四：流量限制

Telegram 对你的 bot 每秒钟能发送多少条信息进行了限制。
这意味着你的任何 API 请求都可能会出现错误，并显示状态代码 429（Too Many Requests）和 [此处](https://core.telegram.org/bots/api#responseparameters) 规定的 `retry_after` 响应参数。
这随时都可能发生。

处理这些情况只有一种正确的方法：

1. 等待指定的秒数。
2. 重试请求。

幸运的是，有一个 [插件](../plugins/auto-retry) 可以做到这一点。

该插件 [非常简单](https://github.com/grammyjs/auto-retry/blob/main/src/mod.ts)。
它实际上只是暂停一段时间然后重试。
然而，使用它主要是有一个潜在假设：**任何请求都可能很慢**。
这意味着当你以 webhook 的方式运行你的 bot 时，不管你做什么，[你都不得不使用一个队列](../guide/deployment-types#及时结束-webhook-请求)，否则你需要配置自动重试插件并确保它不会花费太多时间，但这样一来你的 bot 可能会跳过一些请求。

## 确切的限制是什么

它们是未规定的。

处理它。

我们对关于你能处理多少请求有一些好想法，但确切的数字是未知的。
（如果有人告诉你实际限制，说明他们并不了解情况。）
这些限制并不是你通过尝试 Bot API 可以找出的硬性阈值。
相反，它们是根据你的 bot 的确切请求载荷、用户数量和其他因素变化的灵活约束，其中并非所有因素都是已知的。

以下是一些关于速率限制的误解和错误假设。

- 我的 bot 太新了，不会收到洪水等待错误。
- 我的 bot 没有足够的流量，不会收到洪水等待错误。
- 我的 bot 的这个功能使用得不够频繁，不会收到洪水等待错误。
- 我的 bot 在 API 调用之间留有足够的时间，所以不会收到洪水等待错误。
- 这个特定的方法调用不会收到洪水等待错误。
- `getMe` 不会收到洪水等待错误。
- `getUpdates` 不会收到洪水等待错误。

所有这些都是错误的。

让我们来看看我们 _确切_ 知道的事情。

## 关于速率限制的安全假设

从 [Bot FAQ](https://core.telegram.org/bots/faq#my-bot-is-hitting-limits-how-do-i-avoid-this) 中，我们知道有一些限制是不能超过的。

1. _“在特定聊天中发送消息时，请避免每秒发送多于一条消息。我们可能允许短暂突发超过此限制，但最终你将开始收到 429 错误。”_

   这点应该很清楚。自动重试插件会为你处理这个问题。

2. _“如果你要向多个用户发送批量通知，API 不会允许每秒超过 30 条左右的消息。为获得最佳结果，请考虑将通知分散在 8-12 小时的较长时间间隔内。”_

   **这仅适用于批量通知**，即主动向多个用户发送消息。
   如果你只是回复用户的消息，则每秒发送 1000 条或更多消息都没有问题。

   当 Bot FAQ 提到你应该 _“考虑将通知分散在较长时间间隔内”_ 时，这并不意味着你应该添加任何人为的延迟。
   相反，这里的主要要点是发送批量通知是一个需要花费很多个小时的过程。
   你不能期望立即同时向所有用户发送消息。

3. _"还要注意，你的 bot 每分钟无法向同一群组发送超过 20 条消息。"_

   同样，非常清楚。
   这完全与批量通知或群组中发送的消息数量无关。
   并且同样，自动重试插件会为你处理这个问题。

在官方 Bot API 文档之外还披露了一些其他已知限制。
例如，[已知](https://t.me/tdlibchat/146123) bot 每分钟只能在每个群聊中进行最多 20 条消息编辑。
然而，这是例外，我们还必须假设这些限制将来可能会发生变化。
因此，此信息不会影响如何编写你的 bot。

例如，基于这些数字限制你的 bot 仍然是一个坏主意：

## 限流

有些人认为遇到速率限制是不好的。
他们更愿意知道确切的限制，以便可以对他们的 bot 进行限流。

这是不正确的。
速率限制是用于洪水控制的有用工具，如果你采取相应的措施，它们不会对你的 bot 产生任何负面影响。
也就是说，遇到速率限制不会导致封禁。
忽视它们才会。

而且，根据 [Telegram](https://t.me/tdlibchat/47285) 的说法，知道确切的限制是“无用和有害”的。

_无用_ 是因为即使你知道限制，你仍然必须处理洪水等待错误。
例如，Bot API 服务器在维护期间关机重新启动时返回 429。

_有害_ 是因为如果你为了避免达到限制而人为延迟某些请求，你的 bot 的性能将远非最佳。
这就是为什么你应该始终尽快发送请求，但要尊重所有的洪水等待错误（使用自动重试插件）。

但是，如果限制请求是不好的，那么如何进行广播呢？

## 如何进行消息广播

可以采用非常简单的方法进行广播。

1. 向用户发送一条消息。
2. 如果收到 429 错误，则等待并重试。
3. 重复。

不要添加人为的延迟。
（这会使广播速度变慢。）

不要忽视 429 错误。
（这可能导致封禁。）

不要同时发送多条消息。
（你可以同时发送非常少量的消息（可能是 3 条左右），但这可能难以实现。）

上述列表中的步骤 2 由自动重试插件自动完成，代码如下所示：

```ts
bot.api.config.use(autoRetry());

for (const [chatId, text] of broadcast) {
  await bot.api.sendMessage(chatId, text);
}
```

这里有意思的部分是 `broadcast` 是什么。
你需要将所有聊天记录存储在某个数据库中，并且需要能够缓慢地获取它们。

目前，你必须自己实现此逻辑。
将来，我们想创建一个广播插件。
我们很乐意接受你的贡献！
[从这里](https://t.me/grammyjs) 加入我们。

### 我可以付费提高速率限制吗？

可以。

> 仅当你的 bot 余额中至少有 10,000 个 Telegram Stars 时，此部分才有意义。

当你广播大量消息时，你会给 Telegram 基础设施带来大量负载。
因此，如果你希望 Telegram 提高限制，你需要为你产生的流量补偿他们。
（最有可能的是，你还需要额外支付一点费用。）

[付费广播](https://core.telegram.org/bots/api#paid-broadcasts) 让你可以使用 [Telegram Stars](https://t.me/BotNews/90) 中的余额来增加你的 bot 速率限制。
然后，你将能够 **每秒发送最多 1000 条消息**。

1. 使用 [@BotFather](https://t.me/BotFather) 启用 _付费广播_。
2. 你可以使用与普通广播相同的代码。
   毕竟，你仍然需要以同样的方式遵守速率限制，即使现在的限制要高得多。
   然而，你需要并发执行多个 API 调用才能获得更高的吞吐量。
3. 在每个 API 调用中指定 `allow_paid_broadcast`。

步骤 2 意味着你应该使用一个队列，该队列可以让你以一定程度的并发性执行任务。
如果你使用的并发太少，你的吞吐量将低于每秒 1000 条消息。
如果你使用太多，你会更快地遇到速率限制。
另外，如果你有很多并发调用 `sendMessage` 的请求，并且其中一个请求返回了 429，那么其他所有的发出请求将忽略这个速率限制。
因此，Telegram 会对你施加更长的冷却时间。

合适的并发调用数量可以通过查看发送消息的平均时间来选择。
例如，如果这个平均值是 57 毫秒，你应该始终尝试保持 57 个并发调用 `sendMessage`。
