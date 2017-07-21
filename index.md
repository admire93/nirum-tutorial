# 니름으로 TODO API 서버 만들기

할 일을 관리하는 API 서버를 니름으로 구현합니다. 그리고 자연스럽게 니름으로
API 서버를 어떻게 구현하는지 소개합니다.


## 기본 스키마 결정하기

우리의 TODO API 서버는 할 일과 할 일의 목록을 저장할 수 있어야합니다.
2가지 모델이 어떤 성질을 가지고 있는지 정의합니다. 그리고 이런 성질들을
니름에서 어떻게 표현하는지 소개합니다.

- 할일(task)은 아래에 나열한 속성들을 가지고 있습니다.
  - 고유 식별자
  - 제목
  - 내용
    - 내용은 markdown로 작성됩니다.
  - 생성일

```nirum
# 니름에서 문서의 작성은 `#`으로 시작합니다.
unboxed markdown (text);
# 마크다운 형식의 텍스트.

record task (
    # 할 일. 여러 속성을 가질 수 있는 `record`로 정의합니다.
    uuid id,
    # 할 일의 고유한 식별자.
    text title,
    # 할 일의 제목.
    markdown content,
    # 할 일의 내용, 마크다운 형식으로 작성합니다.
    datetime created-at,
    # 할 일의 생성 시각.
);
```

- 할 일은 2가지로 나누어집니다.
  - 아직 하지 못한 일(todo)
  - 완료된 일(done)
    - 완료된 일은 완료된 시점(done-at)을 가지고 있습니다.

```nirum
union status
    # 할 일의 완료 여부.
    = todo
    # 미완.
    | done ( datetime done-at,
             # 완료 시점.
           )
    # 완료.
    ;
record task (
    # 할 일. 여러 속성을 가질 수 있는 `record`로 정의합니다.
    uuid id,
    # 할 일의 고유한 식별자.
    text title,
    # 할 일의 제목.
    markdown content,
    # 할 일의 내용, 마크다운 형식으로 작성합니다.
    datetime created-at,
    # 할 일의 생성 시각.
    status status,
    # 할 일의 상태.
);
```

- 리스트(list)는 아래의 속성들을 가지고 있습니다.
  - 고유 식별자
  - 이름
  - 완료되거나/되지못한 일들의 목록
  - 생성일

```nirum
record list (
    # 할 일의 목록.
    uuid id,
    # 목록의 고유 식별자.
    text name,
    # 목록의 이름.
    [task] tasks,
    # 할 일 목록.
    datetime created-at,
    # 목록 생성 시점.
);
```


## RPC 스키마 결정하기

RPC 스키마를 결정하기 위해서는 서비스에서 어떤 행동들을 할지 결정해야합니다.

```nirum
service todo-service (
# TODO API 서비스
);
```

서비스는 리스트의 이름을 받아서 리스트를 생성합니다.

```nirum
service todo-service (
# TODO API 서비스
    list create-list (
        # 할 일 목록을 생성하고 생성한 할 일 목록을 반환합니다.
        text name,
        # 할 일 목록의 이름.
    ),
);
```

서비스는 리스트와 할 일을 받아서 리스트에 할 일을 등록합니다.

```nirum
service todo-service (
    # 할 일 목록 서비스
    list create-list (
        # 할 일 목록을 생성하고 생성한 할 일 목록을 반환합니다.
        text name,
        # 할 일 목록의 이름.
    ),
    list add-task (
        # 할 일을 생성하고 목록에 추가합니다.
        uuid list-id,
        # 목록의 고유 식별자.
        task task,
        # 목록에 추가할 할 일.
    ),
)
```

서비스가 할 일을 추가할 때 할 일 목록을 찾지 못한다면, 에러를 내야합니다.

```nirum
@error
record list-not-found (
    # 찾는 목록이 없음.
    uuid list-id,
    # 찾으려고 했던 목록의 고유 식별자.
);
service todo-service (
    # 할 일 목록 서비스
    list create-list (
        # 할 일 목록을 생성하고 생성한 할 일 목록을 반환합니다.
        text name,
        # 할 일 목록의 이름.
    ),
    list add-task (
        # 할 일을 생성하고 목록에 추가합니다.
        uuid list-id,
        # 목록의 고유 식별자.
        task task,
        # 목록에 추가할 할 일.
    ) throws list-not-found,
)
```

서비스는 할 일을 받아서 완료/진행 중 상태로 만듭니다. 받았던 할 일을 찾을 수
없다면 에러를 발생시킵니다. 이제 서비스가 필요한 타입과 메소드들을
정의했습니다. 완성된 니름 파일은 다음과 같습니다.

```nirum
unboxed markdown (text);
# 마크다운 형식의 텍스트.
record task (
    # 할 일
    uuid id,
    # 할 일의 고유한 식별자.
    text title,
    # 할 일의 제목.
    markdown content,
    # 할 일의 내용, 마크다운 형식으로 작성합니다.
    datetime created-at,
    # 할 일의 생성 시각.
    status status,
    # 할 일의 상태.
);
union status
    # 할 일의 완료 여부.
    = todo
    # 미완.
    | done ( datetime done-at,
             # 완료 시점.
           )
    # 완료.
    ;
record list (
    # 할 일의 목록.
    uuid id,
    # 목록의 고유 식별자.
    text name,
    # 목록의 이름.
    [task] tasks,
    # 할 일 목록.
    datetime created-at,
    # 목록 생성 시점.
);
@error
record list-not-found (
    # 찾는 목록이 없음.
    uuid list-id,
    # 찾으려고 했던 목록의 고유 식별자.
);
@error
record task-not-found (
    # 찾는 할 일이 없음.
    uuid task-id,
    # 찾으려고 했던 할 일의 고유 식별자.
);
service todo-service (
    # 할 일 목록 서비스
    list create-list (
        # 할 일 목록을 생성하고 생성한 할 일 목록을 반환합니다.
        text name,
        # 할 일 목록의 이름.
    ),
    list add-task (
        # 할 일을 생성하고 목록에 추가합니다.
        uuid list-id,
        # 목록의 고유 식별자, 만약 찾으려는 목록이 없다면
        # `list-not-found` 오류가 납니다.
        task task,
        # 목록에 추가할 할 일.
    ) throws list-not-found,
    task update-status (
        # 할 일의 상태를 전환합니다.
        uuid task-id,
        # 상태를 전환할 할 일의 고유 식별자. 만약 찾으려는 할 일이 없다면
        # 않는다면, `task-not-found` 오류가 납니다.
        status status,
        # 전환할 새 상태.
    ) throws task-not-found,
);
```


## 니름 패키지 만들기

package.toml 만드는법?


## WSGI 어플리케이션 만들기

위에서 작성된 니름 패키지를 Python과 함께 사용해봅시다. 니름 패키지를
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

니름 런타임 [0.5.0](https://pypi.org/project/nirum/0.5.0/)  이상에서
`nirum-server` 커맨드를 제공합니다. 다음과 같이 실행합니다.

```
$ nirum-server todo_schema:Todo_Service
* Running on http://0.0.0.0:9322/ (Press CTRL+C to quit)
```

하지만, 니름에서 컴파일된 Python 패키지는 서비스의 모양을 정의하는
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
from todo_schema import List, ListNotFound, Task, Todo_Service


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
