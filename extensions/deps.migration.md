# `deps.migration@1.0`: 아이덴티티와 데이터의 이사 방법

> [!NOTE]
> 이것은 **공식 익스텐션**이다.
>
> 익스텐션 ID: `deps.migration@1.0`

이 익스텐션은 사용자의 모든 데이터를 한 홈서버에서 다른 홈서버로 옮기는 방법을 표준화한다.

이 익스텐션은 다음을 정의한다.

- 아이덴티티 패키지 파일
- 홈서버 추가 API
    - 한 홈서버에서, 아이덴티티 패키지 파일을 반출하는 API
    - 다른 홈서버에서, 아이덴티티 패키지 파일을 반입하는 API

이 익스텐션을 구현하는 서버 간 **홈서버 이사**의 흐름은 다음과 같다.

1. 사용자는 기존 홈서버에서 아이덴티티 패키지 파일을 다운로드한다.
2. 사용자는 새 홈서버에 아이덴티티 패키지 파일을 업로드한다.
    - 새 홈서버는 아이덴티티 패키지 파일의 유효성을 검증하고, 모든 데이터를 업로드한다.

## 1. 아이덴티티 패키지 파일

> [!NOTE]
> 상호운용성을 위해, 구현체는 민감하지 않은 익스텐션 필드(임의 필드)를 보존한 채 반입/반출하는 것을 권장한다.

**아이덴티티 패키지 파일**은 한 아이덴티티와 이에 연계된 모든 데이터를 담은 JSON 파일이다. 이 파일의 구조는 다음과 같다.

```typescript
type IdentityPackage = {
    "version": "deps.migration@1.0",
    "exportedAt": Date,
    "source": string,

    "identity": Identity,
    "solves": SolveCertificate[],

    // 추가로 임의의 필드가 포함될 수 있다.
}
```

> [!NOTE]
> `source`와 `exportedAt`은 어떠한 검증도 포함되지 않는 정보 제공용 필드이다.

- `version`은 반드시 `deps.migration@1.0`이어야 한다.
- `exportedAt`은 패키지 생성 시각이다.
- `source`는 패키지를 반출한 홈서버의 도메인이다.
- `identity`, `solves`는 각각 `01-definitions.md`에서 정의된 타입과 같다.

## 2. 홈서버 추가 API

이 익스텐션을 지원하는 홈서버는 `GET /_deps/home/info`의 `extensions` 목록에 `deps.migration@1.0`을 포함해야 한다.

### 2.1. POST `/_deps/home/migration/export`

> [!NOTE]
> 홈서버는 필요에 따라 이 API와 더불어 다른 방법으로도 아이덴티티 패키지 파일을 제공할 수 있다. 이를테면 JWT나 다른 세션 인증을 활용할 수 있다.

이 홈서버에 등록된 아이덴티티의 전체 데이터를 반출한다.

#### 요청

```typescript
{
    "key": string,
    "data": {
        "signedAt": Date,
        "intent": "deps.migration@1/export",
        "server": string,
    },
    "sign": string,
}
```

- `signedAt`은 현재 시각으로 해야 한다.
- `intent`는 고정값 `"deps.migration@1/export"`이어야 한다.
- `server`는 이 요청을 받는 홈서버의 도메인이어야 한다.

#### 응답

```typescript
{
    "package": {
        "version": "deps.migration@1.0",
        "exportedAt": Date,
        "source": string,
        "identity": Identity,
        "solves": SolveCertificate[],
        // 추가로 임의의 필드가 포함될 수 있다.
    }
}
```

- 서버가 일부 문제증표를 사용자의 요청에 따라 비공개하는 경우에는, 그럼에도 불구하고 `solves` 배열은 비공개 증표까지 전부 포함해야 한다.

#### 서버의 필수 검증

1. `intent`가 `"deps.migration@1/export"`인지 확인한다.
2. `server`가 이 홈서버의 도메인과 같은지 확인한다.
3. `signedAt`이 받아들일 수 있는 범위 (e.g. 30초) 내에 있는지 확인한다.
4. `key`가 이 홈서버에 등록된 아이덴티티 식별자인지 확인한다.
5. `sign`이 올바른 서명인지 검증한다.

#### 오류 응답

- 등록되지 않은 아이덴티티라면, **404 Not Found** (`E_NOT_FOUND`)
- `intent` 또는 `server` 검증에 실패하면, **400 Bad Request** (`E_INVALID_REQUEST`)
- `signedAt`이 허용 범위를 벗어나면, **400 Bad Request** (`E_OUTDATED_SIGN`)
- `sign` 검증에 실패하면, **403 Forbidden** (`E_INVALID_SIGN`)


### 2.2. POST `/_deps/home/migration/import`

> [!NOTE]
> 홈서버는 필요에 따라 패키지를 반입하는 다른 API를 제공할 수 있다.
>
> 이 엔드포인트는 연산을 많이 요구할 수 있다. 따라서 구현체는 적절한 Rate-Limit을 적용하는 것을 권장한다.

아이덴티티 패키지 파일을 이 홈서버로 반입한다.

#### 요청

```typescript
{
    // 반입 요청 자체에 대한 아이덴티티 서명
    "key": string,
    "data": {
        "package": {
            "version": "deps.migration@1.0",
            "exportedAt": Date,
            "source": string,
            "identity": Identity,
            "solves": SolveCertificate[],
            // 추가로 임의의 필드가 포함될 수 있다.
        },
        "signedAt": Date,
        "intent": "deps.migration@1/import",
        "server": string,
    },
    "sign": string,
}
```

- `key`는 이 요청을 받는 홈서버에 등록된 아이덴티티 식별자여야 한다.
    - `key`와 `data.package.identity`가 가리키는 아이덴티티는 같아야 한다.
- `data.package`는 `2.1`에서 정의된 아이덴티티 패키지 파일과 동일한 구조를 가진다.
- `data.signedAt`은 현재 시각으로 해야 한다.
- `data.intent`는 고정값 `"deps.migration@1/import"`이어야 한다.
- `data.server`는 이 홈서버의 도메인이어야 한다.

#### 응답

```typescript
{
    "ok": true,
}
```

반입 요청이 성공적으로 접수되었음을 나타낸다. 서버는 반입 처리를 비동기적으로 수행할 수 있기 때문에 반입 결과는 즉시 적용되지 않을 수 있다.

#### 서버의 필수 검증

1. `key`가 이 홈서버에 등록된 아이덴티티인지 확인한다.
2. 요청의 유효성을 검증한다.
    - `data.intent`가 `"deps.migration@1/import"`인지 확인한다.
    - `data.server`가 이 홈서버의 도메인과 같은지 확인한다.
    - `data.signedAt`이 받아들일 수 있는 범위 (e.g. 30초) 내에 있는지 확인한다.
    - `key`와 `data.package.identity`가 같은 아이덴티티를 가리키는지 확인한다.
3. `data.package.identity`의 서명이 올바른지 검증한다.

#### 반입 동작 규칙

서버는 검증에 성공한 경우 다음 규칙으로 반입을 수행한다.

1. 아이덴티티의 `signedAt`을 비교한다.
    - 패키지의 `identity.data.signedAt`이 더 미래라면 갱신한다.
    - 같거나 과거라면 기존 값을 유지한다.
2. `solves`는 각 항목을 독립적으로 반입한다.
    - 서버는 기존 `PUT _deps/home/identities/[identity]/solves`와 동일한 기준으로 중복을 판정한다.
    - 중복이 아니며 **문제증표 검증**을 통과한 증표만 저장한다.
    - 반입하지 못한 증표는 버려진다.

서버는 이러한 동작을 비동기적으로 처리할 수 있다.

#### 오류 응답
- 등록되지 않은 아이덴티티라면, **404 Not Found** (`E_NOT_FOUND`)
- 요청의 유효성 검증에 실패하면, **400 Bad Request** (`E_INVALID_REQUEST`)
- `signedAt`이 허용 범위를 벗어나면, **403 Forbidden** (`E_OUTDATED_SIGN`)
- `data.package.identity`의 서명 검증에 실패하면, **403 Forbidden** (`E_INVALID_SIGN`)

