# 1 UE开启websocket客户端
1.1 加载WebSockets模块
1.2 重载相关函数

# 2 UE开启websocket服务器
2.1 加载WebSocketNetworking，开启WebSocketNetworking插件
2.2 重载相关函数
2.3 消息传输
	以cstring 传输；//未验证
	UE发消息需要FString->TCHAR->char->uint8
	接收需要 上边流程反向	
	
	## 关于TCHAR_TO_UFT8()
	若存至左值进行使用有可能会找不到内存长度导致字符串乱码，应直接使用

# 3 JS Client Accept message
3.1 base
```javascript
//send a request
var ws = new WebSocket("ws://127.0.0.1:3000");


//connection close
ws.onclose = function (evt) {
                console.log("Connection close");
			};

//connection state
ws.readyState

//connection complete
ws.onopen = function(evt) {
				console.log("Connection open");
			};

//recevie msg
ws.onmessage = function (evt) {}
```

3.2 recevie message
3.2.1 if get a Actor Blob, need analy
```javascript
if(evt.data instanceof Blob)
				{
					console.log("Blob");
					const reader = new FileReader();
					reader.readAsText(evt.data, "UTF-8");
					reader.onload = (e) =>
					{
						console.log("Rece: " + reader.result);
			};
```