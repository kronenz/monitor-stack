# HostPath 로그 수집 가이드

이 문서는 Java 애플리케이션에서 HostPath를 통해 로그를 수집하는 방법을 설명합니다.

## 개요

Fluent Bit이 호스트의 `/var/log/{namespace}/` 디렉토리에서 로그 파일을 수집합니다.
애플리케이션은 지정된 경로와 파일명 패턴에 맞게 로그를 출력해야 합니다.

## 로그 파일 경로 규칙

```
/var/log/{namespace}/{app}-{service}-{jobid}-YYYY-MM-dd.log
```

| 필드 | 설명 | 예시 |
|------|------|------|
| `{namespace}` | Kubernetes 네임스페이스 | `production`, `staging` |
| `{app}` | 애플리케이션 이름 | `myapp`, `order-service` |
| `{service}` | 서비스/컴포넌트 이름 | `api`, `worker`, `batch` |
| `{jobid}` | Pod 이름 또는 Job ID | `pod-abc123`, `job-001` |
| `YYYY-MM-dd` | 날짜 (일별 로테이션) | `2024-01-15` |

### 예시 경로

```
/var/log/production/myapp-api-pod-abc123-2024-01-15.log
/var/log/staging/order-service-worker-job-001-2024-01-15.log
```

## 애플리케이션 설정

### 1. Deployment에 HostPath 볼륨 마운트

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-java-app
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: app
          image: my-java-app:latest
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: LOG_PATH
              value: "/var/log/app"
      volumes:
        - name: app-logs
          hostPath:
            path: /var/log/production
            type: DirectoryOrCreate
```

### 2. Java 로그 설정 (Logback 예시)

`logback-spring.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="APP_NAME" source="spring.application.name" defaultValue="myapp"/>
    <springProperty scope="context" name="SERVICE_NAME" source="app.service.name" defaultValue="api"/>
    
    <property name="POD_NAME" value="${POD_NAME:-unknown}"/>
    <property name="LOG_PATH" value="${LOG_PATH:-/var/log/app}"/>
    
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}-${SERVICE_NAME}-${POD_NAME}-${LOG_DATE}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-${SERVICE_NAME}-${POD_NAME}-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

### 3. Log4j2 설정 예시

`log4j2.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Properties>
        <Property name="APP_NAME">${env:APP_NAME:-myapp}</Property>
        <Property name="SERVICE_NAME">${env:SERVICE_NAME:-api}</Property>
        <Property name="POD_NAME">${env:POD_NAME:-unknown}</Property>
        <Property name="LOG_PATH">${env:LOG_PATH:-/var/log/app}</Property>
        <Property name="LOG_PATTERN">%d{yyyy-MM-dd HH:mm:ss.SSS} %5level %logger{36} - %msg%n</Property>
    </Properties>
    
    <Appenders>
        <RollingFile name="FileAppender"
                     fileName="${LOG_PATH}/${APP_NAME}-${SERVICE_NAME}-${POD_NAME}.log"
                     filePattern="${LOG_PATH}/${APP_NAME}-${SERVICE_NAME}-${POD_NAME}-%d{yyyy-MM-dd}.log">
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
            </Policies>
            <DefaultRolloverStrategy max="7"/>
        </RollingFile>
    </Appenders>
    
    <Loggers>
        <Root level="INFO">
            <AppenderRef ref="FileAppender"/>
        </Root>
    </Loggers>
</Configuration>
```

## 로그 포맷 요구사항

Fluent Bit이 로그를 올바르게 파싱하려면 다음 형식을 준수해야 합니다:

```
YYYY-MM-dd HH:mm:ss.SSS LEVEL logger - message
```

### 예시

```
2024-01-15 10:30:00.123 INFO  c.e.MyService - Application started
2024-01-15 10:30:01.456 ERROR c.e.MyService - Connection failed
java.net.ConnectException: Connection refused
    at java.net.Socket.connect(Socket.java:591)
    at com.example.MyService.connect(MyService.java:42)
```

> **중요**: 스택트레이스는 타임스탬프로 시작하지 않으므로 이전 로그 메시지에 자동으로 병합됩니다.

## 수집된 로그 확인

### OpenSearch Dashboards에서 확인

1. OpenSearch Dashboards 접속: `http://{NODE_IP}:30601`
2. Index Pattern 생성: `java-logs-*`
3. Discover 탭에서 로그 검색

### 로그에 추가되는 메타데이터

| 필드 | 설명 |
|------|------|
| `namespace` | 추출된 네임스페이스 |
| `app` | 추출된 애플리케이션 이름 |
| `service` | 추출된 서비스 이름 |
| `jobid` | 추출된 Job/Pod ID |
| `level` | 로그 레벨 (INFO, ERROR 등) |
| `logger` | 로거 이름 |
| `log_message` | 파싱된 로그 메시지 |
| `filepath` | 원본 파일 경로 |
| `@timestamp` | 로그 수집 시간 |

## 트러블슈팅

### 로그가 수집되지 않는 경우

1. **디렉토리 권한 확인**
   ```bash
   ls -la /var/log/{namespace}/
   ```

2. **Fluent Bit Pod 로그 확인**
   ```bash
   kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit
   ```

3. **파일명 패턴 확인**
   - 파일명이 `{app}-{service}-{jobid}-YYYY-MM-dd.log` 형식인지 확인
   - `.log` 확장자 필수

### 멀티라인 로그가 분리되는 경우

- 로그 타임스탬프 형식이 `YYYY-MM-dd HH:mm:ss.SSS`인지 확인
- 스택트레이스가 타임스탬프로 시작하지 않는지 확인

## 참고 자료

- [Fluent Operator CRDs](https://github.com/fluent/fluent-operator)
- [Data Prepper Documentation](https://opensearch.org/docs/latest/data-prepper/)
- [OpenSearch Dashboards](https://opensearch.org/docs/latest/dashboards/)
