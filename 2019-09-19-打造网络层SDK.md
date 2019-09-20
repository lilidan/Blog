# 打造网络层SDK

网络层的开发并没有多复杂，并不是技术上有很大的难度，而难点多半是基于业务的封装。并不是真的需要我们利用BSD socket去实现一个http协议的请求。
iOS的http请求，我们就基于AFNetworking,因为AFNetworking已经帮我们做一些事了。


### AFNetworking

- 2.0版本封装成NSOperation的形式，提供了resume/cancel等等处理。但是3.0的不再需要，因为NSURLSessionTask自带。
- 不同于2.0的利用NSThread。3.0的直接把operationQueue传给NSURLSession。queue并发数为1,是串行队列。
- 增加了NSData的文件处理，上传/下载。3.0的NSURLSession自带data处理。
- 方便处理JSON/XML数据。
- 方便处理HTTPS(SSLPinning):从bundle中读取证书,检查服务器证书的有效性,跟服务器证书(或者公钥)进行比对。
- 有Reachablity的API:基于SCNetworkReachability，只能区分wifi和蜂窝。如果是蜂窝网络其实可以根据CTTelephonyNetworkInfo来区分2g/3g/4g


## 当前需求和问题

- model对象请求，model对象返回, 框架处理序列化问题。
- 缓存
- IP直连，HTTPDNS
- 刷新类请求，资源类请求。后台类请求。
- 请求失败，也可能是认证失败
- 服务器错误，重发机制
- 是否显示loading，toast
- 实现后门控制，debug和抓包记录
- 网络监控和统计

#### 封装

- delegate还是block？block打不到断点。delegate又多个请求很烦,位置很烦。最好是两者都支持，优先支持block
- 直接在业务逻辑里做序列化是不合适的，传入URL也是不合适的，从解偶复用的角度考虑。每一类请求都应该封装成类，发送单一请求时实例化对象。某类请求可以互相继承。请求的参数作为类的属性，发送请求时利用JSONModel序列化，而非参数的作为method来提供给子类覆写,类似YTKNetwork。
- 不同于YTKNetwork,利用JSONModel，一般情况下返回的也是对象。
- 用户只需要关心request的参数，response的对象即可。不需要关心client，不需要关系序列化。
- 封装全局配置，同时针对每一个请求或者每一类请求，提供配置baseurl,path,header,params,mothod,serializerType

#### 缓存

利用任何缓存框架即可，比如TMCache，YTKNetwork直接使用NSKeyedUnarchiver持久化，性能不好。

#### IP直连
ip直连主要有三个方面，一个是IP测速，一个是SSL校验，一个是Cookie

###### 测速

IP 测速先是针对于每个域名，都有一个ip列表。第一个连接利用运营商来连接。后续的连接根据rtt时间(0.6)和连接成功率(0.4)进行评分。平均延迟超过200ms 切换ip。当一个ip请求个数达到15时，计算分数。当分数差距达到某个阈值时，切换ip。当请求成功率小于一个阈值时也切换iP。

###### SSL 校验
发送请求时把域名替换成ip，并且保存在字典里。
AFSecurityPolicy是在URLSession:didReceiveChallenge:的时候进行证书处理的(NSURLAuthenticationMethodServerTrust而非clientCertificant)。扫描bundle的证书。如果SSLPinning，就在服务端下发的证书链中找是否已经存在本地的证书。
如果IP直连，需要Hook掉evaluateServerTrust:forDomain:的方法。然后把domain参数中的ip替换成域名。

###### Cookie
收到请求的时候把cookie根据url存下来，发送请求的时候再根据url带上

#### HTTPDNS

DNS解析0延迟的主要思路包括：
- 构建客户端DNS缓存
- 热点域名预解析(比如主域名)
- 懒更新策略:利用DNS懒更新策略来实现TTL过期后的DNS快速解析。所谓DNS懒更新策略即客户端不主动探测域名对应IP的TTL时间，当业务请求需要访问某个业务域名时，查询内存缓存并返回该业务域名对应的IP解析结果。如果IP解析结果的TTL已过期，则在后台进行异步DNS网络解析与缓存结果更新

SNI:一个服务器使用多个域名和证书的时候，请求的是ip地址，所以服务器不知道返回哪个证书。通过libcurl解决。

#### 请求子类

有一些请求是常用的，但是又具备一定特性,这种情况下应该使用新的NSURLSession。Apple建议一个App中在10个以内。
- 图片类请求:一般用SDWebImage,主要涉及缓存、重试、渐进式渲染、图片解码等问题
- 大文件类请求:主要涉及断点续传,只在GET,传入断点续传的目录即可

#### 请求逻辑

- 链式请求/批量请求:用PromiseKit或者RAC封装即可

#### 失败处理

网络请求失败的话，暂存起来，一旦网络恢复，继续重发。可配置是否重发。

错误分为三种:
- 标准HTTP请求失败:请求代码>400的,比如401认证失败需要跳转统一登陆逻辑。
- iOS网络层返回的失败，包括了第一种。主要包括网络断开,请求超时,DNS失败,TCP失败,SSL失败等等。
- 业务逻辑失败,交给通用业务层处理。
