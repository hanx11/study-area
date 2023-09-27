+++
title = 'Falcon ASGI API Performance Test'
date = 2023-09-27T14:19:04+08:00
draft = true
+++

公司最近与第三方合作的一款硬件产品需要每隔三秒请求服务器上报设备状态，
并且每隔十秒可能需要通过服务器获取比较大量的数据到设备端进行运算。
这对服务器的并发能力提出了很大的挑战。

目前服务端使用的框架是Django 1.11.4版本，无法支持ASGI。
因此需要寻找一款并发性能可以满足需求且易于使用的ASGI框架。

由于团队成员之前一直使用的框架都是Django，大家对Django都很熟悉。
Django ORM与Django Rest Framework提供的很多特性也可以让开发同学快速高效的写出
安全的代码。所以一开始倾向与将Django升级到3.X版本，这样可以使用Django Channel。

不过，在尝试使用Django Channel框架实现几个异步接口之后，发现Django Channel在使用ORM进行
数据库操作的时候，需要使用`channels.db.database_sync_to_async`装饰器对数据操作方法进行封装。
这一点让人感觉有点不够优雅，并且实际测试下来发现性能也并没有很大的提升，于是放弃使用Django 3.X版本的想法。

之后，又使用Falcon ASGI框架实现接口进行测试，实际测试发现Falcon ASGI的并发性能相对与Django Channel
有很大的提升，并且代码实现也相对要简洁很多。这一点深得团队成员的喜爱，也很符合Python哲学。
> Simple is better than complex

测试方法说明：

实现两个接口，分别返回不同长度数据，接口A返回100B，接口B返回200KB。

在本地通过脚本实现多客户端同时并发请求上述两个接口，脚本动态配置参数需求（以下参数需支持动态配置）： 
- 模拟时长，精度：分钟；默认配置：10分钟。
- 模拟的同时生效的客户端个数；默认配置：500。 
- 接口轮询的间隔，精度到毫秒；默认配置： 500ms 
- 两个接口的请求占比；默认配置：10:1

结果输出需求： 
 - 以客户端为单位，分别统计两个接口的请求结果（成功/失败，最好能看到错误原因，但不强求）、请求耗时（可包含网络时间） 
 - 脚本执行中，服务器的资源占用率

测试结果:
![测试结果](/falcon_asgi_performance_test_result.png)

模拟客户端请求代码:
```python
# -*- coding: utf-8 -*-

import logging
import os
import threading
import time
from concurrent.futures import ThreadPoolExecutor

import click
import redis
import requests

logging.basicConfig(
    level=logging.INFO,
    format='%(levelname)s %(asctime)s %(threadName)s [%(lineno)s] %(message)s',
    datefmt="%Y-%m-%d %H:%M:%S",
    filename='fw_client.log',
    filemode='w'
)

redis_url = os.getenv("REDIS_URL", "redis://localhost:6379")
rds = redis.from_url(redis_url)

host = "http://localhost:8000"


class APICounter:

    @classmethod
    def add_success_count(cls, api_path):
        cache_key = 'ipp_service_success:{}'.format(api_path)
        rds.incr(cache_key, 1)

    @classmethod
    def add_fail_count(cls, api_path):
        cache_key = 'ipp_service_fail:{}'.format(api_path)
        rds.incr(cache_key, 1)

    @classmethod
    def add_req_time(cls, api_path, total_seconds):
        cache_key = 'ipp_service_resp_time:{}'.format(api_path)
        rds.lpush(cache_key, total_seconds)


def call_hello(page=1):
    api_path = '/hello'
    params = {"page": page}
    api_url = 'hello_{}'.format(page)
    try:
        resp = requests.get(host + api_path, params=params, timeout=30)
        if resp.status_code == 200:
            APICounter.add_success_count(api_url)
        else:
            APICounter.add_fail_count(api_url)
        APICounter.add_req_time(api_url, resp.elapsed.total_seconds())
    except Exception as exc:
        logging.error('call_hello_fail: %s', exc)
        APICounter.add_fail_count(api_url)

def fw_client(seconds=60):
    idx = 1
    start_time = time.time()
    while time.time() - start_time < seconds:
        call_hello()
        if idx % 10 == 0:
            call_hello(page=200)
        time.sleep(0.5)
        idx += 1
        logging.info('%s, idx: %s', threading.current_thread().name, idx)


@click.command()
@click.option('--seconds', default=60, help='Test Seconds')
@click.option('--workers', default=1000, help='MAX WORKERS')
def main(seconds=60, workers=100):
    with ThreadPoolExecutor(max_workers=workers) as executor:
        for _ in range(workers):
            executor.submit(fw_client, seconds)


if __name__ == "__main__":
    main()
```


#### 参考资料
- [ASGI-Framework-Benchmark](http://klen.github.io/py-frameworks-bench/)
- [Django Channel Document](https://channels.readthedocs.io/en/latest/introduction.html)
- [Falcon Document](https://falcon.readthedocs.io/en/stable/user/tutorial-asgi.html)
