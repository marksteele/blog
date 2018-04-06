+++
title = "benchmarking cinched"
date = "2016-02-12"
slug = "2016/02/12/benchmarking-cinched"
Categories = []
+++

Now that a first version was cut, it seemed like a good time to start looking at how Cinched performs.

And I must say, things are looking good so far...

<!-- more -->

## Test setup

3 nodes in the Cinched cluster (each 4 vCPUs 4GB RAM)

1 node running the test harness (4 vCPUs, 4GB RAM).

My testing environment is extremely unreliable (VMs running in an overprovisioned cloud environment) so it's been hard to get an extremely clear picture. Nevertheless, the numbers look promising.

The benchmark is run using my fork of basho_bench [here](https://github.com/marksteele/basho_bench) against Cinched v0.0.3. 

The test runs round robin operations cycling through the nodes in the cluster issuing requests for the various API endpoints.

API calls leverage HTTP keep-alives to amortize the cost of establishing the TCP handshake and TLS key exchange.

The document I'm encrypting is rather trivial:

```json
{"foo":"bar","bar":"baz"}
```

## Results

### Time: 60s Concurrency: 32
![cinched 1 minute](/images/cinched-bench1.png)

### Time: 1 hour, Concurrency: 32
![cinched 1 hour](/images/cinched-bench2.png)

### Time: 10 hours, Concurrency: 64
![cinched 10 hours](/images/cinched-bench3.png)


![Moar](http://i.imgur.com/JzekJNP.jpg)
