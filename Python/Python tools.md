###### tags: `Python`
# Python tools

## Concurrent futures

```
import logging
import concurrent.futures
from datetime import datetime


class FuturesThreadPool:
    log = logging.getLogger('FutureThreadPool')

    @staticmethod
    def run(workers: int, kwargs_list: list, func, callback = None):
        result_list = []
        with concurrent.futures.ThreadPoolExecutor(max_workers=workers) as executor:
        # Start the load operations and mark each future with its URL
            future_list = []
            for kwargs in kwargs_list:
                future = executor.submit(func, **kwargs)
                # custom future value
                future.input_parm = kwargs
                future.start_time = datetime.now()
                if callback:
                    # call by future completed
                    future.add_done_callback(callback)
                future_list.append(future)
            # get result if task complete
            for future in concurrent.futures.as_completed(future_list):
                try:
                    data = future.result()
                except Exception as e:
                    FuturesThreadPool.log.debug(str(e))
                else:
                    result_list.extend(data)
        return result_list

```

## Singleton

```
class Singleton(object):
    _instance = None

    def __new__(cls, *args, **kwargs):
        if not isinstance(cls._instance, cls):
            cls._instance = object.__new__(cls, *args, **kwargs)
        return cls._instance

```

## urlencode

requests套件params(query string)預設使用quote_plus做轉換
有特殊需求的話照下面的方法自定義處理

```
quote  # ' '空白改為%20, 這個為舊版本
quote_plus  # ' '空白改為+, 這個為當前版本
urlencode(parm, safe=':', quote_via=quote)  # safe某文字表示不轉換該文字且指定quote方法
```
