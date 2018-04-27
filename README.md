## 树莓派3实现天工智能边缘计算（iotedge）

​	最近在测试百度天工物管理的功能，注意到百度天工推出新服务“智能边缘计算”（iotedge），于是带着两个好奇心申请了试用，1、边缘计算有些什么功能？2、端云一体是怎么实现的？

​	申请到智能边缘计算iotedge测试包后从readme文档获悉，iotedge有三项功能：MQTT，规则引擎，函数计算（这些功能与云端功能相似）。



​	解压缩后得到四个文件夹（bin、conf、example、liberec）。从文件夹命名上可以判断，iotedge的应用程序放在bin文件夹中，liberec存放了库文件，示范例子在example中，配置文件夹在conf中。所有功能都需要在conf/iotedge.yml中进行配置，如下：

```yaml
listen:
  - tcp://:1883
  - ssl://:1884
  - ws://:8080/mqtt
  - wss://:8884/mqtt
certificate:
  cert: 'conf/server.pem'
  key: 'conf/server.key'
principals:
  - username: 'test'
    password: 'be178c0543eb17f5f3043021c9e5fcf30285e557a4fc309cce97ff9ca6182912'
    permissions:
      - action: 'pub'
        permit: ['+', '#', 'test/+', 'test/#', 'test', 'benchmark', 'test/中文']
      - action: 'sub'
        permit: ['+', '#', 'test/+', 'test/#', 'test', 'benchmark', 'test/中文']
      - action: 'shadow'
        permit: ['edge_test_device_1']
```

yaml是专门用于配置文件的语言，非常简洁强大。我个人认为虽然强大，但是对于非IT行业出身的工控人来说还是不易上手，毕竟从事工业物联网的很多是来自工控界的，所以配置方法最好能借助比较友好的UI界面+下载的方式。

​	开始着手开始测试，首先开始试三大功能中第一个主要的功能MQTT，这是数据来源的根本，少了它后面的边缘计算则英雄无用武之地。

### 一、MQTT

​	MQTT一般有消息订阅（sub）和发布（pub）功能，从readme中了解到principals.permissions.action的属性除了订阅和发布功能，还有云端物管理通信（shadow）功能。这里需要着重提一下shadow属性。

​	iotedge可以在端侧局域网内独立运行，也可以端云一体运行。在局域网内运行时，MQTT的设置只需在principals.permissions.action属性种设置pub和sub的主题，遵循MQTT协议规则的主题订阅消息就可以了。

​	而端云一体运行时，需要把principals.permissions.action属性设置成shadow。设置为shadow属性，主题名称则切换成为“$baidu/iot/shadow/{devicename}/xxx”的类型，shadow内的列表名则为设备名(devicename）。这点在readme种没有很清晰的说明。

​	要实现端云一体功能还需要增加“cloud”项，如下：

```yaml
cloud:
  device:
    name: 'your_core_device_name'
    ca: '/path/to/ca.pem'
    address: 'mqtt.iot.gz.baidubce.com:1884'
    username: '0be3daac18054b78b0dc9e267f11111/your_core_device_name'
    password: 'RiR7YP9pF6n3Lr0iUAQqeug7YWMY/pGyNDNde111111='
```

具体的配置可以参考readme。不过需要注意三点：

1、在使用端云一体功能之前需要先熟悉百度天工物管理的功能。

- 要弄清楚物管理中的物模型和物影子的关系
- 物影子建立后的地址、用户名、密码需要保存

2、理解核心设备和关联设备的关系。




- 做为核心设备的物影子的地址、用户名、密码将要在配置文件的cloud项目中使用

3、物影子建立时生成的地址如下：

​	tcp://xxxxxxxxxxxxxx.mqtt.iot.gz.baidubce.com:1883

​	ssl://xxxxxxxxxxxxxx.mqtt.iot.gz.baidubce.com:1884

​	wss://xxxxxxxxxxxxxx.mqtt.iot.gz.baidubce.com: 8884

​	地址后面的1883、1884、8884是端口号。而，在使用MQTT.fx工具或写程序连接物影子时，地址的引用是不需要带端口号的。

​	而，iotedge的中的cloud配置项是需要把端口带上，如：“ address: 'mqtt.iot.gz.baidubce.com:1884' ”。

​	这个地方容易混淆，建议最好能统一。即，cloud配置项再增加一个port对象用于端口号设置。

​	端和云的配置设定好后点击“生成版本”。版本生成后就可以下发版本了，点击“下发版本”后云端的配置将会下载到端设备上。端设备收到新配置后，原来的配置等待正在处理的消息完成后退出，新的配置文件生效，原来的配置文件则无效。

### 二、规则引擎

规则引擎简单易用容易理解。通俗的讲，规则引擎实际上是指挥消息从里来到哪里去。做到灵活地转发和处理设备消息。

- 属性subscriptions.source （从哪里来）

- 属性subscriptions.target  （到哪里去）

  source配置规则依据MQTT协议订阅主题规则，target规则依据MQTT协议发布主题规则。

规则引擎有三种工作方式：

1、消息的简单转发

2、消息的多级转发

3、消息经过函数计算

1和2的工作方式都是在订阅和发布主题之间的消息路由传递

3是将消息转到去做函数计算。目前支持python2.7和sql函数。source和target的配置中只需加上“$function”前缀，后面是“/”+函数名。

### 三、函数计算

可以说MQTT和规则引擎是为函数计算的做准备。函数计算目前支持sql和python2.7，据了解后续会陆续支持python3.0、node.js等语言。

函数相关的配置在function配置项中进行。有四要素：

1、设置函数名。在function.name属性中设置

2、选择执行语言。在function.runtime属性有’sql‘和’python2.7‘

3、运行函数。在function.handler属性中设置函数名称。sql则特指select语句。

4、指定函数python程序的路径。在function.codedir中设定

这是初步测试使用百度天工智能边缘计算的情况。从功能上评价，总体还是很不错的。MQTT实现了对数据的输入输出接口，规制引擎清晰明了的控制着数据流向，函数计算实现智能化计算。不足之处是，友好度欠缺些，工业控制出身的人有点难理解。
