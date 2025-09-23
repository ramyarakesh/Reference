package com.citi.cert_management.container.service;

import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.LocalDateTime;
import java.util.*;

/**
 * Service for managing CSI (Customer Service Identifier) profiles
 * Handles CRUD operations and module subscription management
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class CsiService {
    
    private final CsiMetadataRepository csiRepository;
    
    /**
     * Creates a new CSI with module subscriptions
     */
    public CsiDetails createCsiWithModules(EnhancedCsiOnboardingRequest request) {
        log.info("Creating CSI profile: {}", request.getCsiId());
        
        // Check if CSI already exists
        if (csiRepository.existsByCsiId(request.getCsiId())) {
            throw new CsiAlreadyExistsException("CSI already exists: " + request.getCsiId());
        }
        
        CsiDetails csi = new CsiDetails();
        
        // Map basic fields
        csi.setCsiId(request.getCsiId());
        csi.setCsiName(request.getCsiName());
        csi.setRequestorSoeid(request.getRequestorSoeid());
        csi.setEmailDl(request.getEmailDl());
        csi.setEmail(request.getEmail());
        csi.setPreferredRenewal(request.getPreferredRenewal());
        csi.setNotificationEmailList(request.getNotificationEmailList());
        csi.setEscalationMatrix(request.getEscalationMatrix());
        csi.setBitbucketProjectUrl(request.getBitbucketProjectUrl());
        csi.setGithubProjectUrl(request.getGithubProjectUrl());
        
        // Set metadata
        csi.setOnboardedAt(LocalDateTime.now());
        csi.setOnboardedBy(request.getRequestorSoeid());
        csi.setActive(true);
        csi.setLastModified(new Date());
        
        // Create module subscriptions
        List<ModuleSubscription> subscriptions = new ArrayList<>();
        for (ModuleSubscriptionRequest moduleReq : request.getModuleSubscriptions()) {
            ModuleSubscription subscription = new ModuleSubscription();
            subscription.setModule(moduleReq.getModule());
            subscription.setSelected(moduleReq.isSelected());
            subscription.setConfiguration(mapConfiguration(moduleReq.getConfiguration()));
            subscription.setSubscribedAt(LocalDateTime.now());
            subscription.setSubscribedBy(request.getRequestorSoeid());
            subscriptions.add(subscription);
        }
        csi.setModuleSubscriptions(subscriptions);
        
        // Initialize sync status
        csi.setServerSyncStatus(createInitialSyncStatus());
        csi.setCertificateSyncStatus(createInitialSyncStatus());
        csi.setRepositorySyncStatus(createInitialSyncStatus());
        
        return csiRepository.save(csi);
    }
    
    /**
     * Retrieves CSI by ID
     */
    public CsiDetails getCsi(String csiId) {
        return csiRepository.findByCsiId(csiId)
            .orElseThrow(() -> new CsiNotFoundException("CSI not found: " + csiId));
    }
    
    /**
     * Updates CSI configuration
     */
    public CsiDetails updateCsi(String csiId, CsiUpdateRequest request) {
        CsiDetails csi = getCsi(csiId);
        
        // Update allowed fields
        if (request.getEmailDl() != null) {
            csi.setEmailDl(request.getEmailDl());
        }
        if (request.getNotificationEmailList() != null) {
            csi.setNotificationEmailList(request.getNotificationEmailList());
        }
        if (request.getEscalationMatrix() != null) {
            csi.setEscalationMatrix(request.getEscalationMatrix());
        }
        if (request.getPreferredRenewal() != null) {
            csi.setPreferredRenewal(request.getPreferredRenewal());
        }
        
        csi.setLastModified(new Date());
        
        return csiRepository.save(csi);
    }
    
    /**
     * Updates module configuration for a specific module
     */
    public ModuleSubscription updateModuleConfiguration(String csiId, ModuleType moduleType, 
                                                      ModuleConfigurationRequest configRequest) {
        CsiDetails csi = getCsi(csiId);
        
        ModuleSubscription subscription = csi.getModuleSubscriptions().stream()
            .filter(s -> s.getModule() == moduleType)
            .findFirst()
            .orElseThrow(() -> new ModuleNotSubscribedException(
                "Module not found: " + moduleType));
        
        // Update configuration
        ModuleConfiguration newConfig = mapConfiguration(configRequest);
        subscription.setConfiguration(newConfig);
        
        csiRepository.save(csi);
        
        return subscription;
    }
    
    /**
     * Deactivates a CSI
     */
    public void deactivateCsi(String csiId) {
        CsiDetails csi = getCsi(csiId);
        csi.setActive(false);
        csi.setLastModified(new Date());
        csiRepository.save(csi);
    }
    
    /**
     * Gets all active CSIs
     */
    public List<CsiDetails> getActiveCsis() {
        return csiRepository.findByActiveTrue();
    }
    
    /**
     * Updates sync status for a specific sync type
     */
    public void updateSyncStatus(String csiId, String syncType, SyncState state, String error) {
        CsiDetails csi = getCsi(csiId);
        
        SyncStatus status = null;
        switch (syncType) {
            case "SERVER_SYNC":
                status = csi.getServerSyncStatus();
                break;
            case "CERTIFICATE_SYNC":
                status = csi.getCertificateSyncStatus();
                break;
            case "REPOSITORY_SYNC":
                status = csi.getRepositorySyncStatus();
                break;
        }
        
        if (status != null) {
            status.setState(state);
            status.setLastAttempt(LocalDateTime.now());
            if (state == SyncState.COMPLETED) {
                status.setLastSuccess(LocalDateTime.now());
                csi.setLastSuccessfulSync(LocalDateTime.now());
            }
            if (error != null) {
                status.setErrorMessage(error);
            }
        }
        
        csiRepository.save(csi);
    }
    
    private ModuleConfiguration mapConfiguration(ModuleConfigurationRequest request) {
        ModuleConfiguration config = new ModuleConfiguration();
        
        // CLM configuration
        config.setVmDiscoveryEnabled(request.isVmDiscoveryEnabled());
        config.setContainerDiscoveryEnabled(request.isContainerDiscoveryEnabled());
        config.setAutoDeploymentEnabled(request.isAutoDeploymentEnabled());
        
        // BCM configuration
        config.setBcmMode(request.getBcmMode());
        config.setBcmExceptionTypes(request.getBcmExceptionTypes());
        config.setBcmAutoRemediationEnabled(
            request.getBcmMode() == BcmMode.WITH_AUTO_REMEDIATION);
        
        // EOVS configuration
        config.setEovsMode(request.getEovsMode());
        config.setEovsUserAcknowledgmentRequired(
            request.getEovsMode() == EovsMode.WITH_USER_ACKNOWLEDGED_REMEDIATION);
        config.setEovsChecks(request.getEovsChecks());
        
        return config;
    }
    
    private SyncStatus createInitialSyncStatus() {
        SyncStatus status = new SyncStatus();
        status.setState(SyncState.NOT_STARTED);
        status.setRecordsProcessed(0);
        status.setRecordsFailed(0);
        return status;
    }
}


package com.citi.cert_management.container.service;

import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Service for managing CLM reports
 * Handles report generation, retrieval, and search operations
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class ReportService {
    
    private final ClmReportRepository reportRepository;
    private final ClmJobRepository jobRepository;
    private final CsiMetadataRepository csiRepository;
    private final ActionItemRepository actionItemRepository;
    
    /**
     * Generates inventory report for a CSI
     */
    public ClmReport generateInventoryReport(String csiId, ReportFormat format) {
        log.info("Generating inventory report for CSI: {} in format: {}", csiId, format);
        
        CsiDetails csi = csiRepository.findByCsiId(csiId)
            .orElseThrow(() -> new CsiNotFoundException("CSI not found: " + csiId));
        
        ClmReport report = new ClmReport();
        report.setReportId(UUID.randomUUID().toString());
        report.setCsiId(csiId);
        report.setGeneratedAt(LocalDateTime.now());
        report.setReportType(ReportType.ADHOC);
        report.setFormat(format);
        
        // Get latest CLM results
        ClmResults clmResults = getLatestClmResults(csiId);
        report.setClmResults(clmResults);
        
        // Generate inventory from results
        if (clmResults != null) {
            CertificateInventory inventory = generateInventory(clmResults);
            clmResults.setInventory(inventory);
        }
        
        // Get action items
        List<ActionItem> actionItems = actionItemRepository.findByCsiIdAndStatus(
            csiId, ActionStatus.OPEN);
        report.setActionItems(actionItems);
        
        // Set retention
        report.setExpiresAt(LocalDateTime.now().plusMonths(3));
        
        return reportRepository.save(report);
    }
    
    /**
     * Gets report by job ID
     */
    public ClmReport getReportByJobId(String jobId, ReportFormat format) {
        ClmReport report = reportRepository.findByJobId(jobId)
            .orElseThrow(() -> new ReportNotFoundException("Report not found for job: " + jobId));
        
        // Convert format if needed
        if (format != report.getFormat()) {
            report = convertReportFormat(report, format);
        }
        
        return report;
    }
    
    /**
     * Gets latest report for a CSI
     */
    public ClmReport getLatestReport(String csiId, ReportFormat format) {
        ClmReport report = reportRepository
            .findFirstByCsiIdOrderByGeneratedAtDesc(csiId)
            .orElseThrow(() -> new ReportNotFoundException(
                "No reports found for CSI: " + csiId));
        
        if (format != report.getFormat()) {
            report = convertReportFormat(report, format);
        }
        
        return report;
    }
    
    /**
     * Searches reports based on criteria
     */
    public List<ClmReport> searchReports(String csiId, LocalDateTime startDate, 
                                       LocalDateTime endDate, ReportType reportType) {
        log.info("Searching reports - CSI: {}, Start: {}, End: {}, Type: {}", 
                csiId, startDate, endDate, reportType);
        
        // Build query based on provided criteria
        if (csiId != null && reportType != null && startDate != null && endDate != null) {
            return reportRepository.findByCsiIdAndReportTypeAndGeneratedAtBetween(
                csiId, reportType, startDate, endDate);
        } else if (csiId != null && startDate != null && endDate != null) {
            return reportRepository.findByCsiIdAndGeneratedAtBetween(
                csiId, startDate, endDate);
        } else if (csiId != null && reportType != null) {
            return reportRepository.findByCsiIdAndReportType(csiId, reportType);
        } else if (csiId != null) {
            return reportRepository.findByCsiId(csiId);
        } else {
            return reportRepository.findByGeneratedAtBetween(
                startDate != null ? startDate : LocalDateTime.now().minusMonths(3),
                endDate != null ? endDate : LocalDateTime.now()
            );
        }
    }
    
    /**
     * Converts report summaries for API response
     */
    public List<ReportSummary> convertToReportSummaries(List<ClmReport> reports) {
        return reports.stream()
            .map(this::createReportSummary)
            .collect(Collectors.toList());
    }
    
    /**
     * Generates adhoc report
     */
    public ClmReport generateAdhocReport(ClmJob job) {
        log.info("Generating adhoc report for job: {}", job.getJobId());
        
        ClmReport report = new ClmReport();
        report.setReportId(UUID.randomUUID().toString());
        report.setJobId(job.getJobId());
        report.setCsiId(job.getCsiId());
        report.setGeneratedAt(LocalDateTime.now());
        report.setReportType(ReportType.ADHOC);
        report.setFormat(ReportFormat.JSON);
        
        // Get module results from job parameters
        List<ModuleExecutionResult> moduleResults = 
            (List<ModuleExecutionResult>) job.getJobParameters().get("moduleResults");
        
        if (moduleResults != null) {
            processModuleResults(report, moduleResults);
        }
        
        // Consolidate action items
        report.setActionItems(consolidateActionItems(report));
        
        // Set retention
        report.setExpiresAt(LocalDateTime.now().plusMonths(3));
        
        return reportRepository.save(report);
    }
    
    /**
     * Deletes expired reports
     */
    public void deleteExpiredReports() {
        log.info("Deleting expired reports");
        List<ClmReport> expiredReports = reportRepository
            .findByExpiresAtBefore(LocalDateTime.now());
        
        log.info("Found {} expired reports to delete", expiredReports.size());
        reportRepository.deleteAll(expiredReports);
    }
    
    private ReportSummary createReportSummary(ClmReport report) {
        ReportSummary summary = new ReportSummary();
        summary.setReportId(report.getReportId());
        summary.setCsiId(report.getCsiId());
        summary.setReportType(report.getReportType());
        summary.setGeneratedAt(report.getGeneratedAt());
        summary.setJobId(report.getJobId());
        
        // Add summary statistics
        if (report.getClmResults() != null && report.getClmResults().getInventory() != null) {
            summary.setTotalCertificates(
                report.getClmResults().getInventory().getTotalCertificates());
        }
        
        summary.setActionItemCount(
            report.getActionItems() != null ? report.getActionItems().size() : 0);
        
        summary.setCriticalItemCount(
            report.getActionItems() != null ? 
                (int) report.getActionItems().stream()
                    .filter(item -> item.getPriority() == Priority.CRITICAL)
                    .count() : 0);
        
        return summary;
    }
    
    private ClmReport convertReportFormat(ClmReport report, ReportFormat targetFormat) {
        if (targetFormat == ReportFormat.HTML && report.getFormat() == ReportFormat.JSON) {
            // Convert JSON to HTML using template service
            // This would be implemented with template engine
            log.info("Converting report {} from JSON to HTML", report.getReportId());
        }
        return report;
    }
    
    private void processModuleResults(ClmReport report, List<ModuleExecutionResult> results) {
        for (ModuleExecutionResult result : results) {
            switch (result.getModule()) {
                case CLM:
                    report.setClmResults((ClmResults) result.getResults());
                    break;
                case BCM:
                    report.setBcmResults((BcmResults) result.getResults());
                    break;
                case EOVS:
                    report.setEovsResults((EovsResults) result.getResults());
                    break;
            }
        }
    }
    
    private List<ActionItem> consolidateActionItems(ClmReport report) {
        List<ActionItem> items = new ArrayList<>();
        
        // Add items from each module
        if (report.getClmResults() != null) {
            items.addAll(generateClmActionItems(report.getClmResults()));
        }
        if (report.getBcmResults() != null) {
            items.addAll(generateBcmActionItems(report.getBcmResults()));
        }
        if (report.getEovsResults() != null) {
            items.addAll(generateEovsActionItems(report.getEovsResults()));
        }
        
        return items;
    }
    
    private CertificateInventory generateInventory(ClmResults clmResults) {
        CertificateInventory inventory = new CertificateInventory();
        
        // Combine VM and container certificates
        List<CertificateDetail> allCerts = new ArrayList<>();
        
        if (clmResults.getVmDiscoveryResults() != null) {
            allCerts.addAll(extractCertificates(clmResults.getVmDiscoveryResults()));
        }
        
        if (clmResults.getContainerDiscoveryResults() != null) {
            allCerts.addAll(extractCertificates(clmResults.getContainerDiscoveryResults()));
        }
        
        inventory.setTotalCertificates(allCerts.size());
        inventory.setCertificates(allCerts);
        
        // Categorize by expiry
        Map<String, Integer> expiryCategories = new HashMap<>();
        expiryCategories.put("EXPIRED", 0);
        expiryCategories.put("30_DAYS", 0);
        expiryCategories.put("60_DAYS", 0);
        expiryCategories.put("90_DAYS", 0);
        expiryCategories.put("VALID", 0);
        
        for (CertificateDetail cert : allCerts) {
            String category = categorizeCertificate(cert);
            expiryCategories.merge(category, 1, Integer::sum);
        }
        
        inventory.setCertsByExpiryCategory(expiryCategories);
        
        return inventory;
    }
}


package com.citi.cert_management.container.service;

import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * Service for testing ansible connectivity to servers
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class ConnectionTestService {
    
    private final ServerRepository serverRepository;
    private final AnsibleService ansibleService;
    private final ExecutorService executorService = Executors.newFixedThreadPool(10);
    
    /**
     * Tests ansible connections for all servers of a CSI
     */
    public CompletableFuture<ConnectionTestResult> testServerConnections(String csiId) {
        log.info("Testing server connections for CSI: {}", csiId);
        
        return CompletableFuture.supplyAsync(() -> {
            List<Server> servers = serverRepository.findByCsiId(csiId);
            ConnectionTestResult result = new ConnectionTestResult();
            result.setTotalServers(servers.size());
            
            if (servers.isEmpty()) {
                log.warn("No servers found for CSI: {}", csiId);
                return result;
            }
            
            // Test each server in parallel
            List<CompletableFuture<ServerConnectionResult>> futures = new ArrayList<>();
            
            for (Server server : servers) {
                CompletableFuture<ServerConnectionResult> future = 
                    testSingleServerConnection(server);
                futures.add(future);
            }
            
            // Wait for all tests to complete
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
            
            // Process results
            int successCount = 0;
            int failedCount = 0;
            List<Server> connectedServers = new ArrayList<>();
            
            for (int i = 0; i < servers.size(); i++) {
                ServerConnectionResult connResult = futures.get(i).join();
                Server server = servers.get(i);
                
                server.setConnectionStatus(connResult.getStatus());
                server.setLastConnectionTest(LocalDateTime.now());
                
                if (connResult.getStatus() == ConnectionStatus.SUCCESS) {
                    successCount++;
                    connectedServers.add(server);
                } else {
                    failedCount++;
                    log.warn("Connection failed for server: {} - {}", 
                            server.getServerId(), connResult.getErrorMessage());
                }
            }
            
            // Save updated connection status
            serverRepository.saveAll(servers);
            
            result.setSuccessfulConnections(successCount);
            result.setFailedConnections(failedCount);
            result.setServers(servers);
            
            log.info("Connection test completed for CSI: {} - Success: {}, Failed: {}", 
                    csiId, successCount, failedCount);
            
            return result;
        }, executorService);
    }
    
    /**
     * Tests connection to a single server
     */
    private CompletableFuture<ServerConnectionResult> testSingleServerConnection(Server server) {
        return CompletableFuture.supplyAsync(() -> {
            ServerConnectionResult result = new ServerConnectionResult();
            result.setServerId(server.getServerId());
            
            try {
                log.debug("Testing connection to server: {} ({})", 
                         server.getServerId(), server.getIpAddress());
                
                // Test ansible connectivity
                AnsibleResponse response = ansibleService.testConnection(
                    server.getIpAddress(),
                    server.getAnsibleCredentials()
                );
                
                if (response.isSuccess()) {
                    result.setStatus(ConnectionStatus.SUCCESS);
                    result.setResponseTime(response.getExecutionTime());
                } else {
                    result.setStatus(ConnectionStatus.FAILED);
                    result.setErrorMessage(response.getErrorMessage());
                }
                
            } catch (Exception e) {
                log.error("Connection test failed for server: {}", server.getServerId(), e);
                result.setStatus(ConnectionStatus.FAILED);
                result.setErrorMessage("Connection test error: " + e.getMessage());
            }
            
            return result;
        }, executorService);
    }
    
    /**
     * Retests failed connections
     */
    public CompletableFuture<ConnectionTestResult> retestFailedConnections(String csiId) {
        log.info("Retesting failed connections for CSI: {}", csiId);
        
        return CompletableFuture.supplyAsync(() -> {
            List<Server> failedServers = serverRepository
                .findByCsiIdAndConnectionStatus(csiId, ConnectionStatus.FAILED);
            
            if (failedServers.isEmpty()) {
                log.info("No failed connections to retest for CSI: {}", csiId);
                ConnectionTestResult result = new ConnectionTestResult();
                result.setTotalServers(0);
                return result;
            }
            
            log.info("Retesting {} failed connections", failedServers.size());
            
            // Retest using the same logic
            return testServerConnections(csiId).join();
        });
    }
    
    @Data
    private static class ServerConnectionResult {
        private String serverId;
        private ConnectionStatus status;
        private String errorMessage;
        private Long responseTime;
    }
}



package com.citi.cert_management.container.service;

import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Service for generating various types of reports
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class ReportGenerationService {
    
    private final ClmReportRepository reportRepository;
    private final ClmJobRepository jobRepository;
    private final ActionItemRepository actionItemRepository;
    private final CsiMetadataRepository csiRepository;
    
    /**
     * Generates onboarding report after initial scan
     */
    public ClmReport generateOnboardingReport(ClmJob masterJob) {
        log.info("Generating onboarding report for job: {}", masterJob.getJobId());
        
        ClmReport report = new ClmReport();
        report.setReportId(UUID.randomUUID().toString());
        report.setJobId(masterJob.getJobId());
        report.setCsiId(masterJob.getCsiId());
        report.setGeneratedAt(LocalDateTime.now());
        report.setReportType(ReportType.ONBOARDING);
        report.setFormat(ReportFormat.JSON);
        
        // Extract module results from job
        List<ModuleExecutionResult> moduleResults = 
            (List<ModuleExecutionResult>) masterJob.getJobParameters().get("moduleResults");
        
        if (moduleResults != null) {
            processModuleResults(report, moduleResults);
        }
        
        // Generate consolidated action items
        List<ActionItem> actionItems = generateConsolidatedActionItems(report, masterJob.getCsiId());
        report.setActionItems(actionItems);
        
        // Save action items
        if (!actionItems.isEmpty()) {
            actionItemRepository.saveAll(actionItems);
        }
        
        // Set retention period
        report.setExpiresAt(LocalDateTime.now().plusMonths(3));
        
        return reportRepository.save(report);
    }
    
    /**
     * Generates weekly batch report
     */
    public ClmReport generateBatchReport(String csiId, List<ModuleExecutionResult> moduleResults) {
        log.info("Generating batch report for CSI: {}", csiId);
        
        ClmReport report = new ClmReport();
        report.setReportId(UUID.randomUUID().toString());
        report.setCsiId(csiId);
        report.setGeneratedAt(LocalDateTime.now());
        report.setReportType(ReportType.WEEKLY);
        report.setFormat(ReportFormat.JSON);
        
        processModuleResults(report, moduleResults);
        
        // Compare with previous week's report
        Optional<ClmReport> previousReport = reportRepository
            .findFirstByCsiIdAndReportTypeOrderByGeneratedAtDesc(csiId, ReportType.WEEKLY);
        
        if (previousReport.isPresent()) {
            addComparison(report, previousReport.get());
        }
        
        // Generate action items
        List<ActionItem> actionItems = generateConsolidatedActionItems(report, csiId);
        report.setActionItems(actionItems);
        
        // Update existing action items
        updateExistingActionItems(csiId, actionItems);
        
        report.setExpiresAt(LocalDateTime.now().plusMonths(3));
        
        return reportRepository.save(report);
    }
    
    /**
     * Generates renewal report
     */
    public ClmReport generateRenewalReport(String csiId, RenewalResult renewalResult) {
        log.info("Generating renewal report for CSI: {}", csiId);
        
        ClmReport report = new ClmReport();
        report.setReportId(UUID.randomUUID().toString());
        report.setCsiId(csiId);
        report.setGeneratedAt(LocalDateTime.now());
        report.setReportType(ReportType.RENEWAL);
        report.setFormat(ReportFormat.JSON);
        
        // Add renewal specific data
        Map<String, Object> renewalData = new HashMap<>();
        renewalData.put("renewalId", renewalResult.getRenewalId());
        renewalData.put("certificatesRenewed", renewalResult.getCertificatesRenewed());
        renewalData.put("status", renewalResult.getStatus());
        renewalData.put("details", renewalResult.getDetails());
        
        report.setRawData(renewalData);
        
        report.setExpiresAt(LocalDateTime.now().plusMonths(3));
        
        return reportRepository.save(report);
    }
    
    private void processModuleResults(ClmReport report, List<ModuleExecutionResult> results) {
        for (ModuleExecutionResult result : results) {
            switch (result.getModule()) {
                case CLM:
                    report.setClmResults((ClmResults) result.getResults());
                    break;
                case BCM:
                    report.setBcmResults((BcmResults) result.getResults());
                    break;
                case EOVS:
                    report.setEovsResults((EovsResults) result.getResults());
                    break;
            }
        }
    }
    
    private List<ActionItem> generateConsolidatedActionItems(ClmReport report, String csiId) {
        List<ActionItem> items = new ArrayList<>();
        
        // Generate CLM action items
        if (report.getClmResults() != null) {
            items.addAll(generateClmActionItems(report.getClmResults(), csiId));
        }
        
        // Generate BCM action items
        if (report.getBcmResults() != null) {
            items.addAll(generateBcmActionItems(report.getBcmResults(), csiId));
        }
        
        // Generate EOVS action items
        if (report.getEovsResults() != null) {
            items.addAll(generateEovsActionItems(report.getEovsResults(), csiId));
        }
        
        return items;
    }
    
    private List<ActionItem> generateClmActionItems(ClmResults results, String csiId) {
        List<ActionItem> items = new ArrayList<>();
        
        if (results.getInventory() != null) {
            CertificateInventory inventory = results.getInventory();
            
            // Critical: Expired certificates
            Integer expired = inventory.getCertsByExpiryCategory().get("EXPIRED");
            if (expired != null && expired > 0) {
                ActionItem item = new ActionItem();
                item.setActionId(UUID.randomUUID().toString());
                item.setCsiId(csiId);
                item.setType(ActionType.CERTIFICATE_EXPIRING);
                item.setPriority(Priority.CRITICAL);
                item.setDescription(expired + " certificates have already expired and need immediate renewal");
                item.setDueDate(LocalDateTime.now());
                item.setStatus(ActionStatus.OPEN);
                item.setCreatedAt(LocalDateTime.now());
                items.add(item);
            }
            
            // High: Expiring in 30 days
            Integer expiring30 = inventory.getCertsByExpiryCategory().get("30_DAYS");
            if (expiring30 != null && expiring30 > 0) {
                ActionItem item = new ActionItem();
                item.setActionId(UUID.randomUUID().toString());
                item.setCsiId(csiId);
                item.setType(ActionType.CERTIFICATE_EXPIRING);
                item.setPriority(Priority.HIGH);
                item.setDescription(expiring30 + " certificates expiring within 30 days");
                item.setDueDate(LocalDateTime.now().plusDays(7));
                item.setStatus(ActionStatus.OPEN);
                item.setCreatedAt(LocalDateTime.now());
                items.add(item);
            }
        }
        
        // Connection failures
        if (results.getVmDiscoveryResults() != null && 
            !results.getVmDiscoveryResults().getScanErrors().isEmpty()) {
            ActionItem item = new ActionItem();
            item.setActionId(UUID.randomUUID().toString());
            item.setCsiId(csiId);
            item.setType(ActionType.CONNECTION_FAILED);
            item.setPriority(Priority.MEDIUM);
            item.setDescription("Failed to scan " + 
                results.getVmDiscoveryResults().getScanErrors().size() + " servers");
            item.setDueDate(LocalDateTime.now().plusDays(3));
            item.setStatus(ActionStatus.OPEN);
            item.setCreatedAt(LocalDateTime.now());
            items.add(item);
        }
        
        return items;
    }
    
    private List<ActionItem> generateBcmActionItems(BcmResults results, String csiId) {
        List<ActionItem> items = new ArrayList<>();
        
        if (results.getExceptionsFound() > 0) {
            // Group by severity
            Map<Priority, Integer> exceptionsBySeverity = categorizeExceptions(results.getExceptions());
            
            for (Map.Entry<Priority, Integer> entry : exceptionsBySeverity.entrySet()) {
                if (entry.getValue() > 0) {
                    ActionItem item = new ActionItem();
                    item.setActionId(UUID.randomUUID().toString());
                    item.setCsiId(csiId);
                    item.setType(ActionType.BCM_EXCEPTION);
                    item.setPriority(entry.getKey());
                    item.setDescription(entry.getValue() + " BCM exceptions require attention");
                    item.setDueDate(calculateDueDate(entry.getKey()));
                    item.setStatus(ActionStatus.OPEN);
                    item.setCreatedAt(LocalDateTime.now());
                    items.add(item);
                }
            }
        }
        
        return items;
    }
    
    private List<ActionItem> generateEovsActionItems(EovsResults results, String csiId) {
        List<ActionItem> items = new ArrayList<>();
        
        if (results.getPendingAcknowledgment() > 0) {
            ActionItem item = new ActionItem();
            item.setActionId(UUID.randomUUID().toString());
            item.setCsiId(csiId);
            item.setType(ActionType.EOVS_ACKNOWLEDGMENT_REQUIRED);
            item.setPriority(Priority.HIGH);
            item.setDescription(results.getPendingAcknowledgment() + 
                " EOVS vulnerabilities require user acknowledgment");
            item.setDueDate(LocalDateTime.now().plusDays(5));
            item.setStatus(ActionStatus.OPEN);
            item.setCreatedAt(LocalDateTime.now());
            items.add(item);
        }
        
        return items;
    }
    
    private void updateExistingActionItems(String csiId, List<ActionItem> newItems) {
        // Get existing open action items
        List<ActionItem> existingItems = actionItemRepository
            .findByCsiIdAndStatus(csiId, ActionStatus.OPEN);
        
        // Check if any can be closed
        for (ActionItem existing : existingItems) {
            boolean stillValid = newItems.stream()
                .anyMatch(item -> item.getType() == existing.getType() && 
                         item.getDescription().equals(existing.getDescription()));
            
            if (!stillValid) {
                existing.setStatus(ActionStatus.RESOLVED);
                existing.setResolvedAt(LocalDateTime.now());
                actionItemRepository.save(existing);
            }
        }
    }
    
    private LocalDateTime calculateDueDate(Priority priority) {
        switch (priority) {
            case CRITICAL:
                return LocalDateTime.now().plusDays(1);
            case HIGH:
                return LocalDateTime.now().plusDays(3);
            case MEDIUM:
                return LocalDateTime.now().plusDays(7);
            case LOW:
                return LocalDateTime.now().plusDays(14);
            default:
                return LocalDateTime.now().plusDays(7);
        }
    }
}



package com.citi.cert_management.container.dto;

import lombok.Data;
import lombok.Builder;

/**
 * Summary of discovery operations
 */
@Data
@Builder
public class DiscoverySummary {
    private String discoveryType;
    private int totalServersScanned;
    private int serversWithCertificates;
    private int totalCertificatesFound;
    private int scanErrors;
    private double successRate;
    private LocalDateTime executedAt;
    private long executionTimeMs;
}

package com.citi.cert_management.container.dto;

import lombok.Data;
import lombok.Builder;
import java.util.Map;

/**
 * Summary of certificate inventory
 */
@Data
@Builder
public class CertificateInventorySummary {
    private int totalCertificates;
    private Map<String, Integer> certificatesByType;
    private Map<String, Integer> certificatesByExpiry;
    private int expiredCount;
    private int expiringIn30Days;
    private int expiringIn60Days;
    private int validCount;
    private int autoRenewalEligible;
}


package com.citi.cert_management.container.dto;

import lombok.Data;
import lombok.Builder;
import java.time.LocalDateTime;

/**
 * Summary view of a report for list displays
 */
@Data
@Builder
public class ReportSummary {
    private String reportId;
    private String csiId;
    private ReportType reportType;
    private LocalDateTime generatedAt;
    private String jobId;
    private int totalCertificates;
    private int actionItemCount;
    private int criticalItemCount;
    private boolean emailSent;
    private LocalDateTime emailSentAt;
}

package com.citi.cert_management.container.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import lombok.Data;
import java.time.LocalDateTime;

/**
 * Action item requiring attention
 */
@Data
@Document(collection = "action_items")
public class ActionItem {
    @Id
    private String actionId;
    private String csiId;
    private ActionType type;
    private Priority priority;
    private String description;
    private String targetCertificate;
    private String targetServer;
    private LocalDateTime dueDate;
    private ActionStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime resolvedAt;
    private String resolvedBy;
    private boolean escalated;
    private LocalDateTime escalatedAt;
    private String assignedTo;
    private String notes;
}

package com.citi.cert_management.container.dto;

import lombok.Data;
import lombok.Builder;

/**
 * Summary view of action items
 */
@Data
@Builder
public class ActionItemSummary {
    private String actionId;
    private ActionType type;
    private Priority priority;
    private String shortDescription;
    private int daysOverdue;
    private boolean escalated;
}

package com.citi.cert_management.container.dto;

import lombok.Data;
import javax.validation.constraints.NotNull;
import java.util.Map;

/**
 * Request to execute module-specific operation
 */
@Data
public class ModuleOperationRequest {
    @NotNull
    private String operation;
    
    private Map<String, Object> parameters;
    private boolean forceExecution;
    private String reason;
}

package com.citi.cert_management.container.dto;

import lombok.Data;
import lombok.AllArgsConstructor;
import lombok.Builder;

/**
 * Response for job initiation requests
 */
@Data
@AllArgsConstructor
@Builder
public class JobResponse {
    private String jobId;
    private String message;
    private JobStatus status;
    private LocalDateTime estimatedCompletion;
}

package com.citi.cert_management.container.dto;

import lombok.Data;
import lombok.Builder;

/**
 * Response for remediation requests
 */
@Data
@Builder
public class RemediationResponse {
    private boolean success;
    private String remediationId;
    private String message;
    private int itemsRemediated;
    private int itemsFailed;
    private LocalDateTime completedAt;
}

package com.citi.cert_management.container.model;

import lombok.Data;
import lombok.Builder;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

/**
 * Result of remediation operation
 */
@Data
@Builder
public class RemediationResult {
    private String remediationId;
    private boolean success;
    private String status;
    private String message;
    private String moduleType;
    private List<String> certificatesRenewed;
    private Map<String, Object> details;
    private int totalItems;
    private int successfulItems;
    private int failedItems;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private List<RemediationError> errors;
}

@Data
public class RemediationError {
    private String itemId;
    private String errorCode;
    private String errorMessage;
    private LocalDateTime occurredAt;
}

package com.citi.cert_management.container.model;

import lombok.Data;
import java.util.List;
import java.util.Map;

/**
 * Certificate impact assessment results
 */
@Data
public class ImpactAssessment {
    private String assessmentId;
    private String csiId;
    private LocalDateTime assessedAt;
    
    // Impact analysis
    private int totalCertificatesAnalyzed;
    private int criticalCertificates;
    private int highImpactCertificates;
    private int mediumImpactCertificates;
    private int lowImpactCertificates;
    
    // Dependencies
    private Map<String, List<String>> certificateDependencies;
    private List<ServiceImpact> impactedServices;
    private List<ApplicationImpact> impactedApplications;
    
    // Risk factors
    private List<String> identifiedRisks;
    private OverallRiskLevel overallRisk;
    
    // Recommendations
    private List<String> recommendations;
    private List<String> immediateActions;
}

@Data
public class ServiceImpact {
    private String serviceName;
    private String certificateId;
    private ImpactLevel impactLevel;
    private String description;
    private List<String> affectedEndpoints;
}

@Data
public class ApplicationImpact {
    private String applicationName;
    private String applicationId;
    private List<String> affectedCertificates;
    private ImpactLevel impactLevel;
    private int usersAffected;
}

public enum ImpactLevel {
    CRITICAL, HIGH, MEDIUM, LOW
}

public enum OverallRiskLevel {
    VERY_HIGH, HIGH, MODERATE, LOW, MINIMAL
}

package com.citi.cert_management.container.model;

import lombok.Data;
import java.util.List;
import java.util.Map;

/**
 * Risk assessment for certificates and configurations
 */
@Data
public class RiskAssessment {
    private String assessmentId;
    private String csiId;
    private LocalDateTime assessedAt;
    
    // Certificate risks
    private List<CertificateRisk> certificateRisks;
    
    // Configuration risks
    private List<ConfigurationRisk> configurationRisks;
    
    // Compliance risks
    private List<ComplianceRisk> complianceRisks;
    
    // Overall assessment
    private RiskScore overallRiskScore;
    private String riskSummary;
    private List<RiskMitigation> recommendedMitigations;
}

@Data
public class CertificateRisk {
    private String certificateId;
    private String riskType;
    private RiskLevel level;
    private String description;
    private LocalDateTime identifiedAt;
    private boolean mitigated;
}

@Data
public class ConfigurationRisk {
    private String serverId;
    private String configType;
    private String finding;
    private RiskLevel level;
    private String recommendation;
}

@Data
public class ComplianceRisk {
    private String standard;
    private String requirement;
    private boolean compliant;
    private String gap;
    private String remediation;
}

@Data
public class RiskScore {
    private int score;
    private RiskLevel level;
    private Map<String, Integer> categoryScores;
}

@Data
public class RiskMitigation {
    private String mitigationId;
    private String description;
    private Priority priority;
    private String estimatedEffort;
    private List<String> steps;
}

public enum RiskLevel {
    CRITICAL, HIGH, MEDIUM, LOW, MINIMAL
}

package com.citi.cert_management.container.model;

import lombok.Data;
import lombok.Builder;
import java.time.LocalDateTime;

/**
 * Result of synchronization operation
 */
@Data
@Builder
public class SyncResult {
    private boolean success;
    private int recordsProcessed;
    private int recordsFailed;
    private int recordsSkipped;
    private String errorMessage;
    private String syncType;
    private LocalDateTime startedAt;
    private LocalDateTime completedAt;
    private long durationMs;
    private Map<String, Object> metadata;
}

package com.citi.cert_management.container.model;

import lombok.Data;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

/**
 * Result of repository scanning operation
 */
@Data
public class RepositoryScanResult {
    private int repositoriesFound;
    private int repositoriesScanned;
    private int repositoriesFailed;
    private List<Repository> repositories;
    private List<CertificateLocation> certificateLocations;
    private LocalDateTime scanCompletedAt;
    private long scanDurationMs;
    private Map<String, Integer> certificatesByRepository;
    private List<ScanError> errors;
}

@Data
public class Repository {
    private String repositoryId;
    private String name;
    private String cloneUrl;
    private String branch;
    private RepositoryType type;
    private LocalDateTime lastScanned;
    private int certificatesFound;
}

@Data
public class ScanError {
    private String repository;
    private String errorType;
    private String errorMessage;
    private LocalDateTime occurredAt;
}

public enum RepositoryType {
    BITBUCKET, GITHUB, GITLAB
}

package com.citi.cert_management.container.repository;

import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;
import java.util.List;

/**
 * Repository for CSI metadata operations
 */
@Repository
public interface CsiMetadataRepository extends MongoRepository<CsiDetails, String> {
    
    Optional<CsiDetails> findByCsiId(String csiId);
    
    boolean existsByCsiId(String csiId);
    
    List<CsiDetails> findByActiveTrue();
    
    List<CsiDetails> findByActiveTrueAndModuleSubscriptions_Module(ModuleType module);
    
    List<CsiDetails> findByRequestorSoeid(String soeid);
    
    List<CsiDetails> findByEnvironment(String environment);
    
    List<CsiDetails> findByOnboardedAtBetween(LocalDateTime start, LocalDateTime end);
}

package com.citi.cert_management.container.repository;

import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

/**
 * Repository for CLM job operations
 */
@Repository
public interface ClmJobRepository extends MongoRepository<ClmJob, String> {
    
    Optional<ClmJob> findByJobId(String jobId);
    
    List<ClmJob> findByCsiId(String csiId);
    
    List<ClmJob> findByCsiIdAndStatus(String csiId, JobStatus status);
    
    List<ClmJob> findByStatusIn(List<JobStatus> statuses);
    
    List<ClmJob> findByJobTypeAndStatusAndStartTimeBetween(
        JobType jobType, JobStatus status, LocalDateTime start, LocalDateTime end);
    
    List<ClmJob> findByTriggeredBy(String triggeredBy);
    
    List<ClmJob> findByModuleType(ModuleType moduleType);
    
    List<ClmJob> findByStatusAndStartTimeBefore(JobStatus status, LocalDateTime cutoff);
}

package com.citi.cert_management.container.exception;

/**
 * Base exception for CLM system
 */
public class ClmException extends RuntimeException {
    public ClmException(String message) {
        super(message);
    }
    
    public ClmException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * CSI related exceptions
 */
public class CsiNotFoundException extends ClmException {
    public CsiNotFoundException(String message) {
        super(message);
    }
}

public class CsiAlreadyExistsException extends ClmException {
    public CsiAlreadyExistsException(String message) {
        super(message);
    }
}

/**
 * Module related exceptions
 */
public class ModuleNotSubscribedException extends ClmException {
    public ModuleNotSubscribedException(String message) {
        super(message);
    }
}

public class ModuleExecutionException extends ClmException {
    public ModuleExecutionException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Job related exceptions
 */
public class JobNotFoundException extends ClmException {
    public JobNotFoundException(String message) {
        super(message);
    }
}

public class JobExecutionException extends ClmException {
    public JobExecutionException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Sync related exceptions
 */
public class SyncException extends ClmException {
    public SyncException(String message) {
        super(message);
    }
    
    public SyncException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Discovery related exceptions
 */
public class DiscoveryException extends ClmException {
    public DiscoveryException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Report related exceptions
 */
public class ReportNotFoundException extends ClmException {
    public ReportNotFoundException(String message) {
        super(message);
    }
}

public class ReportGenerationException extends ClmException {
    public ReportGenerationException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Certificate related exceptions
 */
public class CertificateNotFoundException extends ClmException {
    public CertificateNotFoundException(String message) {
        super(message);
    }
}

public class CertificateRenewalException extends ClmException {
    public CertificateRenewalException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Assessment related exceptions
 */
public class AssessmentException extends ClmException {
    public AssessmentException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Remediation related exceptions
 */
public class RemediationException extends ClmException {
    public RemediationException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Notification related exceptions
 */
public class NotificationException extends ClmException {
    public NotificationException(String message) {
        super(message);
    }
    
    public NotificationException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Validation exception
 */
public class ValidationException extends ClmException {
    public ValidationException(String message) {
        super(message);
    }
}

/**
 * External service exceptions
 */
public class ExternalServiceException extends ClmException {
    private String serviceName;
    
    public ExternalServiceException(String serviceName, String message, Throwable cause) {
        super(message, cause);
        this.serviceName = serviceName;
    }
    
    public String getServiceName() {
        return serviceName;
    }
}

// In EnhancedJobOrchestrationService or as a utility method

/**
 * Checks if all sync operations are complete and CSI can proceed with discovery
 */
private boolean canProceedWithDiscovery(CsiDetails csi) {
    return csi.getServerSyncStatus() != null && 
           csi.getServerSyncStatus().getState() == SyncState.COMPLETED &&
           csi.getCertificateSyncStatus() != null && 
           csi.getCertificateSyncStatus().getState() == SyncState.COMPLETED &&
           csi.getRepositorySyncStatus() != null && 
           csi.getRepositorySyncStatus().getState() == SyncState.COMPLETED;
}

// In CsiService or as a utility method

/**
 * Gets module subscription for a specific module type
 */
public ModuleSubscription getModuleSubscription(CsiDetails csi, ModuleType moduleType) {
    return csi.getModuleSubscriptions().stream()
        .filter(subscription -> subscription.getModule() == moduleType)
        .findFirst()
        .orElse(null);
}

package com.citi.cert_management.container.executor;

/**
 * Interface for module-specific execution logic
 */
public interface ModuleExecutor {
    /**
     * Executes module-specific operations
     * 
     * @param csi CSI details
     * @param servers List of connected servers
     * @param subscription Module subscription with configuration
     * @return Execution result
     */
    ModuleExecutionResult execute(CsiDetails csi, List<Server> servers, 
                                 ModuleSubscription subscription);
    
    /**
     * Gets the module type this executor handles
     * 
     * @return Module type
     */
    ModuleType getModuleType();
    
    /**
     * Validates if the module can be executed
     * 
     * @param csi CSI details
     * @return true if can execute, false otherwise
     */
    default boolean canExecute(CsiDetails csi) {
        ModuleSubscription subscription = csi.getModuleSubscriptions().stream()
            .filter(s -> s.getModule() == getModuleType())
            .findFirst()
            .orElse(null);
        
        return subscription != null && subscription.isSelected();
    }
}


