1. 从浏览器到浏览器内核

    1. 查看缓存

1. HTTP请求的发送

    1. DNS查询

        1. 浏览器查询是否有缓存的DNS
        1. 系统查询是否有缓存的DNS
        1. 检查hosts中是否有指定的映射关系
        1. 向根服务器发送解析DNS请求
            > DNS是一个逐步缩小范围的查找过程, fex.baidu.com为列
              1. 根服务器查找到负责.com的服务器, 将结果返回客户端, 客户端继续查询
              1. 负责.com的服务器找到负责baidu.com的服务器, 客户端继续查询
              1. 负责baidu.com的服务器找到fex.baidu.com的ip地址, 将结果返回客户端
    1. 通过Socket发送数据
        > Socket Api 包括TCP协议和UDP协议, HTTP通常选择TCP协议
        > HTTP是纯文本格式
        1. IP协议
        1. TCP协议
        1. HTTP

