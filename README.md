<p align="center"><img width="260px" src="https://chaz6chez.cn/images/workbunny-logo.png" alt="workbunny"></p>

**<p align="center">workbunny/webman-rabbitmq</p>**

**<p align="center">🐇 A PHP implementation of RabbitMQ Client for webman plugin. 🐇</p>**

# A PHP implementation of RabbitMQ Client for webman plugin


## 使用

```
composer require workbunny/webman-rabbitmq
```

## 创建Builder

**Builder** 可以理解为类似 **ORM Model**，创建一个 **Builder** 就对应了一个队列；
使用该 **Builder** 对象进行 **publish()** 时，会向该队列投放消息；

创建多少个 **Builder** 就相当于创建了多少条队列；**注： 前提是将所创建的 Builder
加入了 webman 自定义进程配置 porcess.php**


- 继承FastBuilder
- 实现handler方法
- 重写属性【可选】

以下以 **TestBuilder** 举例：

```php
use Workbunny\WebmanRabbitMQ\FastBuilder;

class TestBuilder extends FastBuilder
{
    // QOS计数 【可选， 默认0】
    protected int $prefetch_count = 1;
    // QOS大小 【可选， 默认0】
    protected int $prefetch_size = 2;
    // 是否全局 【可选， 默认false】
    protected bool $is_global = true;
    // 是否延迟队列 【可选， 默认false】
    protected bool $delayed = false;
    
    public function handler(\Bunny\Message $message,\Bunny\Channel $channel,\Bunny\Client $client) : string{
        var_dump($message->content);
        return Constants::ACK;
        # Constants::NACK
        # Constants::REQUEUE
    }
}
```

## 实现消费

**消费是异步的，不会阻塞当前进程，不会影响webman/workerman的status；**

1. 将 **TestBuilder** 配置入 **Webman** 自定义进程中

```php
return [
    'test-builder' => [
        'handler' => \Examples\TestBuilder::class,
        'count'   => cpu_count(), # 建议与CPU数量保持一致，也可自定义
    ],
];
```

2. 启动 **webman** 后会自动创建queue、exchange并进行消费，连接数与配置的进程数 **count** 相同

## 实现生产

- 每个builder各包含一个连接，使用多个builder会创建多个连接

- 生产消息默认不关闭当前连接

- 异步生产的连接与消费者共用

### 同步生产

**该方法会阻塞等待至消息生产成功，返回bool**

- 向 **TestBuilder** 队列发布消息

```php
use Examples\TestBuilder;

$builder = TestBuilder::instance();
$message = $builder->getMessage();
$message->setBody('abcd');
$builder->syncConnection()->publish($message); # return bool
```

- 使用助手函数向 **TestBuilder** 发布消息

```php
use function Workbunny\WebmanRabbitMQ\sync_publish;
use Examples\TestBuilder;

sync_publish(TestBuilder::instance(), 'abc'); # return bool
```

### 异步生产

**该方法不会阻塞等待，立即返回 promise，可以利用 promise 进行 wait；也可以纯异步不等待**

- 向 **TestBuilder** 队列发布消息

```php
use Examples\TestBuilder;

$builder = TestBuilder::instance();
$message = $builder->getMessage();
$message->setBody('abcd');
$builder->connection()->publish($message); # return PromiseInterface|bool
```

- 使用助手函数向 **TestBuilder** 发布消息

```php
use function Workbunny\WebmanRabbitMQ\async_publish;
use Examples\TestBuilder;

async_publish(TestBuilder::instance(), 'abc'); # return PromiseInterface|bool
```

## 延迟队列

延迟队列需要借助RabbitMQ的插件实现，所以需要先给RabbitMQ安装相关支撑插件。

### 安装插件

1. 进入 rabbitMQ 的 plugins 目录下执行命令下载插件（以rabbitMQ 3.8.x举例）：

```shell
wget https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/3.8.17/rabbitmq_delayed_message_exchange-3.8.17.8f537ac.ez
```

2. 执行安装命令

```shell
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

### 使用

1. 继承重写 **Builder** 的 **delayed** 属性：

```php
use Workbunny\WebmanRabbitMQ\FastBuilder;

class DelayBuilder extends FastBuilder
{
    protected bool $delayed = true;
    
    public function handler(\Bunny\Message $message,\Bunny\Channel $channel,\Bunny\Client $client) : string{
        var_dump($message->content);
        return Constants::ACK;
        # Constants::NACK
        # Constants::REQUEUE
    }
}
```

2. 生产者添加自定义头部 **x-delay** 实现延迟消息，单位毫秒：

   - 同步生产
     ```php
     use Examples\DelayBuilder;

     $builder = DelayBuilder::instance();
     $message = $builder->getMessage();
     $message->setBody('abcd');
     $message->setHeaders(array_merge($message->getHeaders(), [
         'x-delay' => 10000, # 延迟10秒
     ]));
     $builder->syncConnection()->publish($message); # return bool
     ```

     ```php
     use function Workbunny\WebmanRabbitMQ\sync_publish;
     use Examples\DelayBuilder;

     sync_publish(DelayBuilder::instance(), 'abc', [
         'x-delay' => 10000, # 延迟10秒
     ]); # return bool
     ```
   - 异步同理

3. 将 **DelayBuilder** 加入 webman 的 process.php 配置中，启动 webman