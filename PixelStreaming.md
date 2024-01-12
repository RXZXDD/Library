# PixelStreaming

## http 重定向至 https
1. cirrus.js 内添加重定向的报文头 

``` js
if (config.UseHTTPS) {
	app.use(helmet());

	app.use(hsts({
		maxAge: 15552000  // 180 days in seconds
	}));

	//Setup http -> https redirect
	console.log('Redirecting http->https');
	app.use(function (req, res, next) {
		if (!req.secure) {
			if (req.get('Host')) {
				var hostAddressParts = req.get('Host').split(':');
				var hostAddress = hostAddressParts[0];
				if (httpsPort != 443) {
					hostAddress = `${hostAddress}:${httpsPort}`;
				}
				return res.redirect(['https://', hostAddress, req.originalUrl].join(''));
			} else {
				console.error(`unable to get host name from header. Requestor ${req.ip}, url path: '${req.originalUrl}', available headers ${JSON.stringify(req.headers)}`);
				return res.status(400).send('Bad Request');
			}
		}
		res.setHeader('X-Frame-Options', 'ALLOWALL')
		next();
	});
}
```

2. 修改config.json 内 UseHTTPS 项
3. 新增 ./SignallingWebServer/certificates 文件夹，并添加证书「client-cert.pem」和密钥「client-key.pem」