+++
title = "complex event processing for fun and profit part deux"
date = "2013-06-15"
slug = "2013/06/15/complex-event-processing-for-fun-and-profit-part-deux"
Categories = []
+++

In the first instalment, we saw how to use simple moving average. That's just too easy. 

Let's do something more complex to see how things pan out. 

Refer to the previous instalment for details on how things are setup.
<!--more-->
This time around, let.s look at using exponential moving averages to identify trending and generate 
buy/sell events. It.s probably debatable on the effectiveness of exponential moving averages vs. 
simple moving averages in generating signals to buy/sell stock.

Intuitively, you.d think that the weighting would allow for faster/better decision making, however 
literature suggests that it might also penalize on not allowing trends to gather momentum. At any 
rate, it.s an interesting problem to look at.

New hope.epl configuration:

config/hope.epl

```  
@Name('01_create-quote-window-silent')
CREATE WINDOW Quotes.win:length(1) as select * from stocktick;

@Name('02_populate-quote-window-silent')
INSERT INTO Quotes select * from stocktick;

@Name('03_ema5window-silent')
create window EMA5Window.win:length(1) as select price as ema5 from stocktick;

@Name('04_ema5-first5-silent')
insert into EMA5Window SELECT AVG(price) as ema5 from stocktick.win:firstlength(5);

@Name('05_ema5-silent')
INSERT INTO
  EMA5Window
SELECT
  ((price)*(2/6)+(4/6)*(select ema5 from EMA5Window)) as ema5
FROM
   stocktick
OUTPUT
  after 5 events;

@Name('06_ema5-silent')
SELECT
  ema5
from
  EMA5Window
output
  after 5 events
snapshot
  every 5 seconds;

@Name('07_ema10window-silent')
create window EMA10Window.win:length(1) as select price as ema10 from stocktick;

@Name('08_ema10-first10-silent')
insert into EMA10Window SELECT AVG(price) as ema10 from stocktick.win:firstlength(10);

@Name('09_ema10-silent')
INSERT INTO
  EMA10Window
SELECT
  ((price)*(2/11)+(9/11)*(select ema10 from EMA10Window)) as ema10
FROM
   stocktick
OUTPUT
  after 10 events;

@Name('10_ema10-silent')
SELECT
  ema10
from
  EMA10Window
output
  after 10 events
snapshot
  every 5 seconds;

@Name('11_ema25window-silent')
create window EMA25Window.win:length(1) as select price as ema25 from stocktick;

@Name('12_ema25-first25-silent')
insert into EMA25Window SELECT AVG(price) as ema25 from stocktick.win:firstlength(25);

@Name('13_ema25-silent')
INSERT INTO
  EMA25Window
SELECT
  ((price)*(2/26)+(24/26)*(select ema25 from EMA25Window)) as ema25
FROM
   stocktick
OUTPUT
  after 25 events;

@Name('14_ema25-silent')
SELECT ema25 from EMA25Window
output after 25 events
snapshot every 5 seconds;

@Name('BUY')
SELECT
  e5.ema5 as ema5,
  e10.ema10 as ema10,
  e25.ema25 as ema25,
  s.price as price
FROM
  EMA5Window e5,
  EMA10Window e10,
  EMA25Window e25,
  Quotes s
WHERE
  s.price > e5.ema5 AND
  s.price > e10.ema10 AND
  s.price > e25.ema25
OUTPUT
  after 25 seconds
SNAPSHOT
  every 10 second;

@Name('SELL')
SELECT
  e5.ema5 as ema5,
  e10.ema10 as ema10,
  e25.ema25 as ema25,
  s.price as price
FROM
  EMA5Window e5,
  EMA10Window e10,
  EMA25Window e25,
  Quotes s
WHERE
  s.price < e5.ema5 AND
  s.price < e10.ema10 AND
  s.price < e25.ema25
OUTPUT
  after 25 seconds
SNAPSHOT
  every 10 second;
```

We'll change the setup slightly as the above EPL only tracks one stock at a time. We''ll fudge the stock feed 
this time as it's easier/quicker to see the results. The EPL operates on a per-second basis (if you'd want 
to use this for real you probably want to tweak this).

The stock tick generator (generator.php):

``` php generator.php
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
$ex->setType(AMQP_EX_TYPE_TOPIC);
$ex->declare(AMQP_DURABLE);

while(true) {
  sleep(1);
  $payload = json_encode(
  array('symbol' => 'GOOG','price' => floatval(rand(50,100) . '.' . rand(0,99)),
'type' => 'stocktick','quoteTime' => time() * 1000)) . "\n";
  $ex->publish($payload,'stocktick');
}
```

The results viewer:

``` php viewer.php
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
$q->setFlags(AMQP_DURABLE);
$q->declare();
$q->bind('esper-out','#');
$q->consume('getItem',AMQP_AUTOACK);

function getItem($msg) {
  $payload = json_decode($msg->getBody());
  if (is_array($payload->new_events) && 
   !empty($payload->new_events) && in_array($payload->name,array('BUY','SELL'))) {
    echo $payload->name . " CURRENT PRICE: " . 
      $payload->new_events[0]->event->price . 
      " EMA5: " .  $payload->new_events[0]->event->ema5 . 
      ' EMA10: ' .  $payload->new_events[0]->event->ema10 . 
      ' EMA25: ' .  $payload->new_events[0]->event->ema25 . "\n";
  }
}
```

Running the generator/viewer/new-hope together should provide you with a feed of events that fire 
when prices cross the exponential moving averages (stop-loss and buy events).

