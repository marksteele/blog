+++
title = "complex event processing for fun and profit"
date = "2013-06-10"
slug = "2013/06/10/complex-event-processing-for-fun-and-profit"
Categories = []
+++

As an exercise to keep my mind nimble, here.s a write-up on how to use the power of computers to take over the world by out-foxing those slow moving meatbags who do stock trading and compete with skynet on making the most possible profit.

The pieces of this puzzle are:

- A messaging backbone (we.ll use AMQP with the RabbitMQ broker)
- A complex event processing engine (Esper)
- A way to express our greed (EPL statements)
- A software that ties this all together called new-hope (partially written by yours truly)
- A feed of stock prices
- An app to view the actions we must take.
<!--more-->
Let's get everything installed.

On centos with the EPEL repo available:

```bash
yum install php-pecl-amqp
rpm -i http://www.rabbitmq.com/releases/rabbitmq-server/v3.1.1/rabbitmq-server-3.1.1-1.noarch.rpm
rabbitmq-plugins enable rabbitmq_management
/etc/init.d/rabbitmq-server start
rabbitmqctl add_vhost /stockexchange
rabbitmqctl add_user skynet skynet
rabbitmqctl set_permissions -p /stockexchange skynet ".*" ".*" ".*"
```

Need ruby installed with rvm (jruby). Figure it out!

Install new-hope

```bash
git clone https://github.com/marksteele/new-hope.git
```

Next some gems (or run the install gems shell script in the new-hope repo)

``` bash Installing dependancies
gem install rake --version 0.9.2.2
gem install amq-protocol --version 0.9.0
gem install eventmachine --version 0.12.10
gem install amq-client --version 0.9.2
gem install amqp --version 0.9.4
gem install autotest-growl --version 0.2.16
gem install diff-lcs --version 1.1.3
gem install growl --version 1.0.3
gem install json --version 1.5.0
gem install json-jruby --version 1.5.0
gem install rspec-core --version 2.8.0
gem install  rspec-expectations --version 2.8.0
gem install rspec-mocks --version 2.8.0
gem install rspec --version 2.8.0
```

The new hope configs:

config/amqp.cfg

```
amqp_settings :host => "127.0.0.1",
           :port => "5672",
           :user => "skynet",
           :pass => "skynet",
           :vhost => "/stockexchange",
           :exchange_input => "esper-in",
           :exchange_output => "esper-out"
```

config/hope.epl

```sql
@Name('01-avgdatastream-silent')
INSERT INTO
  AvgDataStream
select
  symbol,
  avg(price) as SMA,
  stddev(price) as SD,
  last(price) as price
from
  stocktick.std:groupwin(symbol).win:time(5 days)
GROUP BY
  symbol
OUTPUT
  EVERY 10 minutes;

@Name('02-bollinger')
SELECT
  symbol,
  SMA - 2*SD as LowerBand,
  SMA as MiddleBand,
  SMA + 2*SD as UpperBand,
  price,
  (4*SD)/SMA as Bandwidth
FROM
  AvgDataStream;
```

config/hope.xml:

```xml  
<?xml version="1.0" encoding="UTF-8"?>
<esper-configuration xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="http://www.espertech.com/schema/esper"
    xsi:noNamespaceSchemaLocation="esper-configuration-4-0.xsd">

    <event-type name="stocktick">
                <java-util-map>
                        <map-property name="symbol" class="string"/>
                        <map-property name="price" class="double"/>
                        <map-property name="ask" class="double"/>
                        <map-property name="bid" class="double"/>
                </java-util-map>
        </event-type>
</esper-configuration>
```

A cheapy little stock price fetcher (generator.php):

```php
<?php
$amqp = new AMQPConnection(
  array(
    'host' => 'localhost',
    'vhost' => '/stockexchange',
    'port' => 5672,
    'login' => 'skynet',
    'password' => 'skynet'
  )
);

$amqp->connect();
$ch = new AMQPChannel($amqp);
$ex = new AMQPExchange($ch);
$ex->setName('esper-in');
$ex->setType(AMQP_EX_TYPE_FANOUT);
$ex->declare();

while(true) {
  $payload = '';
  $file = file_get_contents(
   'http://download.finance.yahoo.com/d/quotes.csv?s=RHT+MSFT+GOOG+COST+KO+AMZN&f=b2b3l1s'
  );
  foreach (explode("\r\n",$file) as $line) {
    $data = str_getcsv($line);
    if (count($data) == 4) {
      $payload .= json_encode(
        array(
          'symbol' => $data[3], 
          'ask' => floatval($data[0]), 
          'bid' => floatval($data[1]), 
          'price' => floatval($data[2]),
          'type' => 'stocktick'
        )
      ) . "\n";
    }
  }
  $ex->publish($payload,'stocktick');
  sleep(1);
}
```

A cheapy little event viewer (viewer.php):

```php
<?php
$amqp = new AMQPConnection(
  array(
    'host' => 'localhost',
    'vhost' => '/stockexchange',
    'port' => 5672,
    'login' => 'skynet',
    'password' => 'skynet'
  )
);

$amqp->connect();

$ch = new AMQPChannel($amqp);
$q = new AMQPQueue($ch);
$q->setName('esper_output');
$q->setFlags(AMQP_DURABLE|AMQP_PASSIVE);
$q->declare();
$q->bind('esper-out','#');
$q->consume('getItem',AMQP_AUTOACK);

function getItem($msg) {
  print_r($msg->getBody());
  echo "\n";
}
```

I forget if you have to manually create the exchanges in rabbitmq. If so, that.s pretty straight forward in the rabbitmq management web console.

Running new-hope:


```bash
bin/hope
```

And the generator

```bash
php generator.php
```

And the viewer

```bash
php viewer.php
```

Should produce something like this every 10 minutes or so:

```  
stdClass Object
(
    [new_events] => Array
        (
            [0] => stdClass Object
                (
                    [event_type] => anonymous_75420a3c-b91a-435d-bcc9-ba4a79b23519_result_
                    [event_properties] => Array
                        (
                            [0] => symbol
                            [1] => LowerBand
                            [2] => MiddleBand
                            [3] => UpperBand
                            [4] => price
                            [5] => Bandwidth
                        )

                    [event] => stdClass Object
                        (
                            [price] => 281.07
                            [symbol] => AMZN
                            [MiddleBand] => 281.07
                            [LowerBand] => 281.06999184383
                            [UpperBand] => 281.07000815617
                            [Bandwidth] => 5.803654484539E-8
                        )

                )

        )

    [old_events] => Array
        (
        )

    [name] => 02-bollinger
)
```

Phew! So every 10 minutes, our little viewer will output the Bollinger band data for our selected stocks (GOOG,MSFT,RHT,AMZN,COST,KO). A saavy 
investor watching this might want to buy stocks when the current price is close to the lower bands, and sell when the stocks are close to the 
upper bands, as those indicate that the prices are close to the 5 day variances.

See here for details on the web service to pull the stock data.


