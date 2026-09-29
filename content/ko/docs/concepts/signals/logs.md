---
title: 로그
description: 이벤트 기록
weight: 3
default_lang_commit: c161165987d527c1efd6bc969d7fef905946c561
cSpell:ignore: filelogreceiver semistructured transformprocessor
---

**로그(log)** 는 타임스탬프(timestamp)가 포함된 텍스트 기록으로, 구조화(권장)
또는 비구조화 형태일 수 있으며 메타데이터를 선택적으로 담을 수 있다. 텔레메트리
시그널 중 로그는 가장 오랜 역사를 지니고 있다. 대부분의 프로그래밍 언어에는 내장
로깅 기능이 있거나 널리 쓰이는 로깅 라이브러리가 있다.

## 오픈텔레메트리 로그 {#opentelemetry-logs}

오픈텔레메트리(OpenTelemetry)는 로그 레코드를 만들 Logs API와 SDK, 기존 로깅
프레임워크와 통합하는 언어 SDK·로깅 브리지를 제공한다. 로그는 로거
프로바이더(logger provider)를 통해 보내는 모든 데이터이며, 이벤트(event)는
로그의 특별한 유형이다. 모든 로그가 이벤트는 아니지만, 모든 이벤트는 로그다.
Logs API는 공개되어 있어 애플리케이션 코드에서 직접 쓰거나, 기존 로깅
라이브러리·브리지를 통해 간접적으로 쓸 수 있다.

오픈텔레메트리는 이미 생성하던 로그와 함께 동작하도록 설계되었다. 로그를 다른
시그널과 연관하고, 맥락 속성을 추가하며, 서로 다른 소스를
처리·내보내기(export)에 적합한 공통 표현으로 정규화하는 도구를 제공한다.

### 오픈텔레메트리 컬렉터의 로그 {#opentelemetry-logs-in-the-opentelemetry-collector}

[오픈텔레메트리 컬렉터](/docs/collector/)는 로그를 다루는 여러 도구를 제공한다.

- 알려진 특정 로그 데이터 소스에서 로그를 파싱하는 여러 리시버(receiver)
- 어떤 파일에서든 로그를 읽고, 형식별 파싱이나 정규식 파싱 기능을 제공하는
  `filelogreceiver`
- 중첩 데이터 파싱, 중첩 구조 평탄화, 값 추가·삭제·갱신 등을 할 수 있는
  `transformprocessor` 같은 프로세서(processor)
- 오픈텔레메트리가 아닌 형식으로 로그 데이터를 내보내는 익스포터(exporter)

오픈텔레메트리를 도입할 때 흔히 첫 단계는 컬렉터를 범용 로깅 에이전트로 배포하는
것이다.

### 애플리케이션의 로그 {#opentelemetry-logs-for-applications}

애플리케이션에서는 어떤 로깅 라이브러리나 내장 로깅 기능으로도 오픈텔레메트리
로그를 만들 수 있다. 자동 계측(autoinstrumentation)을 추가하거나 SDK를
활성화하면, 오픈텔레메트리가 기존 로그를 활성 트레이스·스팬과 자동으로 연관하고,
로그 레코드에 해당 ID를 포함한다. 즉, 오픈텔레메트리가 로그와 트레이스를
자동으로 연관한다.

### 언어 지원 {#language-support}

로그는 오픈텔레메트리 명세에서
[stable](/docs/specs/otel/versioning-and-stability/#stable) 등급 시그널이다.
Logs API와 SDK의 언어별 구현 상태는 다음과 같다.

{{% signal-support-table "logs" %}}

## 구조화·비구조화·반구조화 로그 {#structured-unstructured-and-semistructured-logs}

오픈텔레메트리는 어떤 로그 형식이든 받아들이지만, 분석에 유용한 정도는 형식마다
다르다. 다음 절은 구조화·반구조화·비구조화 로그의 차이를 설명한다. 중요:
JSON으로 인코딩했다고 해서 안정적인 스키마를 갖는 **구조화** 로그가 되는 것은
아니며, **반구조화**일 수 있다. 구조화 로그는 유효한 JSON인지 여부가 아니라,
다운스트림(downstream) 처리가 안정적으로 의존할 수 있는 일관된 스키마나 잘
정의된 타입 필드를 의미한다.

### 구조화 로그 {#structured-logs}

구조화 로그는 정의되고 일관된 스키마나 타입이 정해진 필드를 갖춘 로그로,
다운스트림(downstream) 시스템이 안정적으로 파싱·해석할 수 있다. 텍스트 인코딩은
JSON, protobuf 등이 될 수 있지만, 구조화 여부는 유효한 JSON인지가 아니라
안정적인 스키마(필드 이름, 타입, 의미)가 있는지에 달려 있다. 구조화 JSON 로그
예는 다음과 같다.

```json
{
  "timestamp": "2024-08-04T12:34:56.789Z",
  "level": "INFO",
  "service": "user-authentication",
  "environment": "production",
  "message": "User login successful",
  "context": {
    "userId": "12345",
    "username": "johndoe",
    "ipAddress": "192.168.1.1",
    "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36"
  },
  "transactionId": "abcd-efgh-ijkl-mnop",
  "duration": 200,
  "request": {
    "method": "POST",
    "url": "/api/v1/login",
    "headers": {
      "Content-Type": "application/json",
      "Accept": "application/json"
    },
    "body": {
      "username": "johndoe",
      "password": "******"
    }
  },
  "response": {
    "statusCode": 200,
    "body": {
      "success": true,
      "token": "jwt-token-here"
    }
  }
}
```

인프라 구성 요소에서는 Common Log Format(CLF)을 흔히 쓴다.

```text
127.0.0.1 - johndoe [04/Aug/2024:12:34:56 -0400] "POST /api/v1/login HTTP/1.1" 200 1234
```

CLF 필드 뒤에 JSON blob이 이어지는 하이브리드·확장 형식도 흔하다.

```text
192.168.1.1 - johndoe [04/Aug/2024:12:34:56 -0400] "POST /api/v1/login HTTP/1.1" 200 1234 "http://example.com" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36" {"transactionId": "abcd-efgh-ijkl-mnop", "responseTime": 150, "requestBody": {"username": "johndoe"}, "responseHeaders": {"Content-Type": "application/json"}}
```

이런 경우 필요한 부분을 파싱하거나 추출해 정규화된 레코드로 만들면
다운스트림(downstream) 도구가 일관되게 분석할 수 있다.
[오픈텔레메트리 컬렉터](/docs/collector/)의 `filelogreceiver`는 혼합 형식 파싱을
돕는다.

프로덕션에서는 안정적인 스키마 덕분에 검증·파싱·트레이스·메트릭과의 연관·대규모
분석이 수월해 **구조화 로그**를 선호한다.

### 비구조화 로그 {#unstructured-logs}

비구조화 로그는 일관된 구조를 따르지 않는다. 사람이 읽기 쉬워 개발 환경에서 자주
쓰이지만, 프로덕션 옵저버빌리티에는 비구조화 로그를 쓰는 것을 권장하지 않는다.
대규모로 파싱·분석하기가 훨씬 어렵기 때문이다.

비구조화 로그 예:

```text
[ERROR] 2024-08-04 12:45:23 - Failed to connect to database. Exception: java.sql.SQLException: Timeout expired. Attempted reconnect 3 times. Server: db.example.com, Port: 5432

System reboot initiated at 2024-08-04 03:00:00 by user: admin. Reason: Scheduled maintenance. Services stopped: web-server, database, cache. Estimated downtime: 15 minutes.

DEBUG - 2024-08-04 09:30:15 - User johndoe performed action: file_upload. Filename: report_Q3_2024.pdf, Size: 2.3 MB, Duration: 5.2 seconds. Result: Success
```

프로덕션에서도 비구조화 로그를 저장·분석할 수는 있지만, 기계가 읽을 수 있게
파싱하거나 전처리하는 작업이 상당히 필요할 수 있다. 예를 들어 위 세 로그는
타임스탬프를 뽑으려면 정규식이 필요하고, 로그 메시지 본문을 일관되게 추출하려면
맞춤 파서가 필요하다. 로깅 백엔드가 타임스탬프로 정렬·구성하려면 보통 이런
처리가 필요하다. 분석 목적으로 비구조화 로그를 파싱하는 것도 가능하지만,
애플리케이션에서 표준 로깅 프레임워크로 **구조화 로깅**으로 바꾸는 편이 작업량을
줄이는 방법일 수 있다.

### 반구조화 로그 {#semistructured-logs}

반구조화(semistructured) 로그는 기계가 읽을 수 있는 key=value 쌍이나 구분 필드를
포함하지만, 로그를 내보내는 쪽마다 안정적인 스키마를 보장하지는 않는다. 아래
key=value 로깅이나, 메시지마다 필드 이름·타입이 달라지는 JSON blob이 예다.
반구조화 로그는 비구조화보다 파싱하기 쉬운 경우가 많지만, 분석 전 처리·정규화가
여전히 필요할 수 있다.

반구조화 로그 예:

```text
2024-08-04T12:45:23Z level=ERROR service=user-authentication userId=12345 action=login message="Failed login attempt" error="Invalid password" ipAddress=192.168.1.1 userAgent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/104.0.0.0 Safari/537.36"
```

반구조화 로그는 다운스트림(downstream) 분석에 충분히 쓰이려면 수집(ingestion)
단계에서 매핑·타입 강제 변환(type coercion)이 필요할 수 있다.

## 로깅 활성 여부 확인 {#checking-whether-logging-is-enabled}

로깅 호출 전에 로깅이 활성화됐는지 확인해야 하는지 자주 묻는다. 예:

```text
if (logger.Enabled(...)) {
  logger.Info("Hello {name}", name);
}
```

대부분 이 확인은 불필요하며 권장하지 않는다. 오픈텔레메트리 SDK는 효율적으로
설계되어, 로거가 비활성일 때 logging API 호출 오버헤드는 최소다.
`logger.Enabled`를 한 번 더 호출하면 성능이 떨어지고 코드만 복잡해진다.

`Enabled` API는 로깅 호출에 넘기는 **인자를 평가하는 것** 자체가 비용이 클 때만
유용하다. 로거가 비활성일 때 그 비용을 피하고 싶을 때다. 예를 들어 본문이나
속성을 DB에서 가져오거나 값비싼 연산으로 계산해야 할 때:

```text
if (logger.Enabled(...)) {
  logger.Info("Order total {total}", ComputeExpensiveTotal());
}
```

부수 효과가 없는 식만 가드(guard) 처리해야 한다. 가드(guard) 처리된 코드는
로깅이 활성일 때만 실행되기 때문이다. 부수 효과가 있거나 다른 로직이 의존하는
코드를 가드(guard) 처리하면 애플리케이션 동작이 로깅 설정에 따라 달라져, 미묘한
버그의 원인이 되기 쉽다.

`Enabled` 결과는 정적이 아니다. 설정이 바뀌면 시간에 따라 달라질 수 있으므로,
로그 레코드마다 평가하고 캐시해서는 안 된다.

규범 API 안내는 [Logs API 명세](/docs/specs/otel/logs/api/#enabled)를 참고한다.

## 오픈텔레메트리 로깅 구성 요소 {#opentelemetry-logging-components}

오픈텔레메트리 로깅 지원을 이루는 개념·구성 요소는 다음과 같다.

### Log Appender / Bridge {#log-appender--bridge}

애플리케이션 개발자는 **Logs Bridge API**를 직접 호출하지 않는 것이 좋다. 로깅
라이브러리 작성자가 log appender / bridge를 만들도록 제공된 API이기 때문이다.
선호하는 로깅 라이브러리를 쓰고, 오픈텔레메트리 `LogRecordExporter`로 로그를
내보낼 수 있는 log appender(또는 log bridge)를 쓰도록 설정하면 된다.

오픈텔레메트리 언어 SDK가 이 기능을 제공한다.

### 로거 프로바이더 {#logger-provider}

> **Logs Bridge API**의 일부이며, 로깅 라이브러리 작성자만 사용한다.

로거 프로바이더(때로 `LoggerProvider`라고 부름)는 `Logger`를 만드는
팩토리(factory)다. 대부분 로거 프로바이더는 한 번 초기화하며, 수명 주기는
애플리케이션과 같다. 로거 프로바이더 초기화에는 리소스·익스포터 초기화도
포함된다.

### 로거 {#logger}

> **Logs Bridge API**의 일부이며, 로깅 라이브러리 작성자만 사용한다.

로거(logger)는 로그 레코드를 만든다. 로거는 로거 프로바이더에서 만든다.

### 로그 레코드 익스포터 {#log-record-exporter}

로그 레코드 익스포터는 로그 레코드를 소비자(consumer)에게 보낸다. 소비자는
디버깅·개발용 표준 출력, 오픈텔레메트리 컬렉터, 또는 선택한 오픈소스·벤더
백엔드일 수 있다.

### 로그 레코드 {#log-record}

로그 레코드(log record)는 이벤트 기록을 나타낸다. 오픈텔레메트리에서 로그 레코드
필드는 두 종류다.

- 특정 타입·의미를 갖는 이름 있는 최상위 필드
- 임의 값·타입의 리소스·속성(attribute) 필드

최상위 필드는 다음과 같다.

| 필드 이름            | 설명                                             |
| -------------------- | ------------------------------------------------ |
| Timestamp            | 이벤트가 발생한 시각                             |
| ObservedTimestamp    | 이벤트가 관측된 시각                             |
| TraceId              | 요청 trace ID                                    |
| SpanId               | 요청 span ID                                     |
| TraceFlags           | W3C trace flag                                   |
| SeverityText         | 심각도 텍스트(로그 레벨이라고도 함)              |
| SeverityNumber       | 심각도(severity)의 숫자 값                       |
| Body                 | 로그 레코드 본문                                 |
| Resource             | 로그 출처                                        |
| InstrumentationScope | 로그를 발생시킨 계측 범위(instrumentation scope) |
| Attributes           | 이벤트에 대한 추가 정보                          |
| EventName            | 이벤트 종류·유형을 식별하는 이름                 |

로그 레코드·필드 자세한 내용은
[로그 데이터 모델](/docs/specs/otel/logs/data-model/)을 참고한다.

### 명세 {#specification}

자세한 내용은 [로그 명세][logs specification]를 참고한다.

[logs specification]: /docs/specs/otel/overview/#log-signal
