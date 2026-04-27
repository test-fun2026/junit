# Micrometer Tracing Migration (STRICT – Replace MDC Fully)

## 🎯 Objective
- Remove MDC completely
- Use Micrometer Tracing + Baggage
- Propagate requestId across:
  - HTTP
  - Kafka
  - MQ (JMS)
  - Async / Virtual Threads

---

# 1. pom.xml (STRICT – Only Required)

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing</artifactId>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
```

---

# 2. application.yml

```yaml
management:
  tracing:
    enabled: true
    baggage:
      remote-fields:
        - X-Request-Id
      correlation:
        fields:
          - X-Request-Id
```

---

# 3. EXACT CODE CHANGES (VERY IMPORTANT)

## 3.1 CoreIntegrationController.java ❌ REMOVE MDC

### REMOVE
```java
MDC.putCloseable("X-Request-Id", req.getHeader().getRequestId());
MDC.putCloseable("UETR", ...);
```

### DO NOTHING ELSE HERE
👉 Baggage will be set in ValidationAspect

---

## 3.2 ValidationAspect.java ✅ MAIN CHANGE (CRITICAL)

### ADD
```java
@Autowired
private Tracer tracer;
```

### MODIFY METHOD
```java
@Before("@annotation(validateRequest)")
public void before(JoinPoint joinPoint, ValidateRequest validateRequest) {

    if (!validateRequest.enabled()) return;

    if (joinPoint.getArgs().length > 0 &&
        joinPoint.getArgs()[0] instanceof CoreIntegrationRequest req) {

        // ✅ Extract requestId
        String requestId = req.getHeader().getRequestId();

        // ✅ SET BAGGAGE (REPLACES MDC)
        tracer.createBaggage("X-Request-Id").set(requestId);

        // existing validation
        requestValidationService.validate(req);
    }
}
```

---

## 3.3 CoreIntegrationExceptionHandler.java ❌ REMOVE MDC

### REMOVE ALL
```java
MDC.put(...)
```

👉 No replacement needed

---

## 3.4 FraudCheckExecutor.java ❌ NO CHANGE

✔ Already safe (no MDC)

---

## 3.5 OrchestrationService.java ❌ NO CHANGE

✔ Works automatically via tracing

---

## 3.6 ThreadPoolConfig.java ❌ NO CHANGE

✔ Micrometer supports virtual threads automatically

---

## 3.7 SdkExecutionTimeAspect.java ❌ NO CHANGE

✔ Logging will automatically include baggage

---

# 4. LOGGING (IMPORTANT)

```yaml
logging:
  pattern:
    level: "%5p [traceId=%X{traceId},spanId=%X{spanId},X-Request-Id=%X{X-Request-Id}]"
```

---

# 5. HTTP PROPAGATION (AUTO)

## RestTemplate
```java
@Bean
RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder.build();
}
```

## WebClient
```java
@Bean
WebClient webClient(WebClient.Builder builder) {
    return builder.build();
}
```

---

# 6. KAFKA PROPAGATION

## Producer
```java
@Autowired
Tracer tracer;

ProducerRecord<String, String> record = new ProducerRecord<>("topic", message);

tracer.currentSpan().context().inject(record.headers(), (h, k, v) -> {
    h.add(k, v.getBytes());
});

kafkaTemplate.send(record);
```

## Consumer
```java
@KafkaListener(topics = "topic")
public void consume(ConsumerRecord<String, String> record) {

    String requestId =
        new String(record.headers().lastHeader("X-Request-Id").value());

    tracer.createBaggage("X-Request-Id").set(requestId);
}
```

---

# 7. MQ (JMS) PROPAGATION

## Producer
```java
jmsTemplate.convertAndSend("queue", message, msg -> {
    tracer.currentSpan().context().inject(msg, (m, k, v) -> {
        m.setStringProperty(k, v);
    });
    return msg;
});
```

## Consumer
```java
@JmsListener(destination = "queue")
public void receive(Message message) throws Exception {

    String requestId = message.getStringProperty("X-Request-Id");

    tracer.createBaggage("X-Request-Id").set(requestId);
}
```
“How do you maintain tracing when message goes to DLQ?”
dlqRecord.headers().add(original.headers());

Common Failure
DLQ publisher creates new message without headers
DLQ message → NEW traceId ❌

----
“What happens to tracing during retries?”
✅ Micrometer Behavior
Same traceId
New span per retry

----
“What happens during Kafka rebalance?”
🚨 Problem
Partition reassigned
Another consumer picks message
✅ Micrometer Behavior

✔ Safe because context is in message headers
-----
“What if consumer processing is slow?”

🚨 Problem
Kafka timeout
Consumer considered dead
Rebalance triggered
❗ Impact
Same message processed again

✅ Micrometer Role
Helps detect latency via spans
Shows slow processing
------

“What happens if producer fails?”

🚨 Problem
Message not sent
Retry triggered
❗ Risk
Duplicate send
✅ Micrometer Behavior
Same traceId across retries
----
“How does tracing behave across multiple Kafka hops?”

✅ Behavior
traceId = SAME
spanId = different per hop
requestId = SAME
❗ Failure Case
One hop breaks header propagation

----
“What happens in batch processing?”

🚨 Problem
Batch → multiple requestIds
❗ Risk
One baggage reused for all messages
------

---

# 🚨 FINAL RULES

- ❌ MDC completely removed
- ❌ No ThreadLocal usage
- ✅ Baggage ONLY
- ✅ ValidationAspect = single source of requestId
- ✅ Auto propagation across threads, Kafka, MQ, HTTP

---

# ✅ FINAL FLOW

Request → ValidationAspect (set baggage)
       → Service → Executor → Kafka/MQ/HTTP
       → Downstream services receive SAME requestId

