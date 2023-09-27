+++
title = 'Python Regex Example'
date = 2023-09-27T10:43:29+08:00
draft = true
+++

## Python Regex Example Record

### 从日志文本中提取字典结构数据

```python

import re
import json

s = '''[2023-04-19 10:18:36,070] [73709] [INFO] {"request": "GET /hello/?format=json HTTP/1.1", "status": "200", "bytes": "30", "remote_address": "127.0.0.1","duration": "0.883924", "referer": "-", "agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36"}'''

pattern = r"{.+[:,].+}"

items = re.findall(pattern, s)

for item in items:
    print(json.loads(item))
```
