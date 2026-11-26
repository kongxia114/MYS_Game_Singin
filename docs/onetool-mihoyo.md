# OneTool `mihoyo` 模块观察记录

记录时间：2026-11-26。  
目的：判断旧代挂平台 OneTool 的米游社签到模块是否值得逆向，以及能否复用。

## 1. 文件位置与形态

```text
extend/mihoyo/Mihoyo.php
```

- 约 45 KB，单文件
- 文件头注释：`God - Team`，创建时间 `2022-08-30`
- 注释声明“严禁反编译、逆向等任何形式的侵权行为”
- 配套页面：`app/index/view/console/mihoyo/add.html`、`add2.html`、`info.html`

## 2. 它不是 AES / ionCube

静态检查结果：

- `eval` 出现 17 次
- `base64_decode` 出现 14 次
- 出现 `str_rot13`
- 大量二进制/非 ASCII 变量名
- 自定义 XOR 解码逻辑
- 没有 ionCube、SG11、Zend 等商业加密扩展特征

结论：这是**多阶段 PHP 代码混淆**，不是数据加密。真正的内容在运行时通过 `eval` 逐层还原。

## 3. 它的工作模式是“账号密码登录”

`add.html` / `add2.html` 的表单字段是：

```text
login-username   账号
login-password   密码
```

说明它保存的是米游社账号密码，由服务端模拟登录换取 Cookie/Token，再执行签到。  
这也是它需要加密模块的原因：登录流程比单纯签到更敏感。

风险：

- 需要把米游社密码交给第三方平台
- 平台已经停止维护
- 账号密码一旦泄露，影响范围远大于 Cookie

**不建议用真实账号运行这套旧代码。**

## 4. 为什么不需要解密它

米游社签到协议本身已经明确，见 [signin-protocol.md](./signin-protocol.md)：

```text
GET  /binding/api/getUserGameRolesByCookie
GET  /event/luna/info
POST /event/luna/sign
```

OneTool 的模块本质上只是：

```text
账号池 + 定时任务 + 登录换 Cookie + 调上面三个接口 + 结果通知
```

加密层保护的是“账号密码登录”和平台自身逻辑，不是签到算法。

## 5. 如果仍然要研究

按成本从低到高：

1. **抓包**（推荐）  
   让旧平台在隔离环境运行，用 Fiddler / mitmproxy 记录请求，直接看接口、参数和返回，不需要解密代码。

2. **动态调试**  
   PHP + Xdebug 在 `eval` 处断点，逐层取出运行时字符串。工作量大，且会遇到完整性校验。

3. **静态解码**  
   先处理 `base64_decode` / `str_rot13`，再还原自定义 XOR。适合单层混淆，多阶段混淆会很耗时。

注意：

- 该文件明确禁止逆向，且属于旧平台的“开心版”源码；本文只做协议层面的观察记录，不包含其代码，也不提供破解/授权绕过方法。
- 任何动态分析都应在隔离虚拟机、无真实账号的环境里进行。

## 6. 推荐替代方案

如果要恢复“多账号定时签到”，不需要复活 OneTool 的加密模块：

- 用本项目 `npm run login` 抓完整 Cookie（不保存密码）
- 用 `main.js` / GitHub Actions 定时执行
- 需要多平台编排时，把签到协议封装成独立任务即可

这样既避免密码托管，也不依赖已经停止维护的加密代码。
