# junit

@ExtendWith(MockitoExtension.class)
class FraudCheckConsumerAdditionalTest {

    @Mock
    private FraudCheckExecutor executor;

    @Mock
    private FraudValidator validator;

    @Mock
    private KafkaTemplate<String, Object> producer;

    @Mock
    private Acknowledgment ack;

    private FraudCheckConsumer consumer;

    private ObjectMapper objectMapper = new ObjectMapper();

    @BeforeEach
    void setup() throws Exception {
        consumer = new FraudCheckConsumer(executor, validator, producer);

        setPrivateField(consumer, "messageFormat", "json");
        setPrivateField(consumer, "successTopic", "success-topic");
        setPrivateField(consumer, "failureTopic", "failure-topic");
    }

    @Test
    void shouldPublishSuccessMessage() throws Exception {

        CoreIntegrationRequest request = buildRequest("FraudCheck", "RID-100");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 0L, "key1", json);

        OrchestratedRequest orchestrated = mock(OrchestratedRequest.class);

        when(executor.orchestrate(any())).thenReturn(orchestrated);

        try (MockedStatic<RequestBuilder> builder =
                     mockStatic(RequestBuilder.class)) {

            builder.when(() -> RequestBuilder.prepareRequest(any()))
                    .thenReturn(orchestrated);

            consumer.listen(rec, ack);

            verify(producer).send(any());
            verify(ack).acknowledge();
        }
    }

    @Test
    void shouldHandleValidationException() throws Exception {

        CoreIntegrationRequest request = buildRequest("FraudCheck", "RID-101");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 0L, "key1", json);

        doThrow(new ValidationException("validation failed"))
                .when(validator).validate(any());

        consumer.listen(rec, ack);

        verify(producer).send(any());
        verify(ack).acknowledge();
    }

    @Test
    void shouldHandleGenericException() throws Exception {

        CoreIntegrationRequest request = buildRequest("FraudCheck", "RID-102");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 0L, "key1", json);

        doThrow(new RuntimeException("unknown error"))
                .when(validator).validate(any());

        consumer.listen(rec, ack);

        verify(producer).send(any());
        verify(ack).acknowledge();
    }

    @Test
    void shouldThrowRetryableExceptionWhenTimeout() throws Exception {

        CoreIntegrationRequest request = buildRequest("FraudCheck", "RID-103");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 0L, "key1", json);

        SdkException ex = mock(SdkException.class);

        when(ex.getErrorCode()).thenReturn("TITAN_SERVICE_TIMEOUT");

        doThrow(ex).when(validator).validate(any());

        assertThrows(RetryableProcessingException.class,
                () -> consumer.listen(rec, ack));
    }

    @Test
    void shouldSkipResponseForSkipType() throws Exception {

        CoreIntegrationRequest request = buildRequest("Skip", "RID-104");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 0L, "key1", json);

        consumer.listen(rec, ack);

        verify(ack).acknowledge();
        verifyNoInteractions(producer);
    }

    private CoreIntegrationRequest buildRequest(String operation, String requestId) {

        Header header = new Header();
        header.setOperation(operation);
        header.setRequestId(requestId);

        CoreIntegrationRequest req = new CoreIntegrationRequest();
        req.setHeader(header);

        return req;
    }

    private static void setPrivateField(Object target, String fieldName, Object value)
            throws Exception {

        Field field = target.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(target, value);
    }
}
------------------------------------------------

@ExtendWith(MockitoExtension.class)
@DisplayName("FraudCheckConsumer Unit Tests - 100% Coverage")
class FraudCheckConsumerTest {

    private ObjectMapper objectMapper;

    @Mock
    private FraudCheckExecutor executor;

    @Mock
    private FraudValidator validator;

    @Mock
    private KafkaTemplate<String, Object> producer;

    @Mock
    private Acknowledgment ack;

    private FraudCheckConsumer consumer;

    @BeforeEach
    void setup() throws Exception {

        objectMapper = new ObjectMapper();

        consumer = new FraudCheckConsumer(executor, validator, producer);

        setPrivateField(consumer, "messageFormat", "json");
        setPrivateField(consumer, "successTopic", "fraud-success-topic");
        setPrivateField(consumer, "failureTopic", "fraud-failure-topic");
    }

    // ------------------------------
    // SUCCESS FLOW
    // ------------------------------

    @Test
    @DisplayName("Should process valid message and acknowledge")
    void shouldProcessValidMessage() throws Exception {

        CoreIntegrationRequest request = buildRequest("FraudCheck", "RID-1");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 1L, "key1", json);

        OrchestratedRequest orchestrated = mock(OrchestratedRequest.class);

        when(executor.orchestrate(any())).thenReturn(orchestrated);

        try (MockedStatic<RequestBuilder> builder =
                     mockStatic(RequestBuilder.class)) {

            builder.when(() -> RequestBuilder.prepareRequest(any()))
                    .thenReturn(orchestrated);

            consumer.listen(rec, ack);

            verify(validator).validate(any());
            verify(producer).send(any());
            verify(ack).acknowledge();
        }
    }

    // ------------------------------
    // VALIDATION EXCEPTION
    // ------------------------------

    @Test
    @DisplayName("Should handle ValidationException and send failure message")
    void shouldHandleValidationException() throws Exception {

        CoreIntegrationRequest request = buildRequest("FraudCheck", "RID-2");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 1L, "key1", json);

        doThrow(new ValidationException("validation error"))
                .when(validator).validate(any());

        consumer.listen(rec, ack);

        verify(producer).send(any());
        verify(ack).acknowledge();
    }

    // ------------------------------
    // GENERIC EXCEPTION
    // ------------------------------

    @Test
    @DisplayName("Should handle generic exception and publish failure")
    void shouldHandleGenericException() throws Exception {

        CoreIntegrationRequest request = buildRequest("FraudCheck", "RID-3");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 1L, "key1", json);

        doThrow(new RuntimeException("unexpected error"))
                .when(validator).validate(any());

        consumer.listen(rec, ack);

        verify(producer).send(any());
        verify(ack).acknowledge();
    }

    // ------------------------------
    // RETRYABLE EXCEPTION
    // ------------------------------

    @Test
    @DisplayName("Should throw RetryableProcessingException for timeout")
    void shouldThrowRetryableException() throws Exception {

        CoreIntegrationRequest request = buildRequest("FraudCheck", "RID-4");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 1L, "key1", json);

        SdkException sdkException = mock(SdkException.class);

        when(sdkException.getErrorCode())
                .thenReturn("TITAN_SERVICE_TIMEOUT");

        doThrow(sdkException).when(validator).validate(any());

        assertThrows(
                RetryableProcessingException.class,
                () -> consumer.listen(rec, ack)
        );
    }

    // ------------------------------
    // INVALID OPERATION
    // ------------------------------

    @Test
    @DisplayName("Should fail when operation is invalid")
    void shouldFailInvalidOperation() throws Exception {

        CoreIntegrationRequest request = buildRequest("InvalidOperation", "RID-5");

        String json = objectMapper.writeValueAsString(request);

        ConsumerRecord<String, Object> rec =
                new ConsumerRecord<>("topic", 0, 1L, "key1", json);

        consumer.listen(rec, ack);

        verify(producer).send(any());
        verify(ack).acknowledge();
    }

    // ------------------------------
    // HELPER METHODS
    // ------------------------------

    private CoreIntegrationRequest buildRequest(String operation, String requestId) {

        Header header = new Header();
        header.setOperation(operation);
        header.setRequestId(requestId);
        header.setSourceId("SRC1");

        CoreIntegrationRequest req = new CoreIntegrationRequest();
        req.setHeader(header);

        return req;
    }

    private static void setPrivateField(Object target,
                                        String fieldName,
                                        Object value) throws Exception {

        Field field = target.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(target, value);
    }
}
