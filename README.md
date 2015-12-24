[![Build Status](https://travis-ci.org/funny/gateway.svg?branch=master)](https://travis-ci.org/funny/gateway)
[![Coverage Status](https://coveralls.io/repos/funny/gateway/badge.svg?branch=master&service=github)](https://coveralls.io/github/funny/gateway?branch=master)

介绍
====

本项目是一个基于[`xindong/frontd`](https://github.com/xindong/frontd)重制的通用网关程序。

本网关只有TCP或HTTP流量转发功能，负责为每个客户端连接建立一个后端连接进行流量转发。

本网关有以下使用价值：

1. 避免应用服务器直接暴露到公网
2. 提高故障转移的效率

本网关有以下特性：

1. 易接入，只需要对客户端做小量修改即可接入，不需要修改已又通讯协议
2. 可扩展，可以任意多开水平扩展以实现负载均衡和高可
3. 零配置，运维人员无需手工进行后端服务器列表配置
4. 高性能，网关内部初始化好连接后，使用[`sendfile`](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy2/)进行零拷贝的数据传输
5. 端口重用，利用高版本Linux内核的[`reuseport`](http://www.blogjava.net/yongboy/archive/2015/02/12/422893.html)机制，可以开多个网关进程守候同一个端口，以提高多核利用率

协议
====

基本通信流程：

1. 客户端连接网关
2. 客户端发送目标服务器地址密文
    * 如果读取失败，网关回发`400`状态码给客户端
3. 网关解密目标服务器地址
    * 如果解密失败，网关回发`401`状态码给客户端
4. 网关连接目标服务器
    * 如果发生超时，网关回发`503`状态码给客户端
    * 如果发生IO错误，网关回发`502`状态码给客户端
6. 网关发送缓存中残余数据
    * 发送残余数据时如果出错，网关回发`502`状态码给客户端
7. 网关回发成功状态码`200`给客户端
8. 客户端连接和目标服务器连接开始对传数据

状态码说明：

| 状态码 | 状态说明 |
|-----|---------|
| 200 | 握手完成，可以开始传输数据 |
| 400 | 请求数据读取过程中发生错误 |
| 401 | 网关解密地址信息失败 |
| 502 | 网关无法连接后端服务器 |
| 504 | 网关连接后端服务器超时 |

客户端发送到网关的目标服务器地址是经过`AES256-CBC`加密的，所以外网攻击者无法对网关后的内往服务器进行猜测和任意连接。

目标服务器地址同时支持二进制和文本两种格式。使用文本格式时，客户端发送一行`base64`编码过的服务器地址密文到网关。

例如：

```
U2FsdGVkX19KIJ9OQJKT/yHGMrS+5SsBAAjetomptQ0=\n
```

使用二进制格式时，首字节必须是`0x0`，第二个字节为密文长度，后续为密文内容，密文内容不需要做`base64`编码。

对应上个例子的二进制地址格式为：

```
0x00 0x20 
0x53 0x61 0x6c 0x74 0x65 0x64 0x5f 0x5f 
0x4a 0x20 0x9f 0x4e 0x40 0x92 0x93 0xff 
0x21 0xc6 0x32 0xb4 0xbe 0xe5 0x2b 0x01 
0x00 0x08 0xde 0xb6 0x89 0xa9 0xb5 0x0d
```

接入
====

1. 生成`Secret`，并保存在安全的文档中
	 * 可以在线生成：https://lastpass.com/generatepassword.php
2. 使用上述`Secret`部署网关
3. 使用`AES`算法加密文本格式的后端地址，生成`base64`编码的密文
    * 可以在线生成：http://tool.oschina.net/encrypt
    * 也可以使用`openssl`命令生成，如：
    ```
    echo -n "127.0.0.1:62863" | openssl enc -e -aes-256-cbc -a -salt -k "p0S8rX680*48"
    ```
    * 举例，当后端地址为`127.0.0.1:62863`并且`Secret`为`p0S8rX680*48`时，密文结果应类似：
    ```
    U2FsdGVkX19KIJ9OQJKT/yHGMrS+5SsBAAjetomptQ0=
    ```
    _注：上述方式都会使用随机Salt，这也是建议的方式。其结果是每次加密得出的密文结果并不一样，但并不会影响解密_

加密后的服务器地址通常应该在拉取服务器列表之类的场景中发送给客户端。

客户端只有加密后的地址，不应该有`Secret`，重要的事情说三遍：

* 切勿将`Secret`写入客户端代码！
* 切勿将`Secret`写入客户端代码！
* 切勿将`Secret`写入客户端代码！

设置
====

网关可以通过以下环境变量进行设置：

| 变量 | 用途 |
|-----|----|
| GW_SECRET | 解密地址用的秘钥，必须设置 |
| GW_ADDR | 网关服务器地址，默认为0.0.0.0:0 |
| GW_REUSE_PORT | 是否启用端口重用特性，值为1时表示启用，默认为0 |
| GW_PPROF_ADDR | [`net/http/pprof`](https://golang.org/pkg/net/http/pprof/)所使用的地址，建议是内网地址，无值的时候不开启，默认无值 |
| GW_DIAL_RETRY | 网关连接目标服务器的重试次数，默认为1 |
| GW_DIAL_TIMEOUT | 网关每次连接目标服务器的超时时间，单位是秒，默认为3 |

网关启动后，会在工作目录下生成一个`gateway.pid`文件记录进程id，可以用以下命令安全退出网关：

```
kill `cat gateway.pid`
```

TODO
====

* 后端服务器主动连接网关？
* 提供客户端接入代码
* 提供后端接入代码
* 提升单元测试覆盖率