import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

@Component
public class MDCFilter implements Filter {

    private static final String REQUEST_ID = "X-Request-Id";
    private static final String UETR = "UETR";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;

        try {
            // 1. Get or generate Request ID
            String requestId = httpRequest.getHeader(REQUEST_ID);
            if (requestId == null || requestId.isEmpty()) {
                requestId = UUID.randomUUID().toString();
            }

            // 2. Optional: extract UETR (if coming in header)
            String uetr = httpRequest.getHeader(UETR);

            // 3. Put into MDC
            MDC.put(REQUEST_ID, requestId);

            if (uetr != null) {
                MDC.put(UETR, uetr);
            }

            // 4. Continue flow
            chain.doFilter(request, response);

        } finally {
            // 5. Clean ONLY what you added
            MDC.remove(REQUEST_ID);
            MDC.remove(UETR);
        }
    }
}

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.web.servlet.FilterRegistrationBean;

@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<MDCFilter> loggingFilter() {
        FilterRegistrationBean<MDCFilter> registrationBean = new FilterRegistrationBean<>();

        registrationBean.setFilter(new MDCFilter());
        registrationBean.addUrlPatterns("/*");
        registrationBean.setOrder(1); // High priority

        return registrationBean;
    }
}

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;

import java.io.IOException;
import java.util.UUID;

public class MDCFilter implements Filter {

    private static final String REQUEST_ID = "X-Request-Id";
    private static final String UETR = "UETR";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String requestId = httpRequest.getHeader(REQUEST_ID);
        if (requestId == null || requestId.isEmpty()) {
            requestId = UUID.randomUUID().toString();
        }

        String uetr = httpRequest.getHeader(UETR);

        try {
            // ✅ Put into MDC
            MDC.put(REQUEST_ID, requestId);
            if (uetr != null) {
                MDC.put(UETR, uetr);
            }

            // ✅ Add response header (THIS is what you asked)
            httpResponse.setHeader(REQUEST_ID, requestId);

            // Continue filter chain
            chain.doFilter(request, response);

        } finally {
            // Clean only your keys
            MDC.remove(REQUEST_ID);
            MDC.remove(UETR);
        }
    }
}
