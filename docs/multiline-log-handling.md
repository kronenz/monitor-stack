# Multiline 로그 처리: Fluent-Bit 직접 연동 vs Data Prepper 파이프라인

## 개요

Java 애플리케이션에서 Exception이 발생하면 스택 트레이스(Stack Trace)가 여러 줄에 걸쳐 출력됩니다. 이러한 멀티라인 로그를 제대로 처리하지 않으면 각 줄이 개별 로그 엔트리로 인식되어 스택 트레이스가 분리되는 문제가 발생합니다.

### 문제 상황 예시

**원본 로그 파일:**
```
2026-01-20 13:15:00.000 ERROR c.e.SampleService - Connection failed
java.net.ConnectException: Connection refused
    at java.net.Socket.connect(Socket.java:591)
    at com.example.SampleService.connect(SampleService.java:42)
```

**잘못된 파싱 결과 (Fluent-Bit → OpenSearch 직접 연동):**
```json
// 문서 1
{"level": "ERROR", "message": "Connection failed", "@timestamp": "..."}

// 문서 2 - 스택트레이스가 별도 문서로 분리됨
{"level": "INFO", "message": "java.net.ConnectException: Connection refused", "@timestamp": "..."}

// 문서 3
{"level": "INFO", "message": "    at java.net.Socket.connect(Socket.java:591)", "@timestamp": "..."}
```

**올바른 파싱 결과 (현재 구성):**
```json
// 단일 문서로 통합
{
  "level": "ERROR",
  "logger": "c.e.SampleService",
  "message": "Connection failed\njava.net.ConnectException: Connection refused\n    at java.net.Socket.connect(Socket.java:591)",
  "@timestamp": "2026-01-20T13:15:00.139Z"
}
```

---

## 아키텍처 비교

### 1. 기존 구성: Fluent-Bit → OpenSearch (직접 연동)

```
┌─────────────────┐      ┌─────────────────┐
│   Log Files     │ ──▶  │   Fluent-Bit    │ ──▶  OpenSearch
│  (hostpath)     │      │  (line-by-line) │
└─────────────────┘      └─────────────────┘
```

**문제점:**
- Fluent-Bit의 기본 tail 입력은 **한 줄씩** 읽어서 처리
- 스택 트레이스의 continuation 라인(공백/탭으로 시작)이 별도 로그로 인식
- timestamp/level이 없는 줄은 파싱 실패 또는 기본값(INFO)으로 처리

### 2. 현재 구성: Fluent-Bit → Data Prepper → OpenSearch

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   Log Files     │ ──▶  │   Fluent-Bit    │ ──▶  │  Data Prepper   │ ──▶  OpenSearch
│  (hostpath)     │      │  (multiline)    │      │  (processing)   │
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

**해결 방법:**
- Fluent-Bit에서 **Multiline Parser**로 여러 줄을 하나의 레코드로 병합
- Data Prepper에서 추가 처리 및 필드 추출

---

## 핵심 설정 비교

### Fluent-Bit 설정

#### 기존 (문제 발생)
```ini
[INPUT]
    Name              tail
    Path              /var/log/sample-app/*.log
    Tag               java.*
    # Multiline 설정 없음 - 한 줄씩 개별 처리
```

#### 현재 (정상 동작)
```ini
[INPUT]
    Name              tail
    Path              /var/log/sample-app/*.log
    Tag               java.*
    Multiline         On
    Parser_Firstline  java-multiline    # 첫 줄 패턴 지정

[PARSER]
    Name              java-multiline
    Format            regex
    # 타임스탬프로 시작하는 줄만 새로운 로그의 시작으로 인식
    Regex             ^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3}) (?<level>\w+)\s+(?<logger>\S+) - (?<message>.*)$
    Time_Key          time
    Time_Format       %Y-%m-%d %H:%M:%S.%L
```

### Multiline 동작 원리

```
입력 로그 파일:
┌────────────────────────────────────────────────────────────────┐
│ 2026-01-20 13:15:00.000 ERROR c.e.SampleService - Connection   │ ◀── Parser 매칭 ✓ (새 레코드 시작)
│ java.net.ConnectException: Connection refused                   │ ◀── Parser 불일치 (이전 레코드에 추가)
│     at java.net.Socket.connect(Socket.java:591)                 │ ◀── Parser 불일치 (이전 레코드에 추가)
│ 2026-01-20 13:15:03.000 INFO  c.e.SampleService - Processing   │ ◀── Parser 매칭 ✓ (새 레코드 시작)
└────────────────────────────────────────────────────────────────┘

처리 결과:
┌────────────────────────────────────────────────────────────────┐
│ Record 1: {                                                     │
│   "time": "2026-01-20 13:15:00.000",                           │
│   "level": "ERROR",                                             │
│   "message": "Connection failed\njava.net.ConnectException..." │
│ }                                                               │
├────────────────────────────────────────────────────────────────┤
│ Record 2: {                                                     │
│   "time": "2026-01-20 13:15:03.000",                           │
│   "level": "INFO",                                              │
│   "message": "Processing..."                                    │
│ }                                                               │
└────────────────────────────────────────────────────────────────┘
```

---

## Data Prepper의 역할

Data Prepper는 Fluent-Bit에서 전달받은 로그를 추가 처리합니다.

### 파이프라인 설정 (pipelines.yaml)

```yaml
java-logs-pipeline:
  workers: 2
  delay: 100
  
  source:
    http:
      port: 2022
      health_check_service: true
  
  processor:
    # 1. 타임스탬프 설정
    - date:
        from_time_received: true
        destination: "@timestamp"
    
    # 2. 파이프라인 식별 필드 추가
    - add_entries:
        entries:
          - key: pipeline
            value: java-application
  
  sink:
    - opensearch:
        hosts: ["http://opensearch.logging.svc.cluster.local:9200"]
        insecure: true
        index: "java-logs-%{+yyyy.MM.dd}"
        bulk_size: 4
```

### Data Prepper 장점

| 기능 | 설명 |
|------|------|
| **버퍼링** | 로그 배치 처리로 OpenSearch 부하 감소 |
| **변환** | Grok, Date, Add Entries 등 다양한 프로세서 |
| **라우팅** | 조건에 따라 다른 인덱스로 분기 가능 |
| **확장성** | OTEL (traces, metrics) 수집 지원 |

---

## 왜 직접 연동에서 문제가 발생하는가?

### 1. Line-by-Line 처리의 한계

Fluent-Bit의 `tail` 입력은 기본적으로 각 줄을 독립적인 로그 이벤트로 처리합니다.

```
로그 파일의 각 줄  →  개별 Fluent-Bit 레코드  →  개별 OpenSearch 문서
```

### 2. 스택 트레이스 줄의 특성

Java 스택 트레이스의 continuation 줄들은:
- 타임스탬프가 없음
- 공백이나 탭으로 시작
- 로그 레벨 정보가 없음

```
2026-01-20 13:15:00.000 ERROR c.e.SampleService - Connection failed  ← 정상 파싱
java.net.ConnectException: Connection refused                         ← 파싱 실패 (타임스탬프 없음)
    at java.net.Socket.connect(Socket.java:591)                       ← 파싱 실패 (공백 시작)
```

### 3. 파싱 실패 시 동작

| 설정 | 동작 |
|------|------|
| `Parser` 없음 | 전체 줄이 `log` 필드에 저장 |
| `Parser` 실패 | 기본값 사용 또는 레코드 드롭 |
| 레벨 기본값 | 보통 INFO로 설정되어 ERROR 스택트레이스가 INFO로 표시 |

---

## 현재 구성의 Multiline 처리 흐름

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Fluent-Bit                                      │
│                                                                              │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐      │
│  │  tail INPUT      │    │  Multiline       │    │  http OUTPUT     │      │
│  │                  │───▶│  Parser          │───▶│                  │      │
│  │  Path: /var/log/ │    │                  │    │  Port: 2022      │      │
│  │  sample-app/*.log│    │  Parser_Firstline│    │  URI: /log/ingest│      │
│  └──────────────────┘    │  : java-multiline│    └────────┬─────────┘      │
│                          └──────────────────┘             │                 │
└───────────────────────────────────────────────────────────┼─────────────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                             Data Prepper                                     │
│                                                                              │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐      │
│  │  http SOURCE     │    │  PROCESSORS      │    │ opensearch SINK  │      │
│  │                  │───▶│                  │───▶│                  │      │
│  │  Port: 2022      │    │  - date          │    │  Index:          │      │
│  │                  │    │  - add_entries   │    │  java-logs-*     │      │
│  └──────────────────┘    └──────────────────┘    └────────┬─────────┘      │
│                                                           │                 │
└───────────────────────────────────────────────────────────┼─────────────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              OpenSearch                                      │
│                                                                              │
│  Index: java-logs-2026.01.20                                                │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ {                                                                       │ │
│  │   "@timestamp": "2026-01-20T13:15:00.139Z",                            │ │
│  │   "level": "ERROR",                                                     │ │
│  │   "logger": "c.e.SampleService",                                        │ │
│  │   "message": "Connection failed\njava.net.ConnectException:...",       │ │
│  │   "pipeline": "java-application"                                        │ │
│  │ }                                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 실제 데이터 비교

### 잘못된 처리 (기존 구성)

```json
// 3개의 문서로 분리됨
{"_id": "doc1", "level": "ERROR", "message": "Connection failed"}
{"_id": "doc2", "level": "INFO",  "message": "java.net.ConnectException: Connection refused"}
{"_id": "doc3", "level": "INFO",  "message": "    at java.net.Socket.connect(Socket.java:591)"}
```

**문제점:**
- 에러 검색 시 스택트레이스 누락
- 로그 분석 어려움
- 잘못된 로그 레벨 통계

### 올바른 처리 (현재 구성)

```json
// 단일 문서
{
  "_id": "Uq6L25sBGyVA_S58wFWA",
  "level": "ERROR",
  "logger": "c.e.SampleService",
  "message": "Connection failed\njava.net.ConnectException: Connection refused\n    at java.net.Socket.connect(Socket.java:591)",
  "source": "hostpath",
  "app_type": "java",
  "@timestamp": "2026-01-20T13:15:00.139Z",
  "pipeline": "java-application"
}
```

**장점:**
- 완전한 스택트레이스 보존
- 정확한 로그 레벨
- 효율적인 에러 분석

---

## Parser_Firstline 정규식 설명

```regex
^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3}) (?<level>\w+)\s+(?<logger>\S+) - (?<message>.*)$
```

| 패턴 | 설명 | 예시 |
|------|------|------|
| `^` | 줄의 시작 | - |
| `(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3})` | 타임스탬프 캡처 | `2026-01-20 13:15:00.000` |
| `(?<level>\w+)` | 로그 레벨 캡처 | `ERROR`, `INFO` |
| `\s+` | 공백 | - |
| `(?<logger>\S+)` | 로거 이름 캡처 | `c.e.SampleService` |
| ` - ` | 구분자 | - |
| `(?<message>.*)` | 메시지 캡처 | `Connection failed` |
| `$` | 줄의 끝 | - |

**핵심 포인트:** 이 정규식과 매칭되지 않는 줄은 이전 레코드의 `message`에 추가됩니다.

---

## 다른 Multiline 패턴 예시

### Log4j/Logback 패턴
```ini
[PARSER]
    Name        java-log4j
    Format      regex
    Regex       ^(?<time>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}[.,]\d{3}) \[(?<thread>[^\]]+)\] (?<level>\w+)\s+(?<logger>\S+) - (?<message>.*)$
```

### Spring Boot 패턴
```ini
[PARSER]
    Name        spring-boot
    Format      regex
    Regex       ^(?<time>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}.\d{3})\s+(?<level>\w+)\s+\d+\s+---\s+\[(?<thread>[^\]]+)\]\s+(?<logger>\S+)\s+:\s+(?<message>.*)$
```

### Python Traceback 패턴
```ini
[PARSER]
    Name        python-traceback
    Format      regex
    Regex       ^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3}) - (?<logger>\S+) - (?<level>\w+) - (?<message>.*)$
```

---

## 결론

| 항목 | Fluent-Bit 직접 연동 | Data Prepper 파이프라인 |
|------|---------------------|------------------------|
| **Multiline 처리** | 별도 설정 필요 (누락 시 분리) | Fluent-Bit에서 처리 후 전달 |
| **스택트레이스** | 줄 단위로 분리됨 | 단일 메시지로 통합 |
| **로그 레벨 정확도** | 스택트레이스가 INFO로 오인 | 정확한 레벨 유지 |
| **확장성** | 제한적 | OTEL, 다중 파이프라인 지원 |
| **버퍼링** | Fluent-Bit 자체 버퍼 | Data Prepper 배치 처리 |

**핵심 차이점:** Fluent-Bit의 `Multiline` 옵션과 `Parser_Firstline` 설정을 통해 여러 줄의 로그를 하나의 레코드로 병합하여 처리합니다. 이 설정이 없으면 각 줄이 개별 문서로 저장되어 스택트레이스가 분리되는 문제가 발생합니다.
