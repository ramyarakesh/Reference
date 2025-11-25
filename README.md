Phase 2: IO API Integration Layer - Required Classes/Methods
Overview
Phase 2 focuses on building the complete IO API integration layer to replace Ansible Tower. This includes authentication, playbook execution, status tracking, pod management, and log retrieval.

Required Classes and Methods
1. Configuration Classes
IOApiConfig.java ✅ (Already Created)
Purpose: Store all IO API endpoints and configuration
java@Configuration
@ConfigurationProperties(prefix = "clm.io-api")
public class IOApiConfig {
    // Endpoints
    private String baseUrl;
    private String authUrl;
    private String executeUrl;
    private String orderStatusUrl;
    private String listPodsUrl;
    private String downloadLogUrl;
    
    // Authentication
    private String basicAuthCredentials;
    private String grantType;
    private String scope;
    
    // Timeout & Retry
    private int tokenCacheMinutes;
    private int connectTimeoutSeconds;
    private int readTimeoutSeconds;
    private int maxRetries;
    private int retryDelaySeconds;
    
    // Callback URLs
    private String cmtHostUrl;
    private String inventoryHostUrl;
    
    // Other config
    private String programmaticName;
    private boolean bypassOverride;
    private String executionMode;
    private String validationMode;
}
TODO: Set actual values in application.yml

2. DTO Classes (Data Transfer Objects)
IOAuthRequest.java ✅ (Already Created)
javapublic class IOAuthRequest {
    private String grantType;
    private String scope;
}
IOAuthResponse.java ✅ (Already Created)
javapublic class IOAuthResponse {
    @JsonProperty("token_type")
    private String tokenType;
    
    @JsonProperty("access_token")
    private String accessToken;
    
    @JsonProperty("expires_in")
    private String expiresIn;
    
    private String scope;
}
IOExecuteRequest.java ✅ (Already Created)
javapublic class IOExecuteRequest {
    private String csiAppId;
    private String environment;
    private String principal;
    private String target;
    private String ticket;
    private List<TaskDefinition> tasks;
    
    @JsonProperty("cmtHostURL")
    private String cmtHostUrl;
    
    @JsonProperty("inventory_HostURL")
    private String inventoryHostUrl;
    
    private String transactionId;
    
    @JsonProperty("token_value")
    private String tokenValue;
    
    public static class TaskDefinition {
        private String identifier;
        private String programmaticName;
        private String action;
        private Map<String, String> parameters;
    }
}
TODO: Define parameter mappings for each playbook type
IOExecuteResponse.java ✅ (Already Created)
javapublic class IOExecuteResponse {
    private String orderId;
    private String status;
    private String message;
    private List<TaskResponse> tasks;
    
    @JsonProperty("Module")
    private String module;
    
    private String transactionId;
}
IOOrderStatusResponse.java ✅ (Already Created)
javapublic class IOOrderStatusResponse {
    private String id;
    private String status;
    private String message;
    private OrderData data;
    
    public static class OrderData {
        private String csiAppId;
        private String orderStatus;
        private String orderId;
        private String message;
        private String createdDate;
        private String lastModifiedDate;
    }
}
IOPodListResponse.java ✅ (Already Created)
javapublic class IOPodListResponse {
    private List<PodInfo> pods;
    
    public static class PodInfo {
        private String podName;
        private String componentProviderName;
        private String componentProviderVersion;
        private String deployableComponentName;
        private String deployableComponentVersion;
        private String startTime;
        private String status;
    }
}
IOPodLogResponse.java ✅ (Already Created)
javapublic class IOPodLogResponse {
    private String podName;
    private String log;
    private String status;
    private String timestamp;
}

3. Service Classes
IOAuthService.java ✅ (Already Created)
Purpose: Handle authentication token generation and caching
Key Methods:
javapublic class IOAuthService {
    // Public API
    public String getAuthToken();
    public String refreshToken();
    public void clearCache();
    
    // Private methods
    private boolean isTokenValid();
    private IOAuthResponse fetchNewToken();
}
Implementation Details:

Token caching with expiry tracking
Thread-safe using ReentrantLock
Automatic refresh when expired
60-second buffer before expiry

TODO: None - fully implemented

IOExecutionService.java ✅ (Already Created)
Purpose: Execute playbooks and retrieve execution details
Key Methods:
javapublic class IOExecutionService {
    // Execute playbook (fire-and-forget)
    public IOExecuteResponse executePlaybook(
        WorkflowInstance workflowInstance,
        WorkflowStepInstance step,
        CertificateDetails certificate,
        String pfyToken
    );
    
    // Status tracking
    public IOOrderStatusResponse getOrderStatus(String orderId);
    
    // Pod management
    public IOPodListResponse listPods(String orderId);
    public IOPodLogResponse downloadPodLog(String podName);
    
    // Private helper
    private IOExecuteRequest buildExecuteRequest(...);
}
Implementation Details:

Uses IOAuthService for token
Sets custom headers (X-BYPASS-OVERRIDE, X-EXECUTION-MODE, etc.)
Logs all API calls via TransactionLogService
Error handling with IOApiException

TODO Items:

Complete buildExecuteRequest() method:

java   private IOExecuteRequest buildExecuteRequest(...) {
       // TODO: Map certificate fields to parameters
       parameters.put("certificatePath", certificate.getKeyDBPath());
       parameters.put("keystorePassword", "..."); // Encrypted
       
       // TODO: Get principal ID
       String principal = getPrincipalForEnvironment(environment);
       
       // TODO: Generate or retrieve SNOW ticket
       String ticket = generateSnowTicket(certificate);
       
       // TODO: Map environment (DEV/UAT/PROD)
       String environment = mapEnvironment(certificate.getEnvType());
   }

Add parameter mapping for each IO action:

PROCUREMENT: CSR details, CA info
DEPLOYMENT: Certificate path, keystore details
RESTART: Service names, restart commands
CMP_CREATION: CMP parameters
IMPORT: Import parameters
PUSH: Push parameters
CR_VALIDATION: Validation parameters




IOApiRetryHandler.java (NEW - Need to Create)
Purpose: Handle retry logic for transient failures
java@Service
@RequiredArgsConstructor
@Slf4j
public class IOApiRetryHandler {
    
    private final IOApiConfig ioApiConfig;
    
    /**
     * Execute with retry logic
     */
    public <T> T executeWithRetry(Supplier<T> operation, String operationName) {
        int attempt = 0;
        Exception lastException = null;
        
        while (attempt < ioApiConfig.getMaxRetries()) {
            try {
                return operation.get();
            } catch (Exception e) {
                lastException = e;
                attempt++;
                
                if (attempt < ioApiConfig.getMaxRetries()) {
                    log.warn("Attempt {} failed for {}: {}. Retrying...", 
                        attempt, operationName, e.getMessage());
                    
                    try {
                        Thread.sleep(ioApiConfig.getRetryDelaySeconds() * 1000L);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
        
        throw new IOApiException(
            "Operation failed after " + attempt + " attempts: " + operationName,
            lastException
        );
    }
    
    /**
     * Check if error is retryable
     */
    public boolean isRetryableError(Exception e) {
        // Network errors, timeouts, 5xx errors are retryable
        if (e instanceof IOApiException) {
            IOApiException ioEx = (IOApiException) e;
            int status = ioEx.getHttpStatus();
            return status >= 500 || status == 408 || status == 429;
        }
        return e instanceof java.net.SocketTimeoutException ||
               e instanceof java.net.ConnectException;
    }
}
TODO: Create this class

IOApiCircuitBreaker.java (NEW - Optional but Recommended)
Purpose: Prevent cascading failures when IO API is down
java@Service
@Slf4j
public class IOApiCircuitBreaker {
    
    private enum State { CLOSED, OPEN, HALF_OPEN }
    
    private State state = State.CLOSED;
    private int failureCount = 0;
    private int successCount = 0;
    private LocalDateTime lastFailureTime;
    
    private static final int FAILURE_THRESHOLD = 5;
    private static final int SUCCESS_THRESHOLD = 2;
    private static final int TIMEOUT_MINUTES = 5;
    
    /**
     * Execute operation through circuit breaker
     */
    public <T> T execute(Supplier<T> operation) {
        if (state == State.OPEN) {
            if (shouldAttemptReset()) {
                state = State.HALF_OPEN;
                log.info("Circuit breaker moved to HALF_OPEN state");
            } else {
                throw new IOApiException("Circuit breaker is OPEN - IO API unavailable");
            }
        }
        
        try {
            T result = operation.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }
    
    private synchronized void onSuccess() {
        if (state == State.HALF_OPEN) {
            successCount++;
            if (successCount >= SUCCESS_THRESHOLD) {
                state = State.CLOSED;
                successCount = 0;
                failureCount = 0;
                log.info("Circuit breaker CLOSED - IO API recovered");
            }
        } else {
            failureCount = 0;
        }
    }
    
    private synchronized void onFailure() {
        failureCount++;
        lastFailureTime = LocalDateTime.now();
        
        if (state == State.HALF_OPEN) {
            state = State.OPEN;
            successCount = 0;
            log.warn("Circuit breaker OPEN - IO API still failing");
        } else if (failureCount >= FAILURE_THRESHOLD) {
            state = State.OPEN;
            log.error("Circuit breaker OPEN - IO API failure threshold reached");
        }
    }
    
    private boolean shouldAttemptReset() {
        return lastFailureTime != null &&
               LocalDateTime.now().isAfter(
                   lastFailureTime.plusMinutes(TIMEOUT_MINUTES)
               );
    }
    
    public State getState() {
        return state;
    }
}
TODO: Create this class (optional but recommended for production)

4. Exception Classes
IOApiException.java ✅ (Already Created)
javapublic class IOApiException extends RuntimeException {
    private int httpStatus;
    private String endpoint;
    private String responseBody;
    
    // Constructors with various parameter combinations
}

5. Repository Classes
AnsibleResultRequestRepository.java ✅ (Already Created)
Purpose: Store execution results from both IO and AAP
Key Methods:
javapublic interface AnsibleResultRequestRepository extends MongoRepository<...> {
    List<AnsibleResultRequest> findByTransactionId(String transactionId);
    List<AnsibleResultRequest> findByWorkflowInstanceId(String workflowInstanceId);
    Optional<AnsibleResultRequest> findByOrderId(String orderId);
    List<AnsibleResultRequest> findByCsi(Integer csi);
    List<AnsibleResultRequest> findByCertificateId(String certificateId);
    List<AnsibleResultRequest> findByExecutionEnvironment(ExecutionEnvironment env);
    Optional<AnsibleResultRequest> findByWorkflowInstanceIdAndStepOrder(
        String workflowInstanceId, Integer stepOrder
    );
}

6. Utility Classes
IORequestBuilder.java (NEW - Recommended)
Purpose: Build IO API requests with proper parameter mapping
java@Component
@RequiredArgsConstructor
@Slf4j
public class IORequestBuilder {
    
    private final IOApiConfig ioApiConfig;
    
    /**
     * Build execute request for a workflow step
     */
    public IOExecuteRequest buildExecuteRequest(
            WorkflowInstance workflow,
            WorkflowStepInstance step,
            CertificateDetails certificate,
            String pfyToken) {
        
        // Build task parameters based on IO action
        Map<String, String> parameters = buildParameters(step, certificate);
        
        IOExecuteRequest.TaskDefinition task = IOExecuteRequest.TaskDefinition.builder()
            .identifier("IOACTION")
            .programmaticName(ioApiConfig.getProgrammaticName())
            .action(step.getIoAction())
            .parameters(parameters)
            .build();
        
        return IOExecuteRequest.builder()
            .csiAppId(String.valueOf(workflow.getCsi()))
            .environment(mapEnvironment(certificate.getEnvType()))
            .principal(getPrincipal(certificate))
            .target(certificate.getTargetHostname())
            .ticket(getSnowTicket(workflow, certificate))
            .tasks(List.of(task))
            .cmtHostUrl(ioApiConfig.getCmtHostUrl())
            .inventoryHostUrl(ioApiConfig.getInventoryHostUrl())
            .transactionId(workflow.getTransactionId())
            .tokenValue(pfyToken)
            .build();
    }
    
    /**
     * Build parameters based on action type
     */
    private Map<String, String> buildParameters(
            WorkflowStepInstance step, 
            CertificateDetails certificate) {
        
        Map<String, String> params = new HashMap<>();
        
        switch (step.getIoAction()) {
            case "PROCUREMENT":
                params.put("certificateId", certificate.getId());
                params.put("uniqueNumber", certificate.getUniqueNumber());
                params.put("san", certificate.getSan());
                params.put("csr", certificate.getCsr());
                // TODO: Add CA details, certificate type, etc.
                break;
                
            case "DEPLOYMENT":
                params.put("certificateId", certificate.getId());
                params.put("targetPath", certificate.getKeyDBPath());
                params.put("keystoreType", certificate.getKeyDBType());
                params.put("targetHostname", certificate.getTargetHostname());
                // TODO: Add keystore password (encrypted), backup options
                break;
                
            case "RESTART":
                params.put("targetHostname", certificate.getTargetHostname());
                params.put("productType", certificate.getProductType());
                params.put("instance", certificate.getInstance());
                // TODO: Add service names, restart commands
                break;
                
            case "CMP_CREATION":
                params.put("certificateId", certificate.getId());
                params.put("cmpId", certificate.getCmpId());
                // TODO: Add CMP-specific parameters
                break;
                
            case "IMPORT":
                params.put("certificateId", certificate.getId());
                params.put("keystorePath", certificate.getKeyDBPath());
                // TODO: Add import parameters
                break;
                
            case "PUSH":
                params.put("certificateId", certificate.getId());
                params.put("targetHostname", certificate.getTargetHostname());
                // TODO: Add push parameters
                break;
                
            case "CR_VALIDATION":
                params.put("certificateId", certificate.getId());
                params.put("snowReqNo", certificate.getSnowReqNo());
                // TODO: Add validation parameters
                break;
                
            default:
                log.warn("Unknown IO action: {}", step.getIoAction());
        }
        
        // Common parameters
        params.put("workflowInstanceId", step.getStepOrder().toString());
        params.put("stepOrder", String.valueOf(step.getStepOrder()));
        
        return params;
    }
    
    /**
     * Map environment
     */
    private String mapEnvironment(String envType) {
        if (envType == null) return "PROD";
        
        // TODO: Map your environment values to IO API values
        switch (envType.toUpperCase()) {
            case "DEVELOPMENT":
            case "DEV":
                return "DEV";
            case "UAT":
            case "STAGING":
                return "UAT";
            case "PRODUCTION":
            case "PROD":
            default:
                return "PROD";
        }
    }
    
    /**
     * Get principal for environment
     */
    private String getPrincipal(CertificateDetails certificate) {
        // TODO: Implement principal ID logic
        // May depend on environment, CSI, or other factors
        return "TODO_PRINCIPAL_ID";
    }
    
    /**
     * Get or generate SNOW ticket
     */
    private String getSnowTicket(WorkflowInstance workflow, CertificateDetails certificate) {
        // Check if certificate already has a SNOW ticket
        if (certificate.getSnowReqNo() != null && !certificate.getSnowReqNo().isEmpty()) {
            return certificate.getSnowReqNo();
        }
        
        // TODO: Generate new SNOW ticket or retrieve from SNOW API
        // For now, return placeholder
        return "INC" + System.currentTimeMillis();
    }
}
TODO: Create this class and implement all TODOs

PfyTokenGenerator.java (NEW - Required)
Purpose: Generate PFY bearer token for callback authentication
java@Component
@RequiredArgsConstructor
@Slf4j
public class PfyTokenGenerator {
    
    // TODO: Add required dependencies for token generation
    
    /**
     * Generate PFY token for callback authentication
     */
    public String generateToken() {
        // TODO: Implement token generation logic
        // This might involve:
        // 1. JWT token generation with secret key
        // 2. OAuth token from PFY service
        // 3. Custom token format specific to your organization
        
        // Placeholder implementation
        return "PFY_TOKEN_" + UUID.randomUUID().toString();
    }
    
    /**
     * Validate PFY token from callback request
     */
    public boolean validateToken(String token) {
        // TODO: Implement token validation
        return token != null && token.startsWith("PFY_TOKEN_");
    }
}
TODO: Create this class and implement token generation

7. Testing Classes
IOAuthServiceTest.java (NEW - Recommended)
java@SpringBootTest
public class IOAuthServiceTest {
    
    @Test
    public void testTokenCaching() {
        // Test that token is cached and reused
    }
    
    @Test
    public void testTokenRefresh() {
        // Test that expired token is refreshed
    }
    
    @Test
    public void testConcurrentTokenRequests() {
        // Test thread safety
    }
}
IOExecutionServiceTest.java (NEW - Recommended)
java@SpringBootTest
public class IOExecutionServiceTest {
    
    @MockBean
    private RestTemplate restTemplate;
    
    @Test
    public void testExecutePlaybook() {
        // Mock IO API response
        // Verify request structure
    }
    
    @Test
    public void testGetOrderStatus() {
        // Test status retrieval
    }
    
    @Test
    public void testErrorHandling() {
        // Test various error scenarios
    }
}

Integration Checklist
Phase 2 Implementation Steps:

✅ Configuration

 IOApiConfig created
 Set actual values in application.yml


✅ DTOs

 All request/response DTOs created


✅ Core Services

 IOAuthService with caching
 IOExecutionService skeleton
 Complete buildExecuteRequest() method
 Add parameter mappings for each action


⚠️ Utility Classes (Need to Create)

 IORequestBuilder
 PfyTokenGenerator
 IOApiRetryHandler (recommended)
 IOApiCircuitBreaker (optional)


⚠️ Testing

 Unit tests for IOAuthService
 Unit tests for IOExecutionService
 Integration tests with mocked IO API


⚠️ Configuration Values (Must Provide)

 IO API basic auth credentials
 Callback URLs (CMT, Inventory)
 Principal ID format/logic
 SNOW ticket generation
 PFY token generation
 Parameter mappings for all playbook types




Summary
Already Created:

IOAuthService (complete)
IOExecutionService (90% complete)
All DTO classes
IOApiConfig
Exception handling

Need to Create:

IORequestBuilder
PfyTokenGenerator
IOApiRetryHandler (recommended)
IOApiCircuitBreaker (optional)
Test classes

Need to Complete:

Parameter mapping in buildExecuteRequest()
Principal ID logic
SNOW ticket generation
PFY token generation
Configuration values in application.yml
