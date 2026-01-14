### 配置
webhook回调Request Body设置：
```json
{
    "msg_type": "post",
    "content": {
        "post": {
            "zh_cn": {
                "title": "xanny server IP changed",
                "content": [
                    [
                        {
                            "tag": "text",
                            "text": "IPv6地址：#{ipv6Addr}"
                        }
                    ],
                    [
                        {
                            "tag": "text",
                            "text": "域名更新结果：#{ipv6Result}"
                        }
                    ]
                ]
            }
        }
    }
}
```

### 参考资料
- [ddns-go-resource-code](https://github.com/jeessy2/ddns-go)