```
composer require hanwenbo\rabbitmq
```

第一步-配置：
```php
    'RABBITMQ' => [
        'host' => '172.22.0.2',
        'port' => 5672,
        'user' => 'jddtest',
        'passwd' => 'jddtest',
        'vhost' => '/' ,  // 自己创建的任意host名字
    ],
    'RABBITMQ_CHANNEL' => [
        'exchange' => 'exchange',
        'queue' => 'queue',
        'routekey' => '',
        'connectionPoolName' => 'rabbitmq-pool'
    ],
```

第二步-EasySwooleEvent.php文件中initialize方法中注册：
```php
class EasySwooleEvent implements Event
{

    public static function initialize()
    {
        // TODO: Implement initialize() method.
        date_default_timezone_set('Asia/Shanghai');
        // rabbitmq-pool
        $rabbitmqPoolConfig = RabbitMQ::getInstance()->register('rabbitmq-pool', new RabbitMQConfig(Config::getInstance()->getConf('RABBITMQ')));
        $rabbitmqPoolConfig->setMinObjectNum(1);
        $rabbitmqPoolConfig->setMaxObjectNum(2);
        $rabbitmqPoolConfig->setIntervalCheckTime(30);
        $rabbitmqPoolConfig->setMaxIdleTime(30000000);
        // 废弃了
//        $rabbitmqChannelPoolConfig = RabbitMQChannel::getInstance()->register('rabbitmq-channel-pool', new RabbitMQChannelConfig(Config::getInstance()->getConf('RABBITMQ_CHANNEL')));
//        $rabbitmqChannelPoolConfig->setMaxIdleTime(3000000);
    }
```
第三步-使用：
```php
    public function rabbitmq()
    {
        $request = $this->request();
        $params = $request->getRequestParam();
        //rabbitmq-pool
        go(function () use ($params) {
						$this->channelInvoke($params);
        });
        $this->writeJson(200, $params);
    }

    private function channelInvoke($params)
    {
        try {
            $res = RabbitMQChannel::invoke('rabbitmq-channel-pool', function (AMQPChannel $channel) use ($params) {
                $exchange = 'exchange';
                $messageBody = "这是内容1";
                $message = new AMQPMessage($messageBody, array('content_type' => 'text/plain', 'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT));
                try {
                    $channel->basic_publish($message, $exchange);
                    return true;
                } catch (\Exception $exception) {
                    var_dump($exception->getMessage());
                    return false;
                }
            });
            return $res;
        } catch (\Exception $exception) {
            var_dump($exception->getMessage());
            return false;
        }
    }
```