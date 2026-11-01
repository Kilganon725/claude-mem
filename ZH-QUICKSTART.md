# claude-mem 中文快速上手

> 来源：上游项目 thedotmack/claude-mem（Apache-2.0）。本文是本 fork 自写的中文
> 使用说明，不是上游官方文档；命令与行为以当天上游 README 为准。

## 一句话理解

claude-mem 给 Claude Code 加了一个“长期记忆”：它自动记录每次会话里做了
什么、决定了什么，下次会话开始时把相关的记忆找回来注入上下文，Agent 不用
每次都从零开始问你。

## 安装

最直接的一条命令：

```
npx claude-mem install
```

安装器会先装好一切，再请你在浏览器登录 claude-mem（邮箱 magic link，
不需要银行卡），开通托管 observer（30 天免费试用，到期后自动回落到你
自己的 Anthropic 套餐）。想完全本地、不登录也可以：传明确的 `--provider`
参数、设 `CLAUDE_MEM_ONLINE_OPTIN=false`，或在 CI / 非交互 shell 里跑，
安装器就不会要求账号交互。

也可以在 Claude Code 会话内用插件方式安装：

```
/plugin marketplace add thedotmack/claude-mem
/plugin install claude-mem
```

然后重启 Claude Code。下一次会话结束后，再下一次会话就会自动带上之前的
上下文——记忆是“隔一次会话”才开始生效的，不要装完立刻期待它回忆本次内容。

## 千万别踩的坑

`npm install -g claude-mem` 只装 SDK/库，**不会**注册 hooks，也不会启动
worker 服务，装完看起来成功了但实际什么都不记录。一定要走上面的插件安装
路径（或上游 README 给出的 `npx` 安装路径）。

## 它是怎么记的（四步）

1. **Hooks 抓事件**：SessionStart、UserPromptSubmit、PreToolUse、PostToolUse、
   Stop、SessionEnd 六个生命周期事件都会触发记录。
2. **压缩成观察（observation）**：原始会话记录太长，先由模型压缩成简短的
   观察，只留“做了什么 / 决定了什么”。
3. **存两份索引**：SQLite（FTS5 全文检索）+ Chroma（向量语义检索），既能
   按词找，也能按意思找。
4. **分层取回**：先给索引层（ID、标题、类型、时间），再按需看时间线，
   最后只取真正需要的完整观察，省 token。

## 隐私：不想被记的东西

把内容包在 `<private>` 标签里，claude-mem 会跳过存储。密钥、个人笔记、
不希望进记忆库的内容都用这个办法。默认数据在 `~/.claude-mem/` 下。
注意两点：其一，压缩步骤若调用外部模型，仍可能传输提示词或会话上下文；
其二，当前默认安装路径偏向托管 observer，想严格本地运行要在安装时就
选好本地 provider，不要一路默认点下去。

## 适用与不适用

- 适用：一个人或小团队长期维护同一个代码库，决策和踩过的坑会越积越多。
- 不适用：把它当成“正确性保证”——记错的决策也会被重新注入，需要偶尔
  回顾清理；也不适合直接当多租户产品的记忆服务用。

## 下一步（给想深入改的人）

先在本地跑通 worker，再动观察结构或搜索工具并配测试；不要在没有跑起来
的情况下直接改 hooks / 压缩 / 检索管线。
