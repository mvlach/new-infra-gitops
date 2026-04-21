# Spring Boot → ClickHouse (structured logging)

Short recipe for producing JSON logs that Vector will fan out into the
`logs.pod_logs` ClickHouse table with structured columns (`level`, `logger`,
`thread`, `trace_id`, `span_id`, `stack_trace`).

## 1. Dependency

```xml
<!-- pom.xml -->
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>8.0</version>
</dependency>
```

For Gradle:

```kotlin
implementation("net.logstash.logback:logstash-logback-encoder:8.0")
```

## 2. `src/main/resources/logback-spring.xml`

Emits one JSON document per line to stdout. Spring Boot auto-detects this
file and replaces the default pattern encoder.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <!-- Ensure the fields match Vector's VRL extraction -->
      <fieldNames>
        <timestamp>@timestamp</timestamp>
        <message>message</message>
        <logger>logger_name</logger>
        <thread>thread_name</thread>
        <level>level</level>
        <stackTrace>stack_trace</stackTrace>
      </fieldNames>
      <!-- Promote Micrometer Tracing MDC entries into top-level fields so
           Vector can extract them directly. -->
      <includeMdcKeyName>traceId</includeMdcKeyName>
      <includeMdcKeyName>spanId</includeMdcKeyName>
      <!-- Pretty-print = false, one line per event (Vector reads per-line). -->
      <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
        <maxDepthPerThrowable>30</maxDepthPerThrowable>
        <maxLength>4096</maxLength>
        <rootCauseFirst>true</rootCauseFirst>
      </throwableConverter>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="JSON"/>
  </root>
</configuration>
```

## 3. Tracing (optional)

If you use Spring Boot 3.x observability (Micrometer Tracing + Brave/OTel),
the MDC already carries `traceId` and `spanId`. They will land in the
`trace_id` / `span_id` ClickHouse columns without any extra code.

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
  <groupId>io.zipkin.reporter2</groupId>
  <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

`application.yml`:
```yaml
management:
  tracing:
    sampling:
      probability: 1.0
```

## 4. Verify

```bash
# from laptop on VPN
curl -s "http://10.0.1.10:8123/?query=
SELECT timestamp, pod, level, logger, trace_id, message
FROM logs.pod_logs
WHERE level != ''
ORDER BY timestamp DESC
LIMIT 20
FORMAT PrettyCompactMonoBlock"
```

Or in Grafana → Explore → ClickHouse SQL editor:

```sql
SELECT timestamp, level, logger, message, stack_trace
FROM logs.pod_logs
WHERE namespace = '<your-ns>'
  AND level IN ('ERROR','WARN')
ORDER BY timestamp DESC
LIMIT 100
```

## Columns available

| Column      | Source                                         |
|-------------|------------------------------------------------|
| timestamp   | k8s log event time                             |
| namespace   | pod namespace                                  |
| pod         | pod name                                       |
| container   | container name                                 |
| node        | k8s node                                       |
| stream      | `stdout` / `stderr`                            |
| message     | JSON `message` (fallback: raw line)            |
| level       | JSON `level`                                   |
| logger      | JSON `logger_name` or `logger`                 |
| thread      | JSON `thread_name` or `thread`                 |
| trace_id    | JSON `traceId` or `trace_id` (from MDC)        |
| span_id    | JSON `spanId`  or `span_id`                    |
| stack_trace | JSON `stack_trace` (full trace incl. causes)   |
| labels      | `Map` of pod labels                            |

Logs from non-JSON workloads (k3s core pods, third-party Helm charts with
plain text output, etc.) still land in `message`; structured columns are
empty strings for those rows.
