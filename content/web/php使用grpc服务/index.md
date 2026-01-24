---
# general config
title: php使用grpc服务(从0开始，并且包含mapfield文件解释)
subtitle: grpc是通过定义服务端和客户端的代码来实现的通信的。但是要实现通信，还是要将其方法包装为一个http请求，除非你把grpc的服务端代码放在本地的端口上。
slug: grpc-php
tags: 
  - gRPC
  - php
  - grpc-php
  - grpc-python
  - grpc-go
description: grpc是Google 开发的高性能、开源的远程过程调用（RPC）框架，基于 HTTP/2 协议进行通信，使用 Protocol Buffers（protobuf）作为接口定义语言，可以看为一种协议。grpc可以用于各种不同服务间的通信，屏蔽底层细节(如编程语言，操作系统等)
date: 2024-03-24
---
grpc是面对微服务框架而风生水起的，上次我用python编写了一个图神经网络处理的微服务，使用grpc放在我的服务器本地端口上。

现在我希望我的一个php项目也可以调用该服务，现在来试一试吧~

<!--more-->

## 流程

- php的服务器安装protoc
- php的服务器安装grpc
- ~~编写服务端代码~~
- 编写客户端代码

由于服务端(python)的代码已经编写或者说已经部署，就不做叙述了。

安装代码请根据自己的php版本和grpc版本酌情自定义。本人使用的php8.0，grpc1.62.0，protobuf4.62.0

## 安装protoc解释器

和windows开发一样，使用grpc服务均需要使用protoc解释器，去官方github下的release下载linux版本：
[protoc-26.0-linux-x86_64.zip](https://github.com/protocolbuffers/protobuf/releases/download/v26.0/protoc-26.0-linux-x86_64.zip)

解压到 `/usr/bin/`目录下

> 如果你使用宝塔，你也可以直接先在本地电脑上解压，把解压后的bin文件夹里的protoc文件上传到 `/usr/bin/`中就好了。

终端中输入 `protoc`,有返回即为成功

## 安装grpc

无论你服务器是否安装pecl，可以直接通过http请求安装php相关包，但是必须安装了php(废话)

```bash
# 下载解压 grpc
cd ~
wget http://pecl.php.net/get/grpc-1.62.0.tgz
tar xvf grpc-1.62.0.tgz
cd grpc-1.62.0

# 生成配置并编译安装(编译安装时间很长，我大概安装了一小时左右)
# 注意选择你自己的路径以及php版本，我是80
/www/server/php/80/bin/phpize
./configure --with-php-config=/www/server/php/80/bin/php-config
make && make install
```

之后要配置php的拓展
**注意，这是必要的，compose安装的grpc依赖，底层还是调用的这个grpc拓展！**

```bash
# 配置PHP扩展
cd grpc-1.62.0
echo "extension = grpc.so" >> /www/server/php/80/etc/php.ini
cd protobuf-4.62.0  # 如果没有路径请仿照grpc安装的方式手动安装安装一下，我个人觉得可能并不需要
echo "extension = protobuf.so" >> /www/server/php/80/etc/php.ini
```

最后重启一下php和nginx服务就大功告成了

## 编译protoc文件

具体的protoc文件的定义详细见我[之前的博客](http://viogami.me/index.php/archives/158/)

需要安装protoc和grpc_php_plugin

使用如下代码生成：

```bash
protoc --php_out ./ you-file.proto  #需要安装protoc解释器，生成protoc的php定义文件在当前目录(./)
protoc --grpc_out ./ you-file.proto #需要grpc_php_plugin插件安装，生成grpc文件在当前目录

```

第一行生成你的proto数据定义文件，我生成了 `GCNResult.php`,`Node.php`,`Edge.php`,`GraphData.php`,
同时还会生成一个GPBMetaData文件夹。
第二行生成php的grpc文件：`GCNServiceClient.php`

注意，如果你没有生成grpc文件的插件(安装grpc出现问题)，可以直接下载该插件
然后通过如下代码生成 `xxxClient.php`文件

```bash
protoc --grpc_out ./ --plugin=protoc-gen-grpc=/your-path-to-plugin/grpc_php_plugin you-filename.proto

:: 自用
:: protoc --grpc_out ./ --plugin=protoc-gen-grpc=/opt/share/grpc_php_plugin gcn.proto
```

## 编写php请求的代码(客户端代码)

### 编写文件前置注意事项

- **注意：如果你使用宝塔，需要把php设置里的禁用函数 `putenv`和 `proc_open`给删除，不然composer安装无法进行。**
- 需要编写composer.json文件，因为使用了 `dirname(__FILE__).'/vendor/autoload.php'`该自动导入功能。json文件内容示例：

```json
{
    "require": {
    "grpc/grpc": "*",
    "google/protobuf": "*"
    },
    "autoload": {
        "psr-4": {
          "GPBMetadata\\": "protoc/GPBMetadata/",
          "protoc\\": "protoc/"
        }
      }
}
```

编写后在服务器该文件目录下启动终端输入 `composer install`即可，会生成vendor文件夹

现在我将编写一个最简单的php文件来调用这个服务。

```php
<?php
require dirname(__FILE__).'/vendor/autoload.php'; // 引入 gRPC PHP 扩展的自动加载文件
require 'protoc/GraphData.php'; // 引入包含 protoc文件夹下的grpc生成文件
require 'protoc/Node.php'; 
require 'protoc/Edge.php'; 
require 'protoc/GCNResult.php'; 
require 'protoc/GCNServiceClient.php';

// 进行grpc请求，获取gcn处理后的数据，返回json字符串
function GCN_request()
{
    $client = new GCNServiceClient('localhost:9999', [
        'credentials' => \Grpc\ChannelCredentials::createInsecure(),
    ]);
  
    // 创建一个实例的图数据
    $G_example = new GraphData();
    $G_example->setNodes([
        (new Node())->setId("node1")->setFeatures([0.1, 0.2, 0.3]),
        (new Node())->setId("node2")->setFeatures([0.4, 0.5, 0.6]),
    ]);
    $G_example->setEdges([
        (new Edge())->setSourceId("node1")->setTargetId("node2"),
    ]);
  
    // 发送请求并接收响应
    list($response, $status) = $client->ProcessGraph($G_example)->wait();
    if ($status->code !== Grpc\STATUS_OK) {
    // gRPC 请求出错
    throw new Exception('Error calling grpc server -> ProcessGraph: ' . $status->details);
    exit(1);
    }
    // 因为我的返回结果是个map数据类型，php中没有该类型，需要做一个遍历取值，如果是string类型可以直接取。
    $NodeScores = [];
    foreach ($response->getNodeScores() as $key => $value) {
        $NodeScores[$key] = $value;
    }
    return json_encode($NodeScores);
}
```

该函数返回一个json数据，想要修改可以使用 `json_decode()` , 至此，大功告成！

## 备注

因为我的protoc文件的返回结果定义为一个map，这是go才有的数据结构，php没有
使用 `var_dump($response->getNodeScores())`可以看到这是谷歌grpc核心中定义的一个mapfield object，可以直接像go的map一样，用 `$response->getNodeScores()['xxx']`获取值。

虽然官网显示有ToString()的方法：[https://cloud.google.com/dotnet/docs/reference/Google.Protobuf/latest/Google.Protobuf.Collections.MapField-2#Google_Protobuf_Collections_MapField_2_ToString](https://cloud.google.com/dotnet/docs/reference/Google.Protobuf/latest/Google.Protobuf.Collections.MapField-2#Google_Protobuf_Collections_MapField_2_ToString)
但是在这里无法使用，所以我只能通过遍历的方式获取所有值存到一个数组里，毕竟map结构本身就不支持一次获取全部，还是要遍历。

