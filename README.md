package com.citi.clm.service.io;

import com.citi.clm.config.IOApiConfig;
import com.citi.clm.dto.io.*;
import com.citi.clm.entity.AnsibleResultRequest;
import com.citi.clm.enums.ExecutionEnvironment;
import com.citi.clm.exception.IOApiException;
import com.citi.clm.repository.AnsibleResultRequestRepository;
import com.citi.clm.service.TransactionLogService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.*;

/**
 * Standalone IO API Service for general-purpose playbook execution.
 * Can be used with or without workflow context.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class IOApiService {
    
    private final IOApiConfig ioApiConfig;
    private final IOAuthService ioAuthService;
    private final RestTemplate restTemplate;
    private final TransactionLogService transactionLogService;
    private final AnsibleResultRequestRepository ansibleResultRequestRepository;
    
    /**
     * Execute playbook without workflow context.
     * General purpose execution for any playbook/action.
     */
    public IOExecuteResponse executePlaybook(IOExecuteRequest request) {
        String url = ioApiConfig.getBaseUrl() + ioApiConfig.getExecuteUrl();
        String token = ioAuthService.getAuthToken();
        
        HttpHeaders headers = buildHeaders(token);
        HttpEntity<IOExecuteRequest> httpRequest = new HttpEntity<>(request, headers);
        
        long startTime = System.currentTimeMillis();
        String transactionId = request.getTransactionId();
        Integer csi = parseCsiFromRequest(request);
        
        try {
            log.info("Executing IO playbook. CSI: {}, Action: {}, TransactionId: {}", 
                csi, getActionFromRequest(request), transactionId);
            
            ResponseEntity<IOExecuteResponse> response = restTemplate.exchange(
                url,
                HttpMethod.POST,
                httpRequest,
                IOExecuteResponse.class
            );
            
            long duration = System.currentTimeMillis() - startTime;
            IOExecuteResponse responseBody = response.getBody();
            
            if (response.getStatusCode().is2xxSuccessful() && responseBody != null) {
                log.info("Playbook execution triggered. OrderId: {}, Status: {}", 
                    responseBody.getOrderId(), responseBody.getStatus());
                
                // Track execution
                trackExecution(request, responseBody, true, null);
                
                // Log API call
                transactionLogService.logIOApiCall(
                    url, "POST", request, responseBody,
                    response.getStatusCode().value(), duration,
                    null, null, csi, getActionFromRequest(request)
                );
                
                return responseBody;
            }
            
            throw new IOApiException(
                "Failed to execute playbook: " + response.getStatusCode(),
                response.getStatusCode().value(),
                url
            );
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            
            // Track failed execution
            trackExecution(request, null, false, e.getMessage());
            
            // Log failure
            transactionLogService.logIOApiCall(
                url, "POST", request, null,
                500, duration,
                null, null, csi, getActionFromRequest(request)
            );
            
            if (e instanceof IOApiException) {
                throw e;
            }
            throw new IOApiException("Error executing playbook: " + e.getMessage(), e);
        }
    }
    
    /**
     * Build execution request for playbook.
     * Simplified builder for common use cases.
     */
    public IOExecuteRequest buildExecuteRequest(
            Integer csi,
            String environment,
            String targetServer,
            String playbookName,
            String action,
            Map<String, String> parameters,
            String transactionId) {
        
        Map<String, String> taskParams = new HashMap<>();
        if (parameters != null) {
            taskParams.putAll(parameters);
        }
        
        // Add standard parameters
        taskParams.put("Module", action);
        taskParams.put("transactionId", transactionId);
        
        IOExecuteRequest.TaskDefinition task = IOExecuteRequest.TaskDefinition.builder()
            .identifier("IOACTION")
            .programmaticName(ioApiConfig.getProgrammaticName())
            .action(action)
            .parameters(taskParams)
            .build();
        
        return IOExecuteRequest.builder()
            .csiAppId(String.valueOf(csi))
            .environment(environment != null ? environment : "PROD")
            .principal("") // TODO: Set principal
            .target(targetServer)
            .ticket("") // TODO: Set ticket if needed
            .tasks(List.of(task))
            .cmtHostUrl(ioApiConfig.getCmtHostUrl())
            .inventoryHostUrl(ioApiConfig.getInventoryHostUrl())
            .transactionId(transactionId)
            .tokenValue("") // TODO: Set PFY token
            .build();
    }
    
    /**
     * Get order status from IO API.
     */
    public IOOrderStatusResponse getOrderStatus(String orderId) {
        String url = ioApiConfig.getBaseUrl() + ioApiConfig.getOrderStatusUrl() + "/" + orderId;
        String token = ioAuthService.getAuthToken();
        
        HttpHeaders headers = buildHeaders(token);
        HttpEntity<Void> request = new HttpEntity<>(headers);
        
        long startTime = System.currentTimeMillis();
        try {
            ResponseEntity<IOOrderStatusResponse> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                request,
                IOOrderStatusResponse.class
            );
            
            long duration = System.currentTimeMillis() - startTime;
            
            if (response.getStatusCode().is2xxSuccessful() && response.getBody() != null) {
                transactionLogService.logIOApiCall(
                    url, "GET", null, response.getBody(),
                    response.getStatusCode().value(), duration,
                    null, null, null, "ORDER_STATUS_CHECK"
                );
                return response.getBody();
            }
            
            throw new IOApiException(
                "Failed to get order status: " + response.getStatusCode(),
                response.getStatusCode().value(),
                url
            );
            
        } catch (Exception e) {
            if (e instanceof IOApiException) {
                throw e;
            }
            throw new IOApiException("Error getting order status: " + e.getMessage(), e);
        }
    }
    
    /**
     * List all pods for a completed order.
     */
    public IOPodListResponse listPods(String orderId) {
        String url = ioApiConfig.getBaseUrl() + ioApiConfig.getListPodsUrl() + 
                     "?orderId=" + orderId;
        String token = ioAuthService.getAuthToken();
        
        HttpHeaders headers = buildHeaders(token);
        HttpEntity<Void> request = new HttpEntity<>(headers);
        
        long startTime = System.currentTimeMillis();
        try {
            ResponseEntity<IOPodListResponse> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                request,
                IOPodListResponse.class
            );
            
            long duration = System.currentTimeMillis() - startTime;
            
            if (response.getStatusCode().is2xxSuccessful() && response.getBody() != null) {
                transactionLogService.logIOApiCall(
                    url, "GET", null, response.getBody(),
                    response.getStatusCode().value(), duration,
                    null, null, null, "LIST_PODS"
                );
                return response.getBody();
            }
            
            throw new IOApiException(
                "Failed to list pods: " + response.getStatusCode(),
                response.getStatusCode().value(),
                url
            );
            
        } catch (Exception e) {
            if (e instanceof IOApiException) {
                throw e;
            }
            throw new IOApiException("Error listing pods: " + e.getMessage(), e);
        }
    }
    
    /**
     * Download execution logs for a specific pod.
     */
    public IOPodLogResponse downloadPodLog(String podName) {
        String url = ioApiConfig.getBaseUrl() + ioApiConfig.getDownloadLogUrl() + 
                     "?podName=" + podName;
        String token = ioAuthService.getAuthToken();
        
        HttpHeaders headers = buildHeaders(token);
        HttpEntity<Void> request = new HttpEntity<>(headers);
        
        long startTime = System.currentTimeMillis();
        try {
            ResponseEntity<IOPodLogResponse> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                request,
                IOPodLogResponse.class
            );
            
            long duration = System.currentTimeMillis() - startTime;
            
            if (response.getStatusCode().is2xxSuccessful() && response.getBody() != null) {
                transactionLogService.logIOApiCall(
                    url, "GET", null, response.getBody(),
                    response.getStatusCode().value(), duration,
                    null, null, null, "DOWNLOAD_POD_LOG"
                );
                return response.getBody();
            }
            
            throw new IOApiException(
                "Failed to download pod log: " + response.getStatusCode(),
                response.getStatusCode().value(),
                url
            );
            
        } catch (Exception e) {
            if (e instanceof IOApiException) {
                throw e;
            }
            throw new IOApiException("Error downloading pod log: " + e.getMessage(), e);
        }
    }
    
    /**
     * Track execution in database for audit.
     */
    private void trackExecution(
            IOExecuteRequest request,
            IOExecuteResponse response,
            boolean success,
            String errorMessage) {
        
        try {
            Integer csi = parseCsiFromRequest(request);
            String action = getActionFromRequest(request);
            
            AnsibleResultRequest record = AnsibleResultRequest.builder()
                .transactionId(request.getTransactionId())
                .servername(request.getTarget())
                .module(action)
                .executionEnvironment(ExecutionEnvironment.IO)
                .executionStatus(success ? "EXECUTING" : "FAILED")
                .createdDate(new Date())
                .csi(csi)
                .build();
            
            if (response != null) {
                record.setOrderId(response.getOrderId());
                record.setIoOrderStatus(response.getStatus());
            }
            
            if (errorMessage != null) {
                record.setExecutionLog(errorMessage);
            }
            
            ansibleResultRequestRepository.save(record);
            
        } catch (Exception e) {
            log.error("Failed to track execution: {}", e.getMessage());
        }
    }
    
    private HttpHeaders buildHeaders(String token) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(token);
        headers.set("X-BYPASS-OVERRIDE", String.valueOf(ioApiConfig.isBypassOverride()));
        headers.set("X-EXECUTION-MODE", ioApiConfig.getExecutionMode());
        headers.set("X-VALIDATION-MODE", ioApiConfig.getValidationMode());
        return headers;
    }
    
    private Integer parseCsiFromRequest(IOExecuteRequest request) {
        try {
            return Integer.parseInt(request.getCsiAppId());
        } catch (Exception e) {
            return null;
        }
    }
    
    private String getActionFromRequest(IOExecuteRequest request) {
        if (request.getTasks() != null && !request.getTasks().isEmpty()) {
            return request.getTasks().get(0).getAction();
        }
        return "UNKNOWN";
    }
}


package com.citi.clm.entity;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;

import java.util.Date;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Document(collection = "server_inventory")
public class ServerInventory {
    
    @Id
    private String id;
    
    @Field("CSI")
    private Integer csi;
    
    private String hostname;
    private String ipAddress;
    private String environment; // DEV, UAT, PROD
    
    // Server details
    private String osType;
    private String osVersion;
    private String productType; // WAS, IHS, Apache, etc.
    
    // Connection status
    private String connectionStatus; // SUCCESS, FAILED, UNKNOWN
    private Date lastConnectionCheck;
    private String lastConnectionError;
    
    // Ansible/IO details
    private String ansibleUser;
    private boolean ansibleEnabled;
    private String executionEnvironment; // IO, AAP
    
    // Certificate scan status
    private Date lastScanDate;
    private String lastScanStatus; // COMPLETED, FAILED, IN_PROGRESS, PENDING
    private String lastScanOrderId;
    private Integer certificatesFound;
    
    // Metadata
    private Date createdDate;
    private Date updatedDate;
    private String createdBy;
    private String updatedBy;
    
    // Active/Inactive flag
    private boolean active;
}


package com.citi.clm.entity;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import java.util.Date;
import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Document(collection = "scan_executions")
public class ScanExecution {
    
    @Id
    private String id;
    
    private Date scanDate;
    private String status; // INITIATED, IN_PROGRESS, COMPLETED, FAILED
    
    // Batch statistics
    private int totalCsis;
    private int processedCsis;
    private int totalServers;
    private int processedServers;
    private int successfulServers;
    private int failedServers;
    private int skippedServers;
    
    // Timing
    private Date startTime;
    private Date endTime;
    private long durationMs;
    
    // Batch details
    private List<CsiBatchStatus> csiBatches;
    
    // Scaling issues tracking
    private boolean hasScalingIssues;
    private List<ScalingIssue> scalingIssues;
    
    // Metadata
    private String triggeredBy;
    private String scanType; // FULL, CSI_SPECIFIC, SERVER_SPECIFIC
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class CsiBatchStatus {
        private Integer csi;
        private int totalServers;
        private int processedServers;
        private int successfulServers;
        private int failedServers;
        private String status; // PENDING, IN_PROGRESS, COMPLETED, FAILED
        private Date startTime;
        private Date endTime;
        private long durationMs;
    }
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class ScalingIssue {
        private String issueType; // RATE_LIMIT, TIMEOUT, CONCURRENT_LIMIT, API_ERROR
        private String description;
        private Date timestamp;
        private Integer csi;
        private String servername;
        private int retryCount;
    }
}

package com.citi.clm.repository;

import com.citi.clm.entity.ServerInventory;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.Date;
import java.util.List;
import java.util.Optional;

@Repository
public interface ServerInventoryRepository extends MongoRepository<ServerInventory, String> {
    
    Optional<ServerInventory> findByHostname(String hostname);
    
    List<ServerInventory> findByCsi(Integer csi);
    
    // Find active servers with successful connection
    @Query("{'csi': ?0, 'active': true, 'connectionStatus': 'SUCCESS'}")
    List<ServerInventory> findActiveByCsiWithSuccessfulConnection(Integer csi);
    
    // Find all active servers with successful connection
    @Query("{'active': true, 'connectionStatus': 'SUCCESS'}")
    List<ServerInventory> findAllActiveWithSuccessfulConnection();
    
    // Find servers that need connection check
    @Query("{'active': true, 'lastConnectionCheck': {$lt: ?0}}")
    List<ServerInventory> findServersNeedingConnectionCheck(Date cutoffDate);
    
    // Find servers pending scan
    @Query("{'active': true, 'connectionStatus': 'SUCCESS', 'lastScanStatus': {$in: ['PENDING', 'FAILED', null]}}")
    List<ServerInventory> findServersPendingScan();
    
    // Count active servers by CSI
    @Query(value = "{'csi': ?0, 'active': true, 'connectionStatus': 'SUCCESS'}", count = true)
    long countActiveByCsiWithSuccessfulConnection(Integer csi);
    
    // Find distinct CSIs with active servers
    @Query(value = "{'active': true, 'connectionStatus': 'SUCCESS'}", fields = "{'csi': 1}")
    List<ServerInventory> findDistinctCsisWithActiveServers();
}

package com.citi.clm.repository;

import com.citi.clm.entity.ScanExecution;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.Date;
import java.util.List;
import java.util.Optional;

@Repository
public interface ScanExecutionRepository extends MongoRepository<ScanExecution, String> {
    
    // Find latest scan
    Optional<ScanExecution> findTopByOrderByScanDateDesc();
    
    // Find scans by date range
    List<ScanExecution> findByScanDateBetween(Date startDate, Date endDate);
    
    // Find scans with scaling issues
    @Query("{'hasScalingIssues': true}")
    List<ScanExecution> findScansWithScalingIssues();
    
    // Find active/running scans
    @Query("{'status': {$in: ['INITIATED', 'IN_PROGRESS']}}")
    List<ScanExecution> findActiveScanExecutions();
    
    // Find scans by triggered by
    List<ScanExecution> findByTriggeredBy(String triggeredBy);
}


package com.citi.clm.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Data
@Configuration
@ConfigurationProperties(prefix = "clm.scanner")
public class ScannerConfig {
    
    // Scheduler configuration
    private boolean enabled = true;
    private String scanCron = "0 0 3 * * ?"; // Daily at 3 AM
    
    // Batch processing
    private int csiConcurrentLimit = 5; // Process 5 CSIs concurrently
    private int serverBatchSize = 20; // Scan 20 servers per batch
    private int maxServersPerCsi = 500; // Max servers to scan per CSI
    
    // Rate limiting
    private int maxRequestsPerMinute = 60;
    private int delayBetweenBatchesMs = 1000; // 1 second delay between batches
    private int delayBetweenCsisMs = 5000; // 5 seconds delay between CSIs
    
    // Retry configuration
    private int maxRetries = 3;
    private int retryDelayMs = 5000;
    
    // Timeout configuration
    private int scanTimeoutMinutes = 30;
    private int connectionCheckTimeoutSeconds = 60;
    
    // Playbook configuration
    private String scanPlaybookName = "clm_certificate_scan"; // TODO: Provide actual name
    private String scanAction = "AUTOSCAN";
    
    // Scaling issue detection
    private int scalingIssueThreshold = 10; // Log scaling issue after 10 failures
    private boolean pauseOnScalingIssue = true;
    
    // Connection check
    private int connectionCheckIntervalHours = 24; // Check connection every 24 hours
}

package com.citi.clm.service;

import com.citi.clm.config.ScannerConfig;
import com.citi.clm.dto.io.IOExecuteRequest;
import com.citi.clm.dto.io.IOExecuteResponse;
import com.citi.clm.entity.*;
import com.citi.clm.entity.ScanExecution.CsiBatchStatus;
import com.citi.clm.entity.ScanExecution.ScalingIssue;
import com.citi.clm.exception.IOApiException;
import com.citi.clm.repository.ScanExecutionRepository;
import com.citi.clm.repository.ServerInventoryRepository;
import com.citi.clm.service.io.IOApiService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

/**
 * Certificate Scanner Service - Scans all servers CSI-wise to identify certificates.
 * Processes in batches with rate limiting and scaling issue detection.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class CertificateScannerService {
    
    private final ScannerConfig scannerConfig;
    private final ServerInventoryRepository serverInventoryRepository;
    private final ScanExecutionRepository scanExecutionRepository;
    private final IOApiService ioApiService;
    private final TransactionLogService transactionLogService;
    
    private final ExecutorService csiExecutor = Executors.newFixedThreadPool(5);
    private final Semaphore rateLimiter = new Semaphore(60); // 60 requests per minute
    private final ScheduledExecutorService rateLimiterRefresh = 
        Executors.newScheduledThreadPool(1);
    
    // Rate limiter refresh - replenish permits every minute
    {
        rateLimiterRefresh.scheduleAtFixedRate(() -> {
            int currentPermits = rateLimiter.availablePermits();
            if (currentPermits < scannerConfig.getMaxRequestsPerMinute()) {
                rateLimiter.release(scannerConfig.getMaxRequestsPerMinute() - currentPermits);
            }
        }, 1, 1, TimeUnit.MINUTES);
    }
    
    /**
     * Scheduled certificate scan - runs daily at configured time.
     */
    @Scheduled(cron = "${clm.scanner.scan-cron:0 0 3 * * ?}")
    public void scheduledCertificateScan() {
        if (!scannerConfig.isEnabled()) {
            log.info("Certificate scanner is disabled");
            return;
        }
        
        log.info("Starting scheduled certificate scan");
        
        // Check if there's already a scan running
        List<ScanExecution> activeScans = scanExecutionRepository.findActiveScanExecutions();
        if (!activeScans.isEmpty()) {
            log.warn("Scan already in progress. Skipping this run.");
            return;
        }
        
        executeCertificateScan("SCHEDULER");
    }
    
    /**
     * Execute certificate scan across all CSIs.
     */
    public ScanExecution executeCertificateScan(String triggeredBy) {
        log.info("Initiating certificate scan. Triggered by: {}", triggeredBy);
        
        Date scanDate = new Date();
        long startTime = System.currentTimeMillis();
        
        // Create scan execution record
        ScanExecution scanExecution = ScanExecution.builder()
            .scanDate(scanDate)
            .status("INITIATED")
            .startTime(scanDate)
            .triggeredBy(triggeredBy)
            .scanType("FULL")
            .csiBatches(new ArrayList<>())
            .scalingIssues(new ArrayList<>())
            .hasScalingIssues(false)
            .build();
        
        scanExecution = scanExecutionRepository.save(scanExecution);
        
        try {
            // Get all distinct CSIs with active servers
            List<Integer> csis = getDistinctCsisWithActiveServers();
            
            log.info("Found {} CSIs with active servers", csis.size());
            
            scanExecution.setTotalCsis(csis.size());
            scanExecution.setStatus("IN_PROGRESS");
            scanExecution = scanExecutionRepository.save(scanExecution);
            
            // Process CSIs in batches
            int processedCsis = 0;
            int totalServers = 0;
            int processedServers = 0;
            int successfulServers = 0;
            int failedServers = 0;
            int skippedServers = 0;
            
            for (Integer csi : csis) {
                try {
                    log.info("Processing CSI: {} ({}/{})", csi, processedCsis + 1, csis.size());
                    
                    // Process this CSI
                    CsiBatchStatus batchStatus = processCsi(csi, scanExecution);
                    
                    // Update statistics
                    totalServers += batchStatus.getTotalServers();
                    processedServers += batchStatus.getProcessedServers();
                    successfulServers += batchStatus.getSuccessfulServers();
                    failedServers += batchStatus.getFailedServers();
                    
                    scanExecution.getCsiBatches().add(batchStatus);
                    processedCsis++;
                    
                    // Update scan execution
                    scanExecution.setProcessedCsis(processedCsis);
                    scanExecution.setTotalServers(totalServers);
                    scanExecution.setProcessedServers(processedServers);
                    scanExecution.setSuccessfulServers(successfulServers);
                    scanExecution.setFailedServers(failedServers);
                    scanExecution = scanExecutionRepository.save(scanExecution);
                    
                    // Delay between CSIs to avoid overwhelming IO API
                    if (processedCsis < csis.size()) {
                        Thread.sleep(scannerConfig.getDelayBetweenCsisMs());
                    }
                    
                    // Check if we should pause due to scaling issues
                    if (scannerConfig.isPauseOnScalingIssue() && scanExecution.isHasScalingIssues()) {
                        int scalingIssueCount = scanExecution.getScalingIssues().size();
                        if (scalingIssueCount >= scannerConfig.getScalingIssueThreshold()) {
                            log.error("Too many scaling issues detected ({}). Pausing scan.", 
                                scalingIssueCount);
                            break;
                        }
                    }
                    
                } catch (Exception e) {
                    log.error("Error processing CSI {}: {}", csi, e.getMessage(), e);
                    
                    // Log as scaling issue
                    logScalingIssue(scanExecution, "CSI_PROCESSING_ERROR", 
                        "Failed to process CSI: " + e.getMessage(), csi, null);
                }
            }
            
            // Complete scan
            long endTime = System.currentTimeMillis();
            scanExecution.setStatus("COMPLETED");
            scanExecution.setEndTime(new Date());
            scanExecution.setDurationMs(endTime - startTime);
            scanExecution = scanExecutionRepository.save(scanExecution);
            
            log.info("Certificate scan completed. Processed {}/{} CSIs, {}/{} servers successful",
                processedCsis, csis.size(), successfulServers, totalServers);
            
            return scanExecution;
            
        } catch (Exception e) {
            log.error("Certificate scan failed: {}", e.getMessage(), e);
            
            scanExecution.setStatus("FAILED");
            scanExecution.setEndTime(new Date());
            scanExecution.setDurationMs(System.currentTimeMillis() - startTime);
            scanExecutionRepository.save(scanExecution);
            
            throw new RuntimeException("Certificate scan failed", e);
        }
    }
    
    /**
     * Process all servers for a single CSI.
     */
    private CsiBatchStatus processCsi(Integer csi, ScanExecution scanExecution) {
        log.info("Processing servers for CSI: {}", csi);
        
        Date startTime = new Date();
        
        // Get all active servers with successful connection for this CSI
        List<ServerInventory> servers = serverInventoryRepository
            .findActiveByCsiWithSuccessfulConnection(csi);
        
        log.info("Found {} active servers with successful connection for CSI {}", 
            servers.size(), csi);
        
        CsiBatchStatus batchStatus = CsiBatchStatus.builder()
            .csi(csi)
            .totalServers(servers.size())
            .processedServers(0)
            .successfulServers(0)
            .failedServers(0)
            .status("IN_PROGRESS")
            .startTime(startTime)
            .build();
        
        // Limit servers per CSI if configured
        if (servers.size() > scannerConfig.getMaxServersPerCsi()) {
            log.warn("CSI {} has {} servers. Limiting to {}", 
                csi, servers.size(), scannerConfig.getMaxServersPerCsi());
            servers = servers.subList(0, scannerConfig.getMaxServersPerCsi());
            batchStatus.setTotalServers(servers.size());
        }
        
        // Process servers in batches
        int successfulServers = 0;
        int failedServers = 0;
        int processedServers = 0;
        
        for (int i = 0; i < servers.size(); i += scannerConfig.getServerBatchSize()) {
            int endIndex = Math.min(i + scannerConfig.getServerBatchSize(), servers.size());
            List<ServerInventory> batch = servers.subList(i, endIndex);
            
            log.info("Processing batch {}-{} of {} for CSI {}", 
                i + 1, endIndex, servers.size(), csi);
            
            for (ServerInventory server : batch) {
                try {
                    // Acquire rate limit permit
                    boolean acquired = rateLimiter.tryAcquire(5, TimeUnit.SECONDS);
                    if (!acquired) {
                        log.warn("Rate limit reached. Waiting...");
                        rateLimiter.acquire(); // Block until permit available
                        
                        // Log potential scaling issue
                        logScalingIssue(scanExecution, "RATE_LIMIT", 
                            "Rate limit reached during scan", csi, server.getHostname());
                    }
                    
                    // Execute scan on server
                    boolean success = scanServer(server, csi);
                    
                    if (success) {
                        successfulServers++;
                    } else {
                        failedServers++;
                    }
                    
                    processedServers++;
                    
                } catch (Exception e) {
                    log.error("Error scanning server {}: {}", server.getHostname(), e.getMessage());
                    failedServers++;
                    processedServers++;
                    
                    // Log scaling issue if it's an IO API error
                    if (e instanceof IOApiException) {
                        logScalingIssue(scanExecution, "API_ERROR", 
                            "IO API error: " + e.getMessage(), csi, server.getHostname());
                    }
                }
            }
            
            // Update batch status
            batchStatus.setProcessedServers(processedServers);
            batchStatus.setSuccessfulServers(successfulServers);
            batchStatus.setFailedServers(failedServers);
            
            // Delay between batches
            if (endIndex < servers.size()) {
                try {
                    Thread.sleep(scannerConfig.getDelayBetweenBatchesMs());
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
        
        // Complete batch
        batchStatus.setStatus("COMPLETED");
        batchStatus.setEndTime(new Date());
        batchStatus.setDurationMs(
            batchStatus.getEndTime().getTime() - batchStatus.getStartTime().getTime()
        );
        
        log.info("Completed CSI {}. Processed {}/{} servers, {} successful, {} failed",
            csi, processedServers, servers.size(), successfulServers, failedServers);
        
        return batchStatus;
    }
    
    /**
     * Scan a single server for certificates.
     */
    private boolean scanServer(ServerInventory server, Integer csi) {
        log.debug("Scanning server: {}", server.getHostname());
        
        String transactionId = UUID.randomUUID().toString();
        
        try {
            // Build scan parameters
            Map<String, String> parameters = new HashMap<>();
            parameters.put("hostname", server.getHostname());
            parameters.put("osType", server.getOsType());
            parameters.put("productType", server.getProductType());
            parameters.put("scanType", "FULL");
            
            // Build IO request
            IOExecuteRequest request = ioApiService.buildExecuteRequest(
                csi,
                server.getEnvironment(),
                server.getHostname(),
                scannerConfig.getScanPlaybookName(),
                scannerConfig.getScanAction(),
                parameters,
                transactionId
            );
            
            // Execute scan
            IOExecuteResponse response = ioApiService.executePlaybook(request);
            
            // Update server scan status
            server.setLastScanDate(new Date());
            server.setLastScanStatus("IN_PROGRESS");
            server.setLastScanOrderId(response.getOrderId());
            server.setUpdatedDate(new Date());
            serverInventoryRepository.save(server);
            
            log.info("Scan initiated for server {}. OrderId: {}", 
                server.getHostname(), response.getOrderId());
            
            return true;
            
        } catch (Exception e) {
            log.error("Failed to scan server {}: {}", server.getHostname(), e.getMessage());
            
            // Update server with failure
            server.setLastScanDate(new Date());
            server.setLastScanStatus("FAILED");
            server.setUpdatedDate(new Date());
            serverInventoryRepository.save(server);
            
            return false;
        }
    }
    
    /**
     * Log a scaling issue.
     */
    private void logScalingIssue(
            ScanExecution scanExecution,
            String issueType,
            String description,
            Integer csi,
            String servername) {
        
        ScalingIssue issue = ScalingIssue.builder()
            .issueType(issueType)
            .description(description)
            .timestamp(new Date())
            .csi(csi)
            .servername(servername)
            .retryCount(0)
            .build();
        
        scanExecution.getScalingIssues().add(issue);
        scanExecution.setHasScalingIssues(true);
        
        scanExecutionRepository.save(scanExecution);
        
        log.warn("Scaling issue detected: {} - {} (CSI: {}, Server: {})", 
            issueType, description, csi, servername);
    }
    
    /**
     * Get distinct CSIs with active servers.
     */
    private List<Integer> getDistinctCsisWithActiveServers() {
        List<ServerInventory> servers = serverInventoryRepository
            .findAllActiveWithSuccessfulConnection();
        
        return servers.stream()
            .map(ServerInventory::getCsi)
            .distinct()
            .sorted()
            .collect(Collectors.toList());
    }
    
    /**
     * Get scan execution statistics.
     */
    public ScanExecution getLatestScanExecution() {
        return scanExecutionRepository.findTopByOrderByScanDateDesc()
            .orElse(null);
    }
    
    /**
     * Get scans with scaling issues.
     */
    public List<ScanExecution> getScansWithScalingIssues() {
        return scanExecutionRepository.findScansWithScalingIssues();
    }
    
    /**
     * Manually trigger scan for a specific CSI.
     */
    public CsiBatchStatus scanCsi(Integer csi, String triggeredBy) {
        log.info("Manual scan triggered for CSI {} by {}", csi, triggeredBy);
        
        ScanExecution scanExecution = ScanExecution.builder()
            .scanDate(new Date())
            .status("IN_PROGRESS")
            .startTime(new Date())
            .triggeredBy(triggeredBy)
            .scanType("CSI_SPECIFIC")
            .totalCsis(1)
            .processedCsis(0)
            .csiBatches(new ArrayList<>())
            .scalingIssues(new ArrayList<>())
            .hasScalingIssues(false)
            .build();
        
        scanExecution = scanExecutionRepository.save(scanExecution);
        
        CsiBatchStatus batchStatus = processCsi(csi, scanExecution);
        
        scanExecution.getCsiBatches().add(batchStatus);
        scanExecution.setProcessedCsis(1);
        scanExecution.setTotalServers(batchStatus.getTotalServers());
        scanExecution.setProcessedServers(batchStatus.getProcessedServers());
        scanExecution.setSuccessfulServers(batchStatus.getSuccessfulServers());
        scanExecution.setFailedServers(batchStatus.getFailedServers());
        scanExecution.setStatus("COMPLETED");
        scanExecution.setEndTime(new Date());
        scanExecution.setDurationMs(
            scanExecution.getEndTime().getTime() - scanExecution.getStartTime().getTime()
        );
        
        scanExecutionRepository.save(scanExecution);
        
        return batchStatus;
    }
}

package com.citi.clm.controller;

import com.citi.clm.entity.ScanExecution;
import com.citi.clm.entity.ScanExecution.CsiBatchStatus;
import com.citi.clm.entity.ServerInventory;
import com.citi.clm.repository.ScanExecutionRepository;
import com.citi.clm.repository.ServerInventoryRepository;
import com.citi.clm.service.CertificateScannerService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/v1/scanner")
@RequiredArgsConstructor
@Tag(name = "Certificate Scanner", description = "Certificate scanning and inventory management")
public class ScannerAdminController {
    
    private final CertificateScannerService certificateScannerService;
    private final ScanExecutionRepository scanExecutionRepository;
    private final ServerInventoryRepository serverInventoryRepository;
    
    /**
     * Manually trigger full certificate scan across all CSIs.
     */
    @PostMapping("/scan/trigger")
    @Operation(summary = "Trigger full certificate scan", 
               description = "Manually initiate certificate scan across all CSIs")
    public ResponseEntity<?> triggerFullScan(
            @RequestHeader(value = "X-User-Id", required = false) String userId) {
        
        log.info("Manual full scan triggered by: {}", userId);
        
        try {
            String triggeredBy = userId != null ? userId : "MANUAL";
            ScanExecution execution = certificateScannerService.executeCertificateScan(triggeredBy);
            
            return ResponseEntity.ok(Map.of(
                "status", "SUCCESS",
                "scanExecutionId", execution.getId(),
                "message", "Certificate scan initiated successfully"
            ));
            
        } catch (Exception e) {
            log.error("Error triggering scan: {}", e.getMessage(), e);
            return ResponseEntity.internalServerError().body(Map.of(
                "status", "ERROR",
                "message", e.getMessage()
            ));
        }
    }
    
    /**
     * Trigger scan for a specific CSI.
     */
    @PostMapping("/scan/csi/{csi}")
    @Operation(summary = "Scan specific CSI", 
               description = "Trigger certificate scan for a specific CSI")
    public ResponseEntity<?> scanCsi(
            @PathVariable Integer csi,
            @RequestHeader(value = "X-User-Id", required = false) String userId) {
        
        log.info("Manual CSI scan triggered for CSI {} by: {}", csi, userId);
        
        try {
            String triggeredBy = userId != null ? userId : "MANUAL";
            CsiBatchStatus batchStatus = certificateScannerService.scanCsi(csi, triggeredBy);
            
            return ResponseEntity.ok(Map.of(
                "status", "SUCCESS",
                "csi", csi,
                "totalServers", batchStatus.getTotalServers(),
                "successfulServers", batchStatus.getSuccessfulServers(),
                "failedServers", batchStatus.getFailedServers(),
                "message", "CSI scan completed"
            ));
            
        } catch (Exception e) {
            log.error("Error scanning CSI {}: {}", csi, e.getMessage(), e);
            return ResponseEntity.internalServerError().body(Map.of(
                "status", "ERROR",
                "message", e.getMessage()
            ));
        }
    }
    
    /**
     * Get latest scan execution details.
     */
    @GetMapping("/scan/latest")
    @Operation(summary = "Get latest scan", 
               description = "Retrieve details of the most recent scan execution")
    public ResponseEntity<?> getLatestScan() {
        ScanExecution execution = certificateScannerService.getLatestScanExecution();
        
        if (execution == null) {
            return ResponseEntity.notFound().build();
        }
        
        return ResponseEntity.ok(execution);
    }
    
    /**
     * Get scan execution by ID.
     */
    @GetMapping("/scan/{scanId}")
    @Operation(summary = "Get scan details", 
               description = "Retrieve details of a specific scan execution")
    public ResponseEntity<?> getScanExecution(@PathVariable String scanId) {
        return scanExecutionRepository.findById(scanId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    /**
     * Get all scan executions.
     */
    @GetMapping("/scan/all")
    @Operation(summary = "Get all scans", 
               description = "List all scan executions")
    public ResponseEntity<List<ScanExecution>> getAllScans() {
        return ResponseEntity.ok(scanExecutionRepository.findAll());
    }
    
    /**
     * Get scans with scaling issues.
     */
    @GetMapping("/scan/scaling-issues")
    @Operation(summary = "Get scans with scaling issues", 
               description = "List all scans that encountered scaling issues")
    public ResponseEntity<List<ScanExecution>> getScansWithScalingIssues() {
        return ResponseEntity.ok(certificateScannerService.getScansWithScalingIssues());
    }
    
    /**
     * Get active scan executions.
     */
    @GetMapping("/scan/active")
    @Operation(summary = "Get active scans", 
               description = "List currently running scan executions")
    public ResponseEntity<List<ScanExecution>> getActiveScans() {
        return ResponseEntity.ok(scanExecutionRepository.findActiveScanExecutions());
    }
    
    /**
     * Get server inventory for a CSI.
     */
    @GetMapping("/servers/csi/{csi}")
    @Operation(summary = "Get CSI servers", 
               description = "List all servers for a specific CSI")
    public ResponseEntity<List<ServerInventory>> getServersByCsi(@PathVariable Integer csi) {
        return ResponseEntity.ok(serverInventoryRepository.findByCsi(csi));
    }
    
    /**
     * Get active servers with successful connection for a CSI.
     */
    @GetMapping("/servers/csi/{csi}/active")
    @Operation(summary = "Get active CSI servers", 
               description = "List active servers with successful connection for a CSI")
    public ResponseEntity<List<ServerInventory>> getActiveServersByCsi(@PathVariable Integer csi) {
        return ResponseEntity.ok(
            serverInventoryRepository.findActiveByCsiWithSuccessfulConnection(csi)
        );
    }
    
    /**
     * Get server by hostname.
     */
    @GetMapping("/servers/{hostname}")
    @Operation(summary = "Get server details", 
               description = "Retrieve server inventory details by hostname")
    public ResponseEntity<?> getServerByHostname(@PathVariable String hostname) {
        return serverInventoryRepository.findByHostname(hostname)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    /**
     * Update server connection status.
     */
    @PutMapping("/servers/{hostname}/connection-status")
    @Operation(summary = "Update connection status", 
               description = "Update server connection status")
    public ResponseEntity<?> updateServerConnectionStatus(
            @PathVariable String hostname,
            @RequestBody Map<String, String> request) {
        
        return serverInventoryRepository.findByHostname(hostname)
            .map(server -> {
                server.setConnectionStatus(request.get("connectionStatus"));
                server.setLastConnectionCheck(new java.util.Date());
                server.setLastConnectionError(request.get("errorMessage"));
                server.setUpdatedDate(new java.util.Date());
                serverInventoryRepository.save(server);
                
                return ResponseEntity.ok(Map.of(
                    "status", "SUCCESS",
                    "message", "Connection status updated"
                ));
            })
            .orElse(ResponseEntity.notFound().build());
    }
    
    /**
     * Get scan statistics.
     */
    @GetMapping("/scan/stats")
    @Operation(summary = "Get scan statistics", 
               description = "Get overall scanning statistics")
    public ResponseEntity<?> getScanStatistics() {
        ScanExecution latest = certificateScannerService.getLatestScanExecution();
        List<ScanExecution> withIssues = certificateScannerService.getScansWithScalingIssues();
        List<ScanExecution> active = scanExecutionRepository.findActiveScanExecutions();
        
        long totalServers = serverInventoryRepository.count();
        long activeServers = serverInventoryRepository
            .findAllActiveWithSuccessfulConnection().size();
        
        Map<String, Object> stats = new HashMap<>();
        stats.put("totalServers", totalServers);
        stats.put("activeServers", activeServers);
        stats.put("lastScanDate", latest != null ? latest.getScanDate() : null);
        stats.put("lastScanStatus", latest != null ? latest.getStatus() : null);
        stats.put("scansWithIssuesCount", withIssues.size());
        stats.put("activeScansCount", active.size());
        
        if (latest != null) {
            stats.put("lastScanCsis", latest.getTotalCsis());
            stats.put("lastScanServersProcessed", latest.getProcessedServers());
            stats.put("lastScanServersSuccessful", latest.getSuccessfulServers());
            stats.put("lastScanServersFailed", latest.getFailedServers());
            stats.put("lastScanDurationMs", latest.getDurationMs());
        }
        
        return ResponseEntity.ok(stats);
    }
}


package com.citi.clm.service;

import com.citi.clm.dto.ResultCallbackRequest;
import com.citi.clm.entity.ServerInventory;
import com.citi.clm.repository.ServerInventoryRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.Map;

/**
 * Service to process scan results from certificate scan playbooks.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class ScanResultProcessorService {
    
    private final ServerInventoryRepository serverInventoryRepository;
    private final TransactionLogService transactionLogService;
    
    /**
     * Process scan result callback.
     */
    public void processScanResult(ResultCallbackRequest request) {
        log.info("Processing scan result for server: {}", request.getServername());
        
        try {
            // Find server
            ServerInventory server = serverInventoryRepository
                .findByHostname(request.getServername())
                .orElse(null);
            
            if (server == null) {
                log.warn("Server not found in inventory: {}", request.getServername());
                return;
            }
            
            // Update scan status
            if ("SUCCESS".equalsIgnoreCase(request.getExecutionStatus()) ||
                "COMPLETED".equalsIgnoreCase(request.getExecutionStatus())) {
                
                server.setLastScanStatus("COMPLETED");
                
                // Extract certificate count from result
                Integer certCount = extractCertificateCount(request.getResult());
                if (certCount != null) {
                    server.setCertificatesFound(certCount);
                    log.info("Found {} certificates on server {}", certCount, server.getHostname());
                }
                
            } else {
                server.setLastScanStatus("FAILED");
                log.warn("Scan failed for server {}: {}", 
                    server.getHostname(), request.getExecutionLog());
            }
            
            server.setLastScanDate(new Date());
            server.setUpdatedDate(new Date());
            serverInventoryRepository.save(server);
            
            // Log the result
            transactionLogService.logWorkflowAction(
                null,
                null,
                server.getCsi(),
                "CERTIFICATE_SCAN",
                "Scan for " + server.getHostname(),
                com.citi.clm.enums.ExecutionEnvironment.IO,
                request.getOrderId(),
                "clm_certificate_scan",
                request.getExecutionStatus(),
                null,
                "SYSTEM"
            );
            
        } catch (Exception e) {
            log.error("Error processing scan result: {}", e.getMessage(), e);
        }
    }
    
    /**
     * Extract certificate count from scan result.
     */
    private Integer extractCertificateCount(Object result) {
        if (result == null) {
            return null;
        }
        
        try {
            // Assuming result is a Map with certificate information
            if (result instanceof Map) {
                Map<String, Object> resultMap = (Map<String, Object>) result;
                
                // Try different possible keys
                if (resultMap.containsKey("certificateCount")) {
                    return Integer.parseInt(resultMap.get("certificateCount").toString());
                }
                if (resultMap.containsKey("certificate_count")) {
                    return Integer.parseInt(resultMap.get("certificate_count").toString());
                }
                if (resultMap.containsKey("certificates_found")) {
                    return Integer.parseInt(resultMap.get("certificates_found").toString());
                }
            }
            
            return null;
            
        } catch (Exception e) {
            log.warn("Could not extract certificate count from result: {}", e.getMessage());
            return null;
        }
    }
}

package com.citi.clm.controller;

import com.citi.clm.dto.ResultCallbackRequest;
import com.citi.clm.service.ResultCallbackService;
import com.citi.clm.service.ScanResultProcessorService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/v1")
@RequiredArgsConstructor
@Tag(name = "Result Callback", description = "Endpoints for playbook result callbacks")
public class ResultCallbackController {
    
    private final ResultCallbackService resultCallbackService;
    private final ScanResultProcessorService scanResultProcessorService;
    
    /**
     * Endpoint called by playbooks to report execution results.
     * This is the /result API that playbooks call on completion.
     * Handles both workflow-based and standalone executions (like scans).
     */
    @PostMapping("/result")
    @Operation(summary = "Process playbook result callback", 
               description = "Called by playbooks to report execution status and results")
    public ResponseEntity<Map<String, Object>> processResult(
            @RequestBody ResultCallbackRequest request) {
        
        log.info("Received result callback: transactionId={}, status={}, module={}", 
            request.getTransactionId(), request.getExecutionStatus(), request.getModule());
        
        try {
            // Determine if this is a workflow-based or standalone execution
            boolean isWorkflowExecution = request.getWorkflowInstanceId() != null;
            boolean isScanExecution = isScanModule(request.getModule());
            
            if (isWorkflowExecution) {
                // Process as workflow callback
                boolean processed = resultCallbackService.processCallbackIdempotent(request);
                
                if (processed) {
                    return ResponseEntity.ok(Map.of(
                        "status", "SUCCESS",
                        "message", "Workflow callback processed successfully",
                        "type", "WORKFLOW"
                    ));
                } else {
                    return ResponseEntity.ok(Map.of(
                        "status", "SKIPPED",
                        "message", "Workflow callback already processed",
                        "type", "WORKFLOW"
                    ));
                }
                
            } else if (isScanExecution) {
                // Process as scan result
                scanResultProcessorService.processScanResult(request);
                
                return ResponseEntity.ok(Map.of(
                    "status", "SUCCESS",
                    "message", "Scan result processed successfully",
                    "type", "SCAN"
                ));
                
            } else {
                // Generic standalone execution - just log it
                log.info("Standalone execution completed: {}", request.getTransactionId());
                
                return ResponseEntity.ok(Map.of(
                    "status", "SUCCESS",
                    "message", "Result received and logged",
                    "type", "STANDALONE"
                ));
            }
            
        } catch (IllegalArgumentException e) {
            log.error("Invalid callback request: {}", e.getMessage());
            return ResponseEntity.badRequest().body(Map.of(
                "status", "ERROR",
                "message", e.getMessage()
            ));
        } catch (Exception e) {
            log.error("Error processing callback: {}", e.getMessage(), e);
            return ResponseEntity.internalServerError().body(Map.of(
                "status", "ERROR",
                "message", "Internal error processing callback"
            ));
        }
    }
    
    /**
     * Legacy endpoint for backward compatibility.
     */
    @PostMapping("/ansible/result")
    @Operation(summary = "Process Ansible result callback (legacy)", 
               description = "Legacy endpoint for Ansible Tower callbacks")
    public ResponseEntity<Map<String, Object>> processAnsibleResult(
            @RequestBody ResultCallbackRequest request) {
        return processResult(request);
    }
    
    /**
     * Check if this is a scan-related module.
     */
    private boolean isScanModule(String module) {
        if (module == null) {
            return false;
        }
        
        String moduleLower = module.toLowerCase();
        return moduleLower.contains("scan") || 
               moduleLower.contains("autoscan") ||
               moduleLower.equals("AUTOSCAN");
    }
}


spring:
  application:
    name: clm-service
  
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://localhost:27017/clm}
      database: clm

  jackson:
    default-property-inclusion: non_null
    serialization:
      write-dates-as-timestamps: false

server:
  port: ${SERVER_PORT:8080}

# CLM Configuration
clm:
  # IO API Configuration
  io-api:
    base-url: https://b2b.cti.otservices.citigroup.net
    auth-url: /2loauth-auth-ioservices/v1/oauth/token
    execute-url: /io-service-gateway/v1/order/v3/execute-automations
    order-status-url: /io-service-gateway/v1/order-info
    list-pods-url: /io-service-gateway/v1/execution/list-execution-pods
    download-log-url: /io-service-gateway/v1/execution/download-execution-log
    
    # TODO: Set actual credentials
    basic-auth-credentials: ${IO_API_BASIC_AUTH:}
    grant-type: client_credentials
    scope: /retrieve-2loauth-auth/v1
    
    token-cache-minutes: 50
    connect-timeout-seconds: 30
    read-timeout-seconds: 60
    max-retries: 3
    retry-delay-seconds: 5
    
    # TODO: Set callback URLs
    cmt-host-url: ${CMT_HOST_URL:https://your-clm-callback-url}
    inventory-host-url: ${INVENTORY_HOST_URL:https://your-inventory-url}
    
    programmatic-name: platformforyou_clm
    bypass-override: true
    execution-mode: GRAPH
    validation-mode: DEBUG

  # Certificate Scanner Configuration
  scanner:
    enabled: ${SCANNER_ENABLED:true}
    scan-cron: "0 0 3 * * ?"  # Daily at 3 AM (1 hour after renewal check)
    
    # Batch processing settings
    csi-concurrent-limit: 5
    server-batch-size: 20
    max-servers-per-csi: 500
    
    # Rate limiting (to avoid overwhelming IO API)
    max-requests-per-minute: 60
    delay-between-batches-ms: 1000  # 1 second
    delay-between-csis-ms: 5000     # 5 seconds
    
    # Retry configuration
    max-retries: 3
    retry-delay-ms: 5000
    
    # Timeouts
    scan-timeout-minutes: 30
    connection-check-timeout-seconds: 60
    
    # Playbook configuration
    scan-playbook-name: clm_certificate_scan  # TODO: Provide actual playbook name
    scan-action: AUTOSCAN
    
    # Scaling issue detection
    scaling-issue-threshold: 10
    pause-on-scaling-issue: true
    
    # Connection check interval
    connection-check-interval-hours: 24

  # Workflow Configuration
  workflow:
    default-expiry-threshold-days: 60
    
    # Product types that use new workflow (procurement, deployment, restart)
    new-workflow-product-types:
      - WAS
      - IHS
    
    # New workflow playbooks (for WAS, IHS)
    new-workflow:
      steps:
        procurement:
          playbook-name: clm_procurement  # TODO: Provide actual playbook name
          io-action: PROCUREMENT
          order: 1
          requires-deployment-check: false
          is-optional: false
          timeout-minutes: 30
        deployment:
          playbook-name: clm_deployment  # TODO: Provide actual playbook name
          io-action: DEPLOYMENT
          order: 2
          requires-deployment-check: true
          is-optional: false
          timeout-minutes: 30
        restart:
          playbook-name: clm_restart  # TODO: Provide actual playbook name
          io-action: RESTART
          order: 3
          requires-deployment-check: true
          is-optional: false
          timeout-minutes: 30
    
    # Old workflow playbooks (for other product types)
    old-workflow:
      steps:
        cmp_creation:
          playbook-name: clm_cmp_creation  # TODO: Provide actual playbook name
          io-action: CMP_CREATION
          order: 1
          requires-deployment-check: false
          is-optional: false
          timeout-minutes: 30
        import:
          playbook-name: clm_import  # TODO: Provide actual playbook name
          io-action: IMPORT
          order: 2
          requires-deployment-check: false
          is-optional: false
          timeout-minutes: 30
        push:
          playbook-name: clm_push  # TODO: Provide actual playbook name
          io-action: PUSH
          order: 3
          requires-deployment-check: true
          is-optional: false
          timeout-minutes: 30
        cr_validation:
          playbook-name: clm_cr_validation  # TODO: Provide actual playbook name
          io-action: CR_VALIDATION
          order: 4
          requires-deployment-check: false
          is-optional: false
          timeout-minutes: 30
        restart:
          playbook-name: clm_restart  # TODO: Provide actual playbook name
          io-action: RESTART
          order: 5
          requires-deployment-check: true
          is-optional: false
          timeout-minutes: 30
    
    # Notification and rollback playbooks
    notification-playbook: clm_notification  # TODO: Provide actual playbook name
    rollback-playbook: clm_rollback  # TODO: Provide actual playbook name
    
    step-timeout-minutes: 30

  # Renewal Scheduler Configuration
  scheduler:
    enabled: ${RENEWAL_SCHEDULER_ENABLED:true}
    renewal-check-cron: "0 0 2 * * ?"  # Daily at 2 AM
    batch-size: 100
    max-concurrent-workflows: 10
    stuck-workflow-timeout-hours: 24
    max-retry-attempts: 3
    retry-delay-minutes: 15

# Logging
logging:
  level:
    com.citi.clm: INFO
    org.springframework.data.mongodb: INFO
    org.springframework.web: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"

# Actuator endpoints
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
