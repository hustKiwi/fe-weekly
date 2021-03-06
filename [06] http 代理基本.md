# http 代理基本

上次说代理太抽象，这次说些具体的。

## 动机

开发中时常会需要利用代理来解决一些临时需求，尤其是搭建封闭开发环境比较耗时时。

提到开发专用代理工具，首先想到的就是 Fiddler，常见用法：

* 将远端文件和本地文件进行替换调试
* 制造测试用假数据。
* 性能测试，作为指定请求的入口点，比起用 chrome 的 network 之类的调试工具，统计的数据更可靠。
  能处理更复杂的 http 统计。比如你只想统计本站的资源，排除所有 CDN 资源。

最近我还看到了 腾讯 AlloyTeam 做的一个开源工具，Github 上 500+ 星：[livepool][0]。
这个工具是基于 Node 的，相较于 Fiddler 跨平台性更好。如果你没时间折腾高级代理，只是想转发下文件之类的，这个工具是个不错的选择。

## http 的特性

了解 http 协议是了解如何代理 http 通信的基础。http 比起用直接用 TCP socket 通信要方便很多，其中重要的一点就是 http 只有一次应答就结束，而 socket 没有限制应答次数，这让 socket 通信需要处理各种中途异常和状态管理。http 作为上层，不关心 socket 内部细节。

这里只是想强调 http 一次应答的特性。它的代理也具备这个特性。

此外 http 对一次应答的描述信息异常丰富，而且描述可以扩展，且描述都是纯文本的，通俗易懂。

## 头解析是 http 代理的关键

直接举个通俗的例子。比如你设置了系统全局的 http 请求都代理到 `127.0.0.1:8013`，代理服务 P，正在监听这个地址。
现在 Chrome 访问 `http://www.baidu.com`，Chrome 默认会使用系统设置的代理，系统会建立一个 socket 到 P，将类似如下的 http 请求头部 write 到 socket 链接里：

```
GET / HTTP/1.1
Host: www.baidu.com
Connection: keep-alive
Accept-Encoding: gzip,deflate,sdch
User-Agent: Chrome
```

P 一定需要做的事情就是先解析这个头信息，而其中对转发最关键的就是 `Host` 的值，P 发现 `Host` 现在为 `www.baidu.com`，于是建立跟这个地址的 http 通信，将 Chrome 想发送的信息继续发给这个地址，最后将取回的数据全部吐给 Chrome，这就算完成了一次简单代理了。

从上面我们知道，虽然 http 没规定一定要有 `Host` ，但如果一个请求里没 `Host`信息，且首行又不是绝对 url 地址（比如 `GET http://baidu.com/ HTTP/1.1` 里用的是绝对地址），那么代理将不知道该往哪儿转发，所以使用时要多加注意这个点。这个也和 Virtual Host 的概念相关联。

## 一个代理的实例

为了说明 http 代理是多么的简单。这里用 Node 写个例子，库用到了 [nobone][1]。

```javascript
var nb = require('nobone')();

// 只代理全部 http GET 请求
nb.service.get('*', function (req, res) {
    // 这里 url 事实上是 Node 通过 Host 等信息拼接成的。
    nb.kit.request(req.url)
    .done(function (body) {
        // 将返回的信息吐给客户端
        res.send(body);
    });
});

nb.service.listen(8013);
```

如果你 Chrome 里安装了 [SwitchySharp][2]，将全部请求代理到 8013 端口，就能工作了。这是个极其简陋的代理，所有 http `POST` 请求都会返回给 Chrome 404，所有 `GET` 请求转发时都被去除了 header 信息。

## 5 行代码实现 Fiddler 文件替换功能

```javascript
var nb = require('nobone')();

nb.service.get('st/a.js', function (req, res) {
    res.sendfile('assets/b.js');
});

nb.service.listen(8013);
```

比如在 [SwitchySharp][2] 里设置 `http://baidu.com/st/a.js` 代理到 8013。

![switchysharp][switchysharp]

之后你的本地文件 `assets/b.js` 就会替换掉远端的 `http://baidu.com/st/a.js` 文件了。

## 在页面内注入脚本

```javascript
var nobone = require('nobone');
var nb = nobone({
    service: {},
    proxy: {}
});

nb.service.use(function (req, res) {
    if (req.url == 'http://baidu.com/') {
        nb.kit.request({
            url: req.url,
            headers: req.headers
        }).done(function (body) {
            // 往页面内注入一些 js helpers，比如自动页面重载库。
            // 比如还可以在某些网页注入 jquery，lodash 之类的工具，辅助调试。
            res.send(body + nobone.client());
        });
    } else {
        // 其他请求按照原样转发。
        nb.proxy.url(req, res);
    }
});

nb.service.listen(8013);
```

## 使用 PAC 来提升 Mobile 端的代理效率

无脑的将全部流量转发到某个代理是很浪费代理带宽的，实际工作时我们往往只想处理某些特定请求而已。

PC 端代理可以用 [SwitchySharp][2] 之类的工具来控制转发，只让匹配模式的 url 流量发往 代理服务，其他请求不走代理直连。但 Mobile 端通常没有这么好用的工具，iOS 甚至没有权限达到这种目的。这种情况下，使用 PAC 自动代理会是不错的选择。关于 PAC 的详细介绍请移步 [Wiki][3]。

简单来讲 PAC 是一个 js 文件，如果设置了系统全局 PAC，那么在发送 http 请求前，这个 js 内的 FindProxyForURL 函数会被执行一次，通过这个函数的返回结果来确定如何代理。上一个例子的 PAC 版本，如下：

```javascript
var nobone = require('nobone');
var nb = nobone({
    service: {},
    proxy: {}
});

// PAC 文件地址为 http://127.0.0.1:8013/pac
nb.service.get('/pac', proxy.pac(function () {
    if (match('http://www.baidu.com/'))
        // 使用当前服务代理，即 127.0.0.1:8013
        return curr_host;
    else
        // 不适用代理服务，直接访问
        return direct;
}));

nb.service.use(function (req, res) {
    nb.kit.request({
        url: req.url,
        headers: req.headers
    }).done(function (body) {
        res.send(body + nobone.client());
    });
});

nb.service.listen(8013);
```

如果你不知道如何设置 PAC，这里有两个例子:

* SwitchySharp: 情景模式 --> 新建情景模式 --> 如下图设置：

  ![switchysharp-pac][switchysharp-pac]

* iOS: 设置 --> Wi-Fi --> 选择连接中的 Wi-Fi 右边的 ⓘ，进入详情页 --> 如下图设置：

  ![ios-pac][ios-pac]

  这里的地址 `i.ysmood.org` 相当于你的代理服务相对于手机的地址。


## 关于 https 代理

处理 https 请求的内容通常需要配置证书等安全操作。用法和一般的中间人攻击类似，但由于有远端3方的证书验证，类似 Chrome 的安全浏览器很容易判断受到了中间人攻击，这时候需要用户主动信任中间人的证书（安装中间人提供的证书）。

鉴于这是个麻烦的过程，http v1.1 提供了 `CONNECT` 方法，有别于 `GET`，`POST` 方法，这个方法会利用 http 建立一个 socket 隧道进行 https 通讯。 这样就可以方便进行 https 代理了，不足在于，你除了转发的目标地址，其他 http 都信息都无法获取到，换句话也就无法做任何通信篡改。当然这不会是大问题，开发时很少会碰到需要篡改 https 的情况，大部分时候只需要转发下就可以。

下面是一个 https 转发示例：

 ```javascript
var nobone = require('nobone');
var nb = nobone({
    service: {},
    proxy: {}
});

nb.service.server.on('connect', function (req, sock, head) {
    // 直接转发全部 https 请求
    nb.proxy.connect(req, sock, head);
})
 ```

## 总结

在恰当的时候使用代理会很大层度上改善开发流程，简化调试复杂度。前提是你知道你在干什么，这就要求我们把 http 协议的常见用法了解清楚。再者作为互联网开发者，了解 http 协议算是基本中的基本。


[0]: https://github.com/rehorn/livepool
[1]: https://github.com/ysmood/nobone
[2]: https://chrome.google.com/webstore/detail/proxy-switchysharp/dpplabbmogkhghncfbfdeeokoefdjegm?hl=en
[3]: http://en.wikipedia.org/wiki/Proxy_auto-config
[switchysharp]: img/[2014.07.28]/switchysharp.jpg
[switchysharp-pac]: img/[2014.07.28]/switchysharp-pac.jpg
[ios-pac]: img/[2014.07.28]/ios-pac.jpg

<style type="text/css">
    img {
        max-width: 800px;
        max-height: 600px;
    }
</style>



