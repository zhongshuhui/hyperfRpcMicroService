# 					<u>**DOCKER+HYPERF+NACOS+微服务**</u>



#### 1.安装nacos服务配置中心

```
docker run --name nacos-quick -e MODE=standalone -p 8848:8848 -d nacos/nacos-server:2.0.2
```



#### 2.服务端：

```
#搭建hyperf运行环境
docker run --name hyperf -v /workspace/skeleton:/data/project -p 9501:9501 -p 9502:9502 -it --privileged -u root --entrypoint /bin/sh hyperf/hyperf:8.0-alpine-v3.15-swoole
cd /data/project
#创建服务端项目
composer create-project hyperf/hyperf-skeleton server_rpc
#引入json-rpc
composer require hyperf/json-rpc
#引入JSON RPC 服务端组件
composer require hyperf/rpc-server
#引入nacos组件
composer require hyperf/service-governance-nacos
```



##### 2.1 定义服务提供者

 * ```
   <?php
   
   namespace App\JsonRpc;
   
   use Hyperf\RpcServer\Annotation\RpcService;
   
   /**
   
    * 注意，如希望通过服务中心来管理服务，需在注解内增加 publishTo 属性
      */
      #[RpcService(name: "CalculatorService", protocol: "jsonrpc-http", server: "jsonrpc-http")]
      class CalculatorService implements CalculatorServiceInterface
      {
     		 // 实现一个加法方法，这里简单的认为参数都是 int 类型
      	public function add(int $a, int $b): int
      	{
          // 这里是服务方法的具体实现
          return $a + $b;
      	}
      }
   ```

   

##### 2.2 定义 JSON RPC Server

```
 // 这里省略了该文件的其它配置
    'servers' => [
        [
            'name' => 'jsonrpc-http',
            'type' => Server::SERVER_HTTP,
            'host' => '0.0.0.0',
            'port' => 9504,
            'sock_type' => SWOOLE_SOCK_TCP,
            'callbacks' => [
                Event::ON_REQUEST => [\Hyperf\JsonRpc\HttpServer::class, 'onRequest'],
            ],
        ],
    ],
```

##### 2.3 发布到服务中心

```
<?php
return [
    'enable' => [
        'discovery' => true,
        'register' => true,
    ],
    'consumers' => [],
    'providers' => [],
    'drivers' => [
        'nacos' => [
            'host' => '宿主机IP',
            'port' => 8848,
            'username' => null,
            'password' => null,
            'heartbeat' => 5,
        ],
    ],
];
```

##### 2.4 启动

```
php bin/hyperf.php start
```

#### 3.客户端

```
#创建客户端项目
composer create-project hyperf/hyperf-skeleton client_rpc
#引入json-rpc
composer require hyperf/json-rpc
#引入JSON RPC 客户端组件
composer require hyperf/rpc-client
#引入nacos组件
composer require hyperf/service-governance-nacos
```



##### 3.1 自动创建代理消费者类

```
<?php
return [
    // 此处省略了其它同层级的配置
    'consumers' => [
        [
            // name 需与服务提供者的 name 属性相同
            'name' => 'CalculatorService',
            // 服务接口名，可选，默认值等于 name 配置的值，如果 name 直接定义为接口类则可忽略此行配置，如 name 为字符串则需要配置 service 对应到接口类
            'service' => \App\JsonRpc\CalculatorServiceInterface::class,
            // 对应容器对象 ID，可选，默认值等于 service 配置的值，用来定义依赖注入的 key
            'id' => \App\JsonRpc\CalculatorServiceInterface::class,
            // 服务提供者的服务协议，可选，默认值为 jsonrpc-http
            // 可选 jsonrpc-http jsonrpc jsonrpc-tcp-length-check
            'protocol' => 'jsonrpc-http',
            // 负载均衡算法，可选，默认值为 random
            'load_balancer' => 'random',
            // 这个消费者要从哪个服务中心获取节点信息，如不配置则不会从服务中心获取节点信息
            'registry' => [
                'protocol' => 'nacos',
                'address' => 'http://宿主机IP:8848',
            ]
        ]
    ],
];
```

##### 3.2 手动创建消费者类

```
<?php

namespace App\JsonRpc;

use Hyperf\RpcClient\AbstractServiceClient;

class CalculatorServiceConsumer extends AbstractServiceClient implements CalculatorServiceInterface
{
    /**
     * 定义对应服务提供者的服务名称
     */
    protected string $serviceName = 'CalculatorService';
    

    /**
     * 定义对应服务提供者的服务协议
     */
    protected string $protocol = 'jsonrpc-http';
    
    public function add(int $a, int $b): int
    {
        return $this->__request(__FUNCTION__, compact('a', 'b'));
    }

}
```

##### 3.3 控制器中调用

```
<?php

declare(strict_types=1);
namespace App\Controller;
use Hyperf\Context\ApplicationContext;
use App\JsonRpc\CalculatorServiceInterface;

class IndexController extends AbstractController
{

    public function index()
    {
        $client = ApplicationContext::getContainer()->get(CalculatorServiceInterface::class);
    	var_dump($client->add(1,2););
    }

}
```

##### 3.4 启动

```
#更改客户端启动端口
config\autoload\server.php中servers.port=9502
```

```
php bin/hyperf.php start
```

```
访问http://宿主机IP9502
```

