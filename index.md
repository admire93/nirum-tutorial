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
