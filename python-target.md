## 서비스 구현하기

예제로 작성된 니름 패키지를 Python과 함께 사용해봅시다. 니름 패키지를
Python 에서 사용할 수 있도록 컴파일합니다.


```shell
$ nirum -o todo-python -t python todo-schema
```

니름 패키지는 Python 패키지로 컴파일되기때문에 pip으로 설치해서 사용할 수
있습니다.

```shell
$ python3.6 -m venv todo
$ source todo/bin/activate
$ pip install todo-python
```

니름에서 컴파일된 Python 패키지는 서비스의 모양을 정의하는
패키지이므로 특별한 기능이 없습니다. 서비스의 실제 기능을 구현하기 위해서
니름이 생성한 패키지의 클래스를 상속받아 서비스를 구현합니다.

```python
""":mod:`todo` --- TODO Service

"""
import datetime
import uuid

from todo_schema import List, Todo_Service


class TodoService(TodoService):
    """TODO Service implementation."""

    def __init__():
        self.db = {
            'list': {},
        }

    def create_list(self, name: str) -> List:
        l = List(
            id=uuid.uuid4(),
            name=name,
            tasks=[],
            created_at=datetime.datetime.now(datetime.timezone.utc)
        )
        self.db['list'][l.id] = l
        return l
```

DB를 대신해 메모리에 저장하는 것을 가정하고 할 일을 만드는 메소드를
구현했습니다. 할 일을 추가하는 메소드도 구현해보겠습니다.

```python
from todo_schema import List, ListNotFound, Task, Todo, Todo_Service


class TodoService(TodoService):

    ...

    def add_task(self, list_id: uuid.UUID, task: Task) -> List:
        if list_id not in self.db['list']:
            raise ListNotFound(
                'list {!s} is not created yet.'.format(list_id)
            )
        l = self.db['list'][list_id]
        l.tasks.append(task)
        return l
```

`add_task` 메소드는 주어진 고유 식별자와 일치하는 할 일의 목록을 찾지못하면,
에러를 내야합니다. `ListNotFound`은 `@error` annotation을
가지고 있으므로 `Exception`을 상속하게됩니다. 주어진 에러 클래스를 사용해
예외 처리를 진행하면됩니다.

`@error` annotation은 컴파일되는 타겟 언어에게 특정 타입이 에러와 관련된
타입인 것을 알려줍니다. 타겟 언어는 해당 언어에서 에러와 관련된 타입으로
표현합니다.


### 니름 서비스를 WSGI와 사용하기

니름 스키마를 상속받아서 서비스를 구현했습니다. 다음은 서비스를
웹 어플리케이션으로 만들어 배포해야합니다.
[nirum-wsgi](https://pypi.org/project/nirum-wsgi/)를 사용하여 니름 서비스를
WSGI 어플리케이션으로 만들 수 있습니다.


```
$ pip install nirum-wsgi
$ nirum-server -H 0.0.0.0 -p 8080 --debug 'todo:TodoService()'
```

`nirum-server` 명령어는 니름 서비스를 WSGI 어플리케이션으로 만든 후
[`werkzeug.serving.run_simple`][werkzeug-run-simple] 함수를 호출하여
서빙합니다.

[werkzeug-run-simple]: http://werkzeug.pocoo.org/docs/0.12/serving/

만약 다른 네트워크 라이브러리를 쓰고싶거나, 다른 프레임워크와 같이 사용하고
싶다면 `nirum_wsgi.WsgiApp`을 사용하여 WSGI 어플리케이션으로 만들 수 있습니다.


#### 니름 서비스를 Flask와 같이 사용하기

`nirum_wsgi.WsgiApp`을 사용하면 [Flask](http://flask.pocoo.org/)와 같이
니름 서비스를 웹 어플리케이션으로 배포할 수 있습니다. 간단한
Flask 어플리케이션을 예시로 작성해보겠습니다.


```python
from flask import Flask


app = Flask(__name__)


@app.route('/', methods=['GET'])
def index():
    return '''<html>
      <head>
        <title>Welcome</title>
      </head>

      <body>
        <h1>Yippee-ki-yay</h1>
      </body>
    </html>'''
```


Flask의 핸들러는 핸들러 안에서 [문서에 적혀있는][flask-response] 타입에
맞는 값을 반환하면, 응답 객체로 변환하여 응답합니다. 문서에 적혀있듯이
Flask 핸들러 내부에서는 WSGI 어플리케이션을 반환할 수 있습니다.

```
from flask import Flask, url_for
from nirum_wsgi import WsgiApp
from werkzeug.wsgi import DispatcherMiddleware

from .service import TodoService


@app.route('/api/', defaults={'path': ''},
           methods=['GET', 'POST', 'PUT', 'DELETE'])
@app.route('/api/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def service_entrypoint(path: str):
    wsgi_app = WsgiApp(TodoService())
    route = url_for('.service_entrypoint', path='').rstrip('/')
    return DispatcherMiddleware(wsgi_app, {route: wsgi_app})
```

위와 같이 구성하면 이미 Flask를 사용하고 있는 어플리케이션과 함께 니름으로
API 서버를 구현할 수 있습니다.

[flask-response]: http://flask.pocoo.org/docs/0.11/quickstart/#about-responses


## 배포된 서비스에 요청하기

위에서 배포한 서비스에 요청하려면 [`nirum-http`][nirum-http] 패키지를
사용할 수 있습니다. 니름 패키지에서 자동으로 생성된 클라이언트 클래스를
사용합니다.

```
import datetime
import uuid

from todo_schema import List, Markdown, Task, TodoService_Client
from nirum_http import HttpTransport


transport = HttpTransport('https://service-host/')
client = TodoService_Client(transport)


if __name__ == '__main__':
    l = client.create_list('My list')
    my_list = client.add_task(
        l.id,
        Task(
            id=uuid.uuid4(),
            title='Write tutorial',
            content=Markdown('''## Index

            1. Installation
            2. Syntax

            ### Installation
            ...''',
            created_at=datetime.datetime.now(datetime.timezone.utc),
            status=Todo()
        )
    )
    assert my_list.tasks[0].title == 'Write tutorial'
```

현재는 HTTP만 지원하고 있지만, 이후에는 MQ 같은 인터페이스도 지원할 예정입니다.

[nirum-http]: (https://github.com/spoqa/nirum-python-http)


