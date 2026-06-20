# Shadowrocket 解决 Claude App 登录异常

## 问题现象

iPhone 上的 Claude App 卡在 `Something went wrong`，或登录态错乱、进不去。

**根本原因**：旧账号在请求里残留了无效的登录 cookie（主要是 `sessionKey`、`routingHint`）。服务端仍按旧登录态处理，导致 App 卡死。

**解决思路**：在请求发出前，删掉 Cookie 里的 `sessionKey`、`routingHint`（或直接删整个 Cookie），让服务端判定登录态失效，App 自动跳回登录页，重新登录即可。

> ⚠️ 安全提醒：HTTPS 解密会接触到登录 cookie，别把抓包内容发给任何人，也别用不可信的代理工具。

---

## 完整操作流程（iPhone Shadowrocket）

### 第 1 步：开启 HTTPS 解密并信任证书

1. 打开 Shadowrocket → 底部 `配置`。
2. 找到当前正在使用的配置文件，进入编辑。
3. 找到 `HTTPS 解密`（有的版本叫 `MITM`），打开开关。
4. 如提示安装证书，按提示安装。
5. 安装后去系统里信任证书：
   ```text
   iPhone 设置 → 通用 → 关于本机 → 证书信任设置
   ```
   把 Shadowrocket 的证书打开信任。

### 第 2 步：添加 MITM 域名

在 `HTTPS 解密` 页面的「域名」栏（提示以逗号分割），填入：

```text
claude.ai,*.claude.ai,claude.com,*.claude.com,anthropic.com,*.anthropic.com
```

保存。

### 第 3 步：添加请求脚本

进入 `配置 → 脚本`，点右上角 `+` 新增：

- 类型：`http-request`（请求脚本）
- 名称：`Claude Clear Cookie`
- 匹配 URL / Pattern：
  ```text
  ^https?:\/\/(.+\.)?(claude\.ai|claude\.com|anthropic\.com)\/.*
  ```
- 脚本内容（只删 `sessionKey` 和 `routingHint`，保留其余 cookie）：

```js
let headers = $request.headers || {};
let cookie = headers["Cookie"] || headers["cookie"];

if (cookie) {
  let nextCookie = cookie
    .split(";")
    .map(x => x.trim())
    .filter(x => !/^sessionKey=/i.test(x))
    .filter(x => !/^routingHint=/i.test(x))
    .join("; ");

  if (nextCookie) {
    headers["Cookie"] = nextCookie;
    delete headers["cookie"];
  } else {
    delete headers["Cookie"];
    delete headers["cookie"];
  }
}

$done({ headers });
```

保存。回到 Shadowrocket 首页，确认总开关已打开。

> 如果嫌只删两个字段不够干脆，可以把脚本简化为直接删整个 Cookie：
> ```js
> let headers = $request.headers || {};
> delete headers["Cookie"];
> delete headers["cookie"];
> $done({ headers });
> ```

### 第 4 步：触发重新登录

1. iPhone 后台彻底划掉 Claude App。
2. 重新打开 Claude App。
3. 正常情况下，App 会发现登录态失效，跳回登录页。重新登录即可。

---

## ⚠️ 关键：收尾清理（重要，否则会 offline）

**登录问题解决后，Shadowrocket 的 HTTPS 解密 + 脚本会持续干扰 Claude 流量**，表现为：

- Claude App 提示 `offline / 你下线了 / 请检查网络`
- 用手机浏览器访问 `https://claude.ai` 官网**也打不开**（即使节点本身没问题）

这说明删 cookie 那一步**其实已经成功**（登录异常已解决），offline 只是 MITM 残留导致的副作用。**必须做清理**：

1. 在 Shadowrocket 的 `HTTPS 解密` 域名里，**删掉所有 claude / anthropic 相关域名**。
2. 把第 3 步那个脚本**先停止，再删除**。
3. **重启手机**。
4. 重启后先用 Safari 测 `https://claude.ai` 官网，能正常打开。
5. 再打开 Claude App，恢复正常。

---

## 经验总结

- 删 cookie 触发重新登录这一步，Shadowrocket 完全可行（和 Proxyman 等价）。
- **真正的坑在收尾**：MITM 域名和脚本用完一定要删干净并重启手机，否则 Claude/Anthropic 流量会被解密层卡住，导致浏览器和 App 双双打不开。
- 判断标准：如果连**浏览器都进不去官网**，那就是 MITM/脚本干扰，而非登录态问题——直接走「收尾清理」即可，不用再折腾 cookie。
- 解决后记得也关掉 iPhone Wi-Fi 里多余的代理设置（如果当时手动配过）。
