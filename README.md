management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0

logging:
  pattern:
    level: "%5p [traceId=%X{traceId}, spanId=%X{spanId}, requestId=%X{requestId:-}, uetr=%X{micro.uetr:-}]"
    ------------------
    @Configuration
public class TracingConfig {

    @Bean
    public CurrentTraceContext currentTraceContext() {
        return ThreadLocalCurrentTraceContext.newBuilder()
                .addScopeDecorator(MDCScopeDecorator.get())
                .build();
    }
}
-----------------------
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class ValidationAspect {

    private final Tracer tracer;
    private final RequestValidationService requestValidationService;

    @Before("@annotation(validateRequest)")
    public void before(JoinPoint joinPoint, ValidateRequest validateRequest) {

        if (!validateRequest.enabled()) {
            return;
        }

        if (joinPoint.getArgs().length > 0 &&
            joinPoint.getArgs()[0] instanceof CoreIntegrationRequest req) {

            String requestId = req.getHeader().getRequestId();
            String uetr = req.getPaymentInfo() != null
                    ? req.getPaymentInfo().getUetr()
                    : "";

            // ✅ GET CURRENT SPAN (DO NOT CREATE NEW)
            Span span = tracer.currentSpan();

            if (span != null) {
                span.tag("requestId", requestId);
                span.tag("micro.uetr", uetr);
            }

            // ✅ PUT INTO MDC FOR LOGGING
            MDC.put("requestId", requestId);
            MDC.put("micro.uetr", uetr);

            requestValidationService.validate(req);
        }
    }
}
--------------
@Configuration
@EnableAsync
@RequiredArgsConstructor
public class ThreadPoolConfig {

    private final Tracer tracer;

    @Bean(name = "virtualThreadExecutor", destroyMethod = "shutdown")
    public ExecutorService virtualThreadExecutor() {

        ExecutorService delegate =
                Executors.newThreadPerTaskExecutor(
                        Thread.ofVirtual().name("vt-", 0).factory()
                );

        return new ContextPropagatingExecutorService(delegate, tracer);
    }
}
------
public class ContextPropagatingExecutorService implements ExecutorService {

    private final ExecutorService delegate;
    private final Tracer tracer;

    public ContextPropagatingExecutorService(ExecutorService delegate, Tracer tracer) {
        this.delegate = delegate;
        this.tracer = tracer;
    }

    private Runnable wrap(Runnable task) {
        Span parent = tracer.currentSpan();
        return () -> {
            try (Tracer.SpanInScope ws = tracer.withSpan(parent)) {
                task.run();
            }
        };
    }

    @Override
    public void execute(Runnable command) {
        delegate.execute(wrap(command));
    }

    @Override public void shutdown() { delegate.shutdown(); }
    @Override public List<Runnable> shutdownNow() { return delegate.shutdownNow(); }
    @Override public boolean isShutdown() { return delegate.isShutdown(); }
    @Override public boolean isTerminated() { return delegate.isTerminated(); }
    @Override public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        return delegate.awaitTermination(timeout, unit);
    }
}
----------------
<pattern>
%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread]
traceId=%X{traceId} spanId=%X{spanId}
requestId=%X{requestId:-} uetr=%X{micro.uetr:-}
- %msg%n
</pattern>
--------------
