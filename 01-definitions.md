# DEPS - 데이터 정의

> [!WARNING]
> 이 문서는 아직 작성 중입니다.

본 문서에서는 DEPS 프로토콜에서 사용되는 주체, 객체, 동작들을 정의한다.

## 1. 서명과 그 표현

DEPS에서는 서명에 관해서는 항상 Ed25519를 사용한다.

- 서명 과정에서 텍스트 인코딩은 항상 UTF-8을 사용한다.
- `base64url_nopad`는 [RFC 4648에서 정의된 "Base 64 Encoding with URL and Filename Safe Alphabet"](https://datatracker.ietf.org/doc/html/rfc4648#page-8)에 **Padding 문자(`=`)를 붙이지 않은 것**을 뜻한다.
- `Blake2b-512`는 512비트의 출력값을 내어놓는 BLAKE2B 암호학적 해시 함수이다.

### 공개키의 텍스트 표현

`인코딩된 공개키`를 다음과 같이 정의한다.

```
base64url_nopad(<Ed25519 공개 키>)
```

Ed25519 공개 키의 크기는 32바이트이다. 따라서 위 표현은 43자가 된다.


### 서명의 텍스트 표현

> [!NOTE]
> 아래의 정의에서 데이터를 `Blake2b-512`로 해시하는 것은 [minisign](https://jedisct1.github.io/minisign/)의 설계를 참고한 것이다.

`인코딩된 서명`을 다음과 같이 정의한다.

```
base64url_nopad(ed25519(Blake2b-512(<데이터>)))
```

Ed25519 서명은 64바이트이며, 따라서 위 표현은 86자가 된다.

### 임의의 JSON 데이터에 대한 서명

임의의 JSON 데이터 `data`를 서명하여 다음과 같이 나타낼 수 있다.

```typescript
{
    data: {
        signedAt: Date,  // 이 데이터를 서명한 시각. 반드시 포함되어야 한다.
        // ... 임의의 데이터가 들어간다 ...
    },

    key: string,   // 서명에 사용된 인코딩된 공개키
    sign: string,  // 인코딩된 서명
}
```

> [!NOTE]
> JCS (RFC8785)는 어떤 JSON 오브젝트가 항상 같은 문자열으로 직렬화됨을 보장하기 위해 고안된 규격이다. **`data`를 JCS로 나타내는 과정을 생략하면 서명값이 달라질 수 있다!**

서명 과정은 아래와 같다.

1. `data`를 [JCS (RFC8785: JSON Canonicalization Scheme)](https://datatracker.ietf.org/doc/html/rfc8785) 형식으로 나타낸다.
2. `data를 JCS로 나타낸 값`을 `key`로 서명하여, 인코딩된 서명을 `sign`에 삽입한다.

검증 과정은 아래와 같다. 서명된 데이터를 받는 주체는 **반드시** 검증을 수행해야 한다.

1. `key`가 예측한 것과 같은지 확인한다.
2. `sign`이 `data를 JCS로 나타낸 값`을 `key`로 올바르게 서명한 것인지 확인한다.


## 2. 식별자의 형식

DEPS에서 사용되는 식별자들은 다음과 같은 형태를 가진다.

### 아이덴티티 식별자

하나의 아이덴티티를 고유하게 결정하는 불변한 값이다. 앞에 `::`가 붙은 `인코딩된 공개 키`로 나타낸다.

```
::[인코딩된 공개키]

::Cd3FRCGA9i83mlvQbO4IzT51TGPpvAucNSCh1CzM0QT
```

### 핸들네임

> [!NOTE]
> 핸들네임은 안정된 식별자가 아니다! 핸들네임은 사용자 경험을 개선하기 위한 별명과 같은 용도로만 사용되어야 한다.

홈서버에서 어떤 아이덴티티에게 부여한 별명이다. 홈서버로의 요청을 통해 아이덴티티와 상호 변환할 수 있다.

```
@[유저네임]::[홈서버 도메인]

@alice::example.org
@Neo-Vim::pshome.example.kr
```

- 핸들네임은 대소문자를 구분하지 않는다.
- [유저네임] 부분은 정규식 `[a-zA-Z0-9-]{2,18}` 으로 나타낼 수 있어야 한다.

아이덴티티와 달리 핸들네임의 연결은 홈서버에 종속적이다. 따라서 핸들네임은 프로토콜 상에서 다음과 같은 기묘한 특징을 지니게 된다.

- 핸들네임이 없는 아이덴티티가 존재할 수 있다.
- 핸들네임은 불변이 아니다. 유저의 핸들네임은 변경될 수 있다.
- 여러 홈서버가 하나의 아이덴티티에게 서로 다른 핸들네임을 붙일 가능성이 있다.
    - 단, 하나의 홈서버는 각 아이덴티티에게 하나의 핸들네임만 붙일 수 있다.
    - 한 홈서버 내에서, 한 아이덴티티가 

### 문제 식별자

어떤 문제서버의 특정한 문제를 지칭하는 식별자.

```
#[서버 내 문제 ID]::[서버 도메인]

#1000::ps.example.org
#AxcqVdCzQN::solutions.home
```

- [서버 내 문제 ID]는 정규식 `[a-zA-Z0-9-]{1,32}`으로 나타낼 수 있어야 한다.

## 3. 아이덴티티

아이덴티티는 아래에 정의된 `Identity` 타입의 데이터이다.

```ts
type IdentityInfo = {
    nickname: string,    // 닉네임
    bio: string?,        // 소개글
    avatarUrl: string?  // 프로필 사진의 URL
    signedAt: Date,     // 이 오브젝트를 서명한 날짜 (Identity 참조)
}

type Identity = {
    // 아이덴티티 식별자. Base64로 인코딩된 Ed25519 공개키.
    key: string,

    // 위의 IdentityInfo 타입.
    data: IdentityInfo,

    // info 문자열을 identity에 대응하는 비밀 키로 서명한 값.
    sign: string,
}
```

## 문제증표

> [!WARNING]
> 이 아래부터는 아직 작성 중입니다. 완성되지 않았습니다.

```ts
type SolveCertificate = {
    issuer: string,        // 문제서버 공개키
    user: string,          // 유저 공개키
    problemId: string,
    score: 100,
    solvedAt: Date,
    sign: string,
}

// TODO: key-data-sign 구조로 변경
```

