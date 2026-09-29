---
title: 트레이스
weight: 1
description: 애플리케이션을 통과하는 요청의 경로
default_lang_commit: 5832fe960e2d910e80c894319047f716c35bd340
cSpell:ignore: Guten
---

**트레이스**는 요청이 애플리케이션에 들어왔을 때 무슨 일이 일어나는지 전체
그림을 보여준다. 단일 데이터베이스를 쓰는 모놀리스이든, 정교한 서비스 메시든,
트레이스는 애플리케이션 안에서 요청이 거치는 전체 **경로**를 이해하는 데
필수적이다.

[스팬](#spans)으로 표현된 세 가지 작업 단위로 살펴본다.

> [!NOTE]
>
> 아래 JSON 예시는 특정 형식을 엄격히 따르지 않으며, 특히
> [OTLP/JSON](/docs/specs/otlp/#json-protobuf-encoding)은 이보다 훨씬 장황하다.

`hello` 스팬:

```json
{
  "name": "hello",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "051581bf3cb55c13"
  },
  "parent_id": null,
  "start_time": "2022-04-29T18:52:58.114201Z",
  "end_time": "2022-04-29T18:52:58.114687Z",
  "attributes": {
    "http.route": "some_route1"
  },
  "events": [
    {
      "name": "Guten Tag!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

이것은 루트 스팬으로, 전체 작업의 시작과 끝을 나타낸다. 트레이스를 가리키는
`trace_id` 필드는 있지만 `parent_id`는 없다. 이를 통해 루트 스팬임을 알 수 있다.

`hello-greetings` 스팬:

```json
{
  "name": "hello-greetings",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "5fb397be34d26b51"
  },
  "parent_id": "051581bf3cb55c13",
  "start_time": "2022-04-29T18:52:58.114304Z",
  "end_time": "2022-04-29T22:52:58.114561Z",
  "attributes": {
    "http.route": "some_route2"
  },
  "events": [
    {
      "name": "hey there!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    },
    {
      "name": "bye now!",
      "timestamp": "2022-04-29T18:52:58.114585Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

이 스팬은 인사말처럼 특정 작업을 담으며, 부모는 `hello` 스팬이다. 루트 스팬과
같은 `trace_id`를 공유하므로 같은 트레이스의 일부임을 나타낸다. 또한 `hello`
스팬의 `span_id`와 일치하는 `parent_id`를 갖는다.

`hello-salutations` 스팬:

```json
{
  "name": "hello-salutations",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "93564f51e1abe1c2"
  },
  "parent_id": "051581bf3cb55c13",
  "start_time": "2022-04-29T18:52:58.114492Z",
  "end_time": "2022-04-29T18:52:58.114631Z",
  "attributes": {
    "http.route": "some_route3"
  },
  "events": [
    {
      "name": "hey there!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

이 스팬은 이 트레이스에서 세 번째 작업을 나타내며, 앞 스팬과 같이 `hello` 스팬의
자식이다. 따라서 `hello-greetings` 스팬과 형제 관계이기도 하다.

이 JSON 세 블록은 모두 같은 `trace_id`를 공유하고, `parent_id` 필드는 계층을
나타낸다. 이것이 하나의 트레이스다!

각 스팬이 구조화된 로그처럼 보인다는 점도 주목할 만하다. 실제로 그렇기 때문이다!
트레이스를 이해하는 한 가지 방법은, 컨텍스트·상관 관계·계층 등이 내장된 구조화
로그의 모음이라고 보는 것이다. 다만 이런 "구조화 로그"는 서로 다른 프로세스,
서비스, VM, 데이터 센터 등에서 올 수 있다. 덕분에 트레이싱은 어떤 시스템이든
엔드투엔드 뷰를 표현할 수 있다.

오픈텔레메트리(OpenTelemetry)에서 트레이싱이 어떻게 동작하는지 이해하려면,
코드를 계측(instrument)할 때 관여하는 구성 요소 목록을 살펴본다.

## 트레이서 프로바이더 {#tracer-provider}

트레이서 프로바이더(때로 `TracerProvider`라고 부름)는 `Tracer`를 만드는
팩토리(factory)이다. 대부분의 애플리케이션에서는 트레이서 프로바이더를 한 번
초기화하고 수명 주기는 애플리케이션과 같다. 트레이서 프로바이더 초기화에는
리소스와 익스포터 초기화도 포함된다. 보통 오픈텔레메트리 트레이싱을 구축할 때
가장 먼저 거치는 단계다. 일부 언어 SDK는 전역 트레이서 프로바이더가 이미
초기화되어 있다.

## 트레이서 {#tracer}

트레이서는 서비스의 요청처럼 주어진 작업에서 무슨 일이 일어나는지에 대한 정보를
담은 스팬을 생성한다. 트레이서는 트레이서 프로바이더에서 만든다.

## 트레이스 익스포터 {#trace-exporters}

트레이스 익스포터는 트레이스를 소비자(consumer)에게 보낸다. 소비자는 디버깅·
개발용 표준 출력, 오픈텔레메트리 컬렉터, 또는 선택한 오픈소스·벤더 백엔드일 수
있다.

## 컨텍스트 전파 {#context-propagation}

컨텍스트 전파(context propagation)는 분산 트레이싱을 가능하게 하는 핵심
개념이다. 컨텍스트 전파를 통해 스팬이 어디서 생성되었든 서로 연관 지어
트레이스로 조립할 수 있다. 자세한 내용은
[컨텍스트 전파](../../context-propagation) 개념 페이지를 참고한다.

## 스팬 {#spans}

**스팬**은 작업(operation) 또는 실행 단위를 나타낸다. 스팬은 트레이스의 구성
요소이다. 오픈텔레메트리에서는 다음 정보를 포함한다.

- 이름
- 부모 스팬 ID(루트 스팬은 비어 있음)
- 시작·종료 타임스탬프
- [스팬 컨텍스트](#span-context)
- [속성](#attributes)
- [스팬 이벤트](#span-events)
- [스팬 링크](#span-links)
- [스팬 상태](#span-status)

스팬 예시:

```json
{
  "name": "/v1/sys/health",
  "context": {
    "trace_id": "7bba9f33312b3dbb8b2c2c62bb7abe2d",
    "span_id": "086e83747d0e381e"
  },
  "parent_id": "",
  "start_time": "2021-10-22 16:04:01.209458162 +0000 UTC",
  "end_time": "2021-10-22 16:04:01.209514132 +0000 UTC",
  "status_code": "STATUS_CODE_OK",
  "status_message": "",
  "attributes": {
    "net.transport": "IP.TCP",
    "net.peer.ip": "172.17.0.1",
    "net.peer.port": "51820",
    "net.host.ip": "10.177.2.152",
    "net.host.port": "26040",
    "http.method": "GET",
    "http.target": "/v1/sys/health",
    "http.server_name": "mortar-gateway",
    "http.route": "/v1/sys/health",
    "http.user_agent": "Consul Health Check",
    "http.scheme": "http",
    "http.host": "10.177.2.152:26040",
    "http.flavor": "1.1"
  },
  "events": [
    {
      "name": "",
      "message": "OK",
      "timestamp": "2021-10-22 16:04:01.209512872 +0000 UTC"
    }
  ]
}
```

부모 스팬 ID의 존재에서 알 수 있듯, 스팬은 중첩될 수 있다. 자식 스팬은 하위
작업을 나타내므로, 애플리케이션에서 수행된 작업을 더 정확히 담을 수 있다.

### 스팬 컨텍스트 {#span-context}

스팬 컨텍스트(span context)는 각 스팬에 있는 불변(immutable) 객체이며 다음을
포함한다.

- 스팬이 속한 트레이스를 나타내는 Trace ID
- 스팬의 Span ID
- 트레이스 정보를 담은 이진 인코딩인 Trace Flags
- 벤더별 트레이스 정보를 담을 수 있는 키-값 쌍 목록인 Trace State

스팬 컨텍스트는 [분산 컨텍스트](#context-propagation) 및 [배기지](../baggage)와
함께 직렬화·전파되는 스팬의 일부이다.

스팬 컨텍스트에 Trace ID가 있으므로 [스팬 링크](#span-links)를 만들 때 사용한다.

### 속성 {#attributes}

속성(attribute)은 스팬에 부가 정보를 추가하여 추적 중인 작업에 대한 상세 정보를
전달하는 키-값 쌍 메타데이터다.

예를 들어 스팬이 eCommerce 시스템에서 사용자 장바구니에 항목을 추가하는 작업을
추적한다면, 사용자 ID, 추가할 항목 ID, 장바구니 ID를 담을 수 있다.

스팬 생성 중이나 생성 후에 속성을 추가할 수 있다. SDK 샘플링에서 속성을 쓰려면
생성 시점에 추가하는 편이 좋다. 생성 후에 값을 넣어야 하면 해당 값으로 스팬을
갱신한다.

속성은 각 언어 SDK가 따르는 규칙이 있다.

- 키는 null이 아닌 문자열이어야 한다.
- 값은 null이 아닌 문자열, 불리언, 부동소수점, 정수, 또는 이들의 배열이어야
  한다.

또한 일반적인 작업에 흔히 쓰이는 메타데이터 이름 규칙인
[시맨틱 속성](/docs/specs/semconv/general/trace/)이 있다. 가능하면 시맨틱 속성
이름을 써서 시스템 간 메타데이터 종류를 표준화하는 것이 좋다.

### 스팬 이벤트 {#span-events}

스팬 이벤트(span event)는 스팬에 추가하는 구조화된 로그 메시지(또는 부가 정보)로
볼 수 있다. 보통 스팬 기간 중 의미 있는 한 시점을 나타낸다.

예를 들어 웹 브라우저에서 두 가지를 생각해 본다.

1. 페이지 로드 추적
2. 페이지가 상호작용이 가능해지는 시점 표시

첫 번째는 시작과 끝이 있는 작업이므로 스팬으로 추적하기에 적합하다.

두 번째는 의미 있는 한 시점을 나타내므로 스팬 이벤트로 추적하기에 적합하다.

#### 스팬 이벤트와 속성 사용 기준 {#when-to-use-span-events-versus-span-attributes}

스팬 이벤트에도 속성이 있으므로, 속성 대신 이벤트를 쓸지 항상 분명하지 않을 수
있다. 특정 타임스탬프가 의미 있는지를 기준으로 판단한다.

예를 들어 스팬으로 작업을 추적하다가 작업이 끝났을 때, 작업에서 나온 데이터를
텔레메트리에 더 넣고 싶을 수 있다.

- 작업 완료 시각이 의미 있거나 중요하면 → 스팬 이벤트에 데이터를 붙인다.
- 타임스탬프가 중요하지 않으면 → 스팬 속성으로 데이터를 붙인다.

### 스팬 링크 {#span-links}

링크는 하나의 스팬을 하나 이상의 스팬과 연관해 인과 관계를 나타내기 위해
존재한다. 예를 들어 어떤 작업이 트레이스로 추적되는 분산 시스템이 있다고 하자.

이 작업들 중 일부에 대해 추가 작업이 큐에 들어가 실행되지만, 실행은 비동기이다.
이후 작업도 트레이스로 추적할 수 있다.

후속 작업의 트레이스를 첫 트레이스와 연관하고 싶지만, 후속 작업이 언제 시작할지
예측할 수 없다. 두 트레이스를 연결하려면 스팬 링크를 쓴다.

첫 트레이스의 마지막 스팬을 두 번째 트레이스의 첫 스팬에 연결할 수 있다. 그러면
둘은 인과적으로 연관된다.

링크는 필수는 아니지만, 트레이스의 스팬끼리 연관하는 좋은 방법이다.

자세한 내용은 [스팬 링크](/docs/specs/otel/trace/api/#link)를 참고한다.

### 스팬 상태 {#span-status}

각 스팬에는 상태(status)가 있다. 가능한 값은 세 가지다.

- `Unset`
- `Error`
- `Ok`

기본값은 `Unset`이다. `Unset`이면 추적한 작업이 오류 없이 정상 완료했다는
뜻이다.

`Error`이면 추적한 작업에서 오류가 났다는 뜻이다. 예를 들어 요청을 처리하는
서버에서 HTTP 500이 발생한 경우다.

`Ok`이면 애플리케이션 개발자가 스팬을 명시적으로 오류 없음으로 표시했다는
뜻이다. 직관과 다르게, 스팬이 오류 없이 끝났다는 사실만으로 `Ok`를 설정할 필요는
없다. 그 경우는 `Unset`으로 충분하다. `Ok`는 사용자가 명시적으로 설정한 스팬
상태에 대한 분명한 "최종 판정"이다. 개발자가 스팬을 "성공" 이외로 해석되길
원하지 않을 때 유용하다.

다시 정리하면, `Unset`은 오류 없이 완료된 스팬, `Ok`는 개발자가 명시적으로
성공으로 표시한 스팬이다. 대부분 `Ok`를 명시할 필요는 없다.

### 스팬 종류(SpanKind) {#span-kind}

스팬을 만들 때 `Client`, `Server`, `Internal`, `Producer`, `Consumer` 중
하나이다. 스팬 종류(span kind)는 트레이스를 어떻게 조립할지에 대한 힌트를
트레이싱 백엔드에 준다. 오픈텔레메트리 명세에 따르면, 서버 스팬의 부모는 흔히
원격 클라이언트 스팬이고, 클라이언트 스팬의 자식은 보통 서버 스팬이다.
마찬가지로 컨슈머 스팬의 부모는 항상 프로듀서이고, 프로듀서 스팬의 자식은 항상
컨슈머이다. 지정하지 않으면 스팬 종류는 internal로 간주한다.

SpanKind에 대한 자세한 내용은 [SpanKind](/docs/specs/otel/trace/api/#spankind)를
참고한다.

#### Client {#client}

클라이언트 스팬은 나가는 HTTP 요청이나 데이터베이스 호출처럼 동기적인 아웃바운드
원격 호출을 나타낸다. 여기서 "동기(synchronous)"는 `async/await`가 아니라, 나중
처리를 위해 큐에 넣지 않는다는 뜻임에 유의한다.

#### Server {#server}

서버 스팬은 들어오는 HTTP 요청이나 원격 프로시저 호출처럼 동기적인 인바운드 원격
호출을 나타낸다.

#### Internal {#internal}

내부 스팬은 프로세스 경계를 넘지 않는 작업을 나타낸다. 함수 호출이나 Express
미들웨어를 계측할 때 내부 스팬을 주로 사용한다.

#### Producer {#producer}

프로듀서 스팬은 나중에 비동기로 처리될 수 있는 작업 생성을 나타낸다. 작업 큐에
넣는 원격 작업이거나, 이벤트 리스너가 처리하는 로컬 작업일 수 있다.

#### Consumer {#consumer}

컨슈머 스팬은 프로듀서가 만든 작업 처리를 나타내며, 프로듀서 스팬이 이미 끝난 뒤
훨씬 나중에 시작될 수 있다.

## 명세 {#specification}

자세한 내용은 [트레이스 명세](/docs/specs/otel/overview/#tracing-signal)를
참고한다.
