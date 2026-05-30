# 从 WSL 连接到 Windows Chrome 远程调试

当用户在 Windows Chrome 上打开了目标页面，Agent 在 WSL 环境运行时，需要通过 Chrome DevTools Protocol (CDP) 连接到 Windows 的 Chrome 实例进行远程操作。

---

## 前置条件

1. Windows Chrome 必须以 `--remote-debugging-port=9222` 启动
2. Windows 防火墙放行端口 9222
3. WSL2 网络和 Windows 之间做端口转发

---

## 操作步骤

### 步骤1：启动 Chrome 远程调试

**方式A：通过 WSL 启动（推荐）**
```bash
"/mnt/c/Users/<用户名>/AppData/Local/Google/Chrome/Application/chrome.exe" \
  --remote-debugging-port=9222 \
  --user-data-dir="C:\Users\<用户名>\AppData\Local\Google\Chrome\User Data"
```

**方式B：通过 PowerShell 启动**
```powershell
Start-Process -FilePath "C:\Users\<UserName>\AppData\Local\Google\Chrome\Application\chrome.exe" -ArgumentList "--remote-debugging-port=9222"
```

### 步骤2：解决 WSL 网络隔离

WSL2 和 Windows 是两台虚拟机，localhost 不互通。Windows 的 127.0.0.1:9222 默认只监听在 Windows 侧，WSL 无法直接访问。

**方案：netsh 端口转发（需要管理员权限）**
```powershell
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=19222 connectaddress=127.0.0.1 connectport=9222
netsh advfirewall firewall add rule name="Allow_CDP_9222" dir=in action=allow protocol=TCP localport=9222
```

然后在 WSL 中访问网关 IP 的 19222 端口。

### 步骤3：获取 Windows 网关 IP
```bash
ip route | grep default | awk '{print $3}'
# 输出: 172.28.16.1 (WSL2典型值)
```

### 步骤4：连接 CDP 并操作

```python
import asyncio, json, urllib.request, base64

CHROME_HOST = "172.28.16.1"  # Windows网关IP
CHROME_PORT = 19222

async def main():
    resp = urllib.request.urlopen(f'http://{CHROME_HOST}:{CHROME_PORT}/json')
    targets = json.loads(resp.read())
    
    for t in targets:
        if '目标网址' in t.get('url', ''):
            target = t; break
    
    ws_url = target['webSocketDebuggerUrl']
    ws_url = ws_url.replace('ws://localhost:', f'ws://{CHROME_HOST}:')
    ws_url = ws_url.replace('ws://127.0.0.1:', f'ws://{CHROME_HOST}:')
    
    import websockets
    async with websockets.connect(ws_url) as ws:
        msg_id = 1
        async def eval_js(js):
            nonlocal msg_id
            mid = msg_id; msg_id += 1
            await ws.send(json.dumps({'id': mid, 'method': 'Runtime.evaluate', 'params': {'expression': js, 'returnByValue': True}}))
            resp = await ws.recv()
            return json.loads(resp).get('result',{}).get('result',{}).get('value','')
        
        await ws.send(json.dumps({'id': msg_id, 'method': 'Page.captureScreenshot', 'params': {'format': 'png'}}))
        resp = await ws.recv()
        with open('/tmp/screenshot.png', 'wb') as f:
            f.write(base64.b64decode(json.loads(resp).get('result',{}).get('data','')))

asyncio.run(main())
```

---

## 二维码

| 问题 | 原因 | 解决 |
|------|------|------|
| Connection refused | 防火墙或端口转发 | 确保防火墙规则已添加 |
| 403 Method Not Allowed | CDP 端点不支持 | 用 WebSocket 而非 HTTP |
| 只能绑定 127.0.0.1 | Chrome 安全限制 | 用 netsh 端口转发 |
