1. Enhanced Domain Models with Module Support
javapackage com.citi.cert_management.container.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import lombok.Data;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

/**
 * Enhanced CSI Details with module subscription support
 */
@Data
@Document(collection = "csi_details")
public class CsiDetails {
    @Id
    private String id;
    
    // Basic CSI Information
    private String csiId;
    private String csiName;
    private String environment;
    private String cmpRequestersOeId;
    private String ownerTeamEmailDlName;
    private String email;
    private String escalationContacts;
    private String primaryContact;
    private String preferredRenewalProvider;
    private boolean autoRenewalEnabled;
    private String customExpiryThresholdDays;
    private String pidUsername;
    private String pidPassword;
    private String repoType;
    private List<String> repositories;
    private String projects;
    private List<String> notificationEmails;
    private boolean dailyDigestEnabled;
    private boolean criticalAlertsOnly;
    private LocalDateTime onboardedAt;
    private String onboardedBy;
    private Date lastModified;
    private boolean active;
    private String appName;
    
    // User mandatory inputs
    private String requestorSoeid;
    private String emailDl;
    private String preferredRenewal;
    private String bitbucketProjectUrl;
    private String githubProjectUrl;
    private List<String> notificationEmailList;
    private List<String> escalationMatrix;
    
    // Module Subscriptions
    private List<ModuleSubscription> moduleSubscriptions;
    
    // Sync Status
    private SyncStatus serverSyncStatus;
    private SyncStatus certificateSyncStatus;
    private SyncStatus repositorySyncStatus;
    private LocalDateTime lastSuccessfulSync;
    
    // Job Tracking
    private String lastJobId;
    private LocalDateTime lastScanDate;
    private LocalDateTime nextScheduledScan;
    private JobStatus lastJobStatus;
}

/**
 * Module subscription configuration
 */
@Data
public class ModuleSubscription {
    private ModuleType module;
    private boolean isSelected;
    private ModuleConfiguration configuration;
    private LocalDateTime subscribedAt;
    private String subscribedBy;
}

/**
 * Module-specific configuration
 */
@Data
public class ModuleConfiguration {
    // CLM Configuration
    private boolean vmDiscoveryEnabled;
    private boolean containerDiscoveryEnabled;
    private boolean autoDeploymentEnabled;
    
    // BCM Configuration
    private BcmMode bcmMode;
    private List<String> bcmExceptionTypes;
    private boolean bcmAutoRemediationEnabled;
    
    // EOVS Configuration
    private EovsMode eovsMode;
    private boolean eovsUserAcknowledgmentRequired;
    private List<String> eovsChecks;
}

/**
 * Sync status tracking
 */
@Data
public class SyncStatus {
    private SyncState state;
    private LocalDateTime lastAttempt;
    private LocalDateTime lastSuccess;
    private String errorMessage;
    private int recordsProcessed;
    private int recordsFailed;
}

/**
 * Server information from drift
 */
@Data
@Document(collection = "servers")
public class Server {
    @Id
    private String id;
    private String serverId;
    private String serverName;
    private String csiId;
    private String ipAddress;
    private String environment;
    private String operatingSystem;
    private boolean ansibleEnabled;
    private ConnectionStatus connectionStatus;
    private LocalDateTime lastConnectionTest;
    private LocalDateTime syncedFromDrift;
    private Map<String, Object> metadata;
}

/**
 * Enhanced CLM Job with module support
 */
@Data
@Document(collection = "clm_jobs")
public class ClmJob {
    @Id
    private String jobId;
    private String csiId;
    private JobType jobType;
    private ModuleType moduleType;
    private JobStatus status;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private String triggeredBy;
    private TriggerType triggerType;
    private Map<String, Object> jobParameters;
    private List<JobStep> steps;
    private String errorMessage;
    private int retryCount;
    private String reportId;
    private List<String> dependentJobIds;
    private boolean prerequisitesMet;
}

/**
 * Enhanced report with multi-module support
 */
@Data
@Document(collection = "clm_reports")
public class ClmReport {
    @Id
    private String reportId;
    private String jobId;
    private String csiId;
    private LocalDateTime generatedAt;
    private ReportType reportType;
    private ReportFormat format;
    
    // Module-specific results
    private ClmResults clmResults;
    private BcmResults bcmResults;
    private EovsResults eovsResults;
    
    // Consolidated action items
    private List<ActionItem> actionItems;
    
    // Raw data for UI
    private Map<String, Object> rawData;
    
    // Report metadata
    private LocalDateTime expiresAt;
    private boolean emailSent;
    private LocalDateTime emailSentAt;
}

/**
 * CLM specific results
 */
@Data
public class ClmResults {
    private DiscoveryResults vmDiscoveryResults;
    private DiscoveryResults containerDiscoveryResults;
    private CertificateInventory inventory;
    private ImpactAssessment impactAssessment;
    private RiskAssessment riskAssessment;
}

/**
 * BCM specific results
 */
@Data
public class BcmResults {
    private int serversScanned;
    private int exceptionsFound;
    private Map<String, Integer> exceptionsByType;
    private List<BcmException> exceptions;
    private int remediationAttempted;
    private int remediationSuccessful;
    private List<RemediationResult> remediationResults;
}

/**
 * EOVS specific results
 */
@Data
public class EovsResults {
    private int vulnerabilitiesFound;
    private Map<String, Integer> vulnerabilitiesBySeverity;
    private List<EovsVulnerability> vulnerabilities;
    private int userAcknowledged;
    private int pendingAcknowledgment;
    private List<RemediationRequest> remediationRequests;
}

// Enums
public enum ModuleType {
    CLM, BCM, EOVS
}

public enum BcmMode {
    DISCOVERY_ONLY, WITH_AUTO_REMEDIATION
}

public enum EovsMode {
    DISCOVERY_ONLY, WITH_USER_ACKNOWLEDGED_REMEDIATION
}

public enum SyncState {
    NOT_STARTED, IN_PROGRESS, COMPLETED, FAILED, PARTIAL
}

public enum ConnectionStatus {
    NOT_TESTED, TESTING, SUCCESS, FAILED, TIMEOUT
}

public enum JobType {
    ONBOARDING_SCAN, SCHEDULED_SCAN, ADHOC_SCAN, RENEWAL, DEPLOYMENT,
    SERVER_SYNC, CERTIFICATE_SYNC, REPOSITORY_SYNC, CONNECTION_TEST,
    BCM_SCAN, BCM_REMEDIATION, EOVS_SCAN, EOVS_REMEDIATION
}
2. Sync Services Implementation
javapackage com.citi.cert_management.container.service;

import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.util.*;
import java.util.concurrent.CompletableFuture;

/**
 * Service for synchronizing server data from drift system
 * This is the first step after CSI onboarding
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class ServerSyncService {
    
    private final DriftApiClient driftClient;
    private final ServerRepository serverRepository;
    private final CsiMetadataRepository csiRepository;
    
    /**
     * Synchronizes server information from drift system
     * 
     * @param csiId CSI identifier
     * @return Sync result with server count
     */
    public CompletableFuture<SyncResult> syncServersFromDrift(String csiId) {
        log.info("Starting server sync from drift for CSI: {}", csiId);
        
        return CompletableFuture.supplyAsync(() -> {
            try {
                updateSyncStatus(csiId, SyncState.IN_PROGRESS);
                
                // Fetch servers from drift
                List<DriftServer> driftServers = driftClient.getServersByCsi(csiId);
                log.info("Found {} servers in drift for CSI: {}", driftServers.size(), csiId);
                
                // Convert and save to MongoDB
                List<Server> servers = new ArrayList<>();
                for (DriftServer driftServer : driftServers) {
                    Server server = convertDriftServer(driftServer, csiId);
                    servers.add(server);
                }
                
                // Bulk save
                serverRepository.saveAll(servers);
                
                // Update sync status
                updateSyncStatus(csiId, SyncState.COMPLETED);
                
                return SyncResult.builder()
                    .success(true)
                    .recordsProcessed(servers.size())
                    .syncType("SERVER_SYNC")
                    .build();
                    
            } catch (Exception e) {
                log.error("Server sync failed for CSI: {}", csiId, e);
                updateSyncStatus(csiId, SyncState.FAILED, e.getMessage());
                
                return SyncResult.builder()
                    .success(false)
                    .errorMessage(e.getMessage())
                    .syncType("SERVER_SYNC")
                    .build();
            }
        });
    }
    
    /**
     * Tests ansible connectivity for synchronized servers
     * 
     * @param csiId CSI identifier
     * @return Connection test results
     */
    public CompletableFuture<ConnectionTestResult> testServerConnections(String csiId) {
        log.info("Testing ansible connections for CSI: {}", csiId);
        
        return CompletableFuture.supplyAsync(() -> {
            List<Server> servers = serverRepository.findByCsiId(csiId);
            ConnectionTestResult result = new ConnectionTestResult();
            result.setTotalServers(servers.size());
            
            List<CompletableFuture<ConnectionStatus>> futures = new ArrayList<>();
            
            for (Server server : servers) {
                CompletableFuture<ConnectionStatus> future = testSingleConnection(server);
                futures.add(future);
            }
            
            // Wait for all tests to complete
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
            
            // Collect results
            int successCount = 0;
            int failedCount = 0;
            
            for (int i = 0; i < servers.size(); i++) {
                ConnectionStatus status = futures.get(i).join();
                servers.get(i).setConnectionStatus(status);
                servers.get(i).setLastConnectionTest(LocalDateTime.now());
                
                if (status == ConnectionStatus.SUCCESS) {
                    successCount++;
                } else {
                    failedCount++;
                }
            }
            
            // Save updated connection status
            serverRepository.saveAll(servers);
            
            result.setSuccessfulConnections(successCount);
            result.setFailedConnections(failedCount);
            result.setServers(servers);
            
            return result;
        });
    }
    
    private CompletableFuture<ConnectionStatus> testSingleConnection(Server server) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                AnsibleResponse response = ansibleService.testConnection(server.getIpAddress());
                return response.isSuccess() ? ConnectionStatus.SUCCESS : ConnectionStatus.FAILED;
            } catch (Exception e) {
                log.error("Connection test failed for server: {}", server.getServerId(), e);
                return ConnectionStatus.FAILED;
            }
        });
    }
    
    private Server convertDriftServer(DriftServer drift, String csiId) {
        Server server = new Server();
        server.setServerId(drift.getServerId());
        server.setServerName(drift.getServerName());
        server.setCsiId(csiId);
        server.setIpAddress(drift.getIpAddress());
        server.setEnvironment(drift.getEnvironment());
        server.setOperatingSystem(drift.getOs());
        server.setAnsibleEnabled(drift.isAnsibleEnabled());
        server.setConnectionStatus(ConnectionStatus.NOT_TESTED);
        server.setSyncedFromDrift(LocalDateTime.now());
        server.setMetadata(drift.getMetadata());
        return server;
    }
}

/**
 * Service for synchronizing certificate data from CAS
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class CertificateSyncService {
    
    private final CasApiClient casClient;
    private final CertificateRepository certificateRepository;
    private final VaultService vaultService;
    
    /**
     * Synchronizes certificates from CAS (Certificate Authority Service)
     * 
     * @param csiId CSI identifier
     * @return Sync result
     */
    public CompletableFuture<SyncResult> syncCertificatesFromCas(String csiId) {
        log.info("Starting certificate sync from CAS for CSI: {}", csiId);
        
        return CompletableFuture.supplyAsync(() -> {
            try {
                // Fetch certificates from CAS
                List<CasCertificate> casCertificates = casClient.getCertificatesByCsi(csiId);
                log.info("Found {} certificates in CAS for CSI: {}", casCertificates.size(), csiId);
                
                // Process each certificate
                int processedCount = 0;
                int failedCount = 0;
                
                for (CasCertificate casCert : casCertificates) {
                    try {
                        Certificate cert = convertCasCertificate(casCert, csiId);
                        
                        // Store in appropriate location based on type
                        if (isContainerCertificate(cert)) {
                            vaultService.storeCertificate(cert);
                        } else {
                            certificateRepository.save(cert);
                        }
                        
                        processedCount++;
                    } catch (Exception e) {
                        log.error("Failed to process certificate: {}", casCert.getCertId(), e);
                        failedCount++;
                    }
                }
                
                return SyncResult.builder()
                    .success(failedCount == 0)
                    .recordsProcessed(processedCount)
                    .recordsFailed(failedCount)
                    .syncType("CERTIFICATE_SYNC")
                    .build();
                    
            } catch (Exception e) {
                log.error("Certificate sync failed for CSI: {}", csiId, e);
                return SyncResult.builder()
                    .success(false)
                    .errorMessage(e.getMessage())
                    .syncType("CERTIFICATE_SYNC")
                    .build();
            }
        });
    }
}

/**
 * Service for retrieving and scanning repositories
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class RepositorySyncService {
    
    private final BitbucketClient bitbucketClient;
    private final GitHubClient githubClient;
    private final CertificateScanner certificateScanner;
    
    /**
     * Retrieves repositories from configured projects and performs prescan
     * 
     * @param csi CSI details with project URLs
     * @return Repository scan results
     */
    public CompletableFuture<RepositoryScanResult> syncRepositories(CsiDetails csi) {
        log.info("Starting repository sync for CSI: {}", csi.getCsiId());
        
        return CompletableFuture.supplyAsync(() -> {
            RepositoryScanResult result = new RepositoryScanResult();
            
            List<CompletableFuture<List<Repository>>> futures = new ArrayList<>();
            
            // Scan Bitbucket if configured
            if (csi.getBitbucketProjectUrl() != null) {
                futures.add(scanBitbucketProject(csi));
            }
            
            // Scan GitHub if configured
            if (csi.getGithubProjectUrl() != null) {
                futures.add(scanGitHubProject(csi));
            }
            
            // Wait for all scans to complete
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
            
            // Collect results
            List<Repository> allRepos = new ArrayList<>();
            futures.forEach(future -> allRepos.addAll(future.join()));
            
            result.setRepositoriesFound(allRepos.size());
            result.setRepositories(allRepos);
            
            // Prescan for certificates
            result.setCertificateLocations(prescanRepositories(allRepos, csi));
            
            return result;
        });
    }
    
    /**
     * Prescans repositories for certificate files
     */
    private List<CertificateLocation> prescanRepositories(List<Repository> repos, 
                                                         CsiDetails csi) {
        log.info("Prescanning {} repositories for certificates", repos.size());
        
        List<CertificateLocation> allLocations = new ArrayList<>();
        
        for (Repository repo : repos) {
            try {
                List<CertificateLocation> locations = certificateScanner.scanRepository(
                    repo.getCloneUrl(),
                    csi.getPidUsername(),
                    csi.getPidPassword()
                );
                
                allLocations.addAll(locations);
            } catch (Exception e) {
                log.error("Failed to prescan repository: {}", repo.getName(), e);
            }
        }
        
        log.info("Found {} potential certificate locations", allLocations.size());
        return allLocations;
    }
}
3. Enhanced Job Orchestration with Module Support
javapackage com.citi.cert_management.container.service;

import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.util.*;
import java.util.concurrent.CompletableFuture;

/**
 * Enhanced job orchestration service supporting multiple modules
 * Ensures proper execution order and dependency management
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class EnhancedJobOrchestrationService {
    
    private final Map<ModuleType, ModuleExecutor> moduleExecutors;
    private final ServerSyncService serverSyncService;
    private final CertificateSyncService certificateSyncService;
    private final RepositorySyncService repositorySyncService;
    private final ConnectionTestService connectionTestService;
    private final ReportGenerationService reportService;
    
    /**
     * Initiates complete onboarding flow with proper sequencing
     * 
     * @param csiId CSI identifier
     * @param userId User who triggered onboarding
     * @return Master job ID tracking all sub-jobs
     */
    public String initiateOnboardingFlow(String csiId, String userId) {
        log.info("Initiating onboarding flow for CSI: {}", csiId);
        
        CsiDetails csi = csiRepository.findByCsiId(csiId)
            .orElseThrow(() -> new CsiNotFoundException("CSI not found: " + csiId));
        
        // Create master job
        ClmJob masterJob = createMasterJob(csiId, userId);
        jobRepository.save(masterJob);
        
        // Execute onboarding flow asynchronously
        CompletableFuture.runAsync(() -> executeOnboardingFlow(masterJob, csi))
            .exceptionally(throwable -> {
                handleJobFailure(masterJob.getJobId(), throwable);
                return null;
            });
        
        return masterJob.getJobId();
    }
    
    /**
     * Executes complete onboarding flow in proper sequence
     */
    private void executeOnboardingFlow(ClmJob masterJob, CsiDetails csi) {
        try {
            updateJobStatus(masterJob.getJobId(), JobStatus.RUNNING);
            
            // Step 1: Parallel sync operations
            CompletableFuture<SyncResult> serverSync = serverSyncService
                .syncServersFromDrift(csi.getCsiId());
            
            CompletableFuture<SyncResult> certSync = certificateSyncService
                .syncCertificatesFromCas(csi.getCsiId());
            
            CompletableFuture<RepositoryScanResult> repoSync = repositorySyncService
                .syncRepositories(csi);
            
            // Wait for all syncs to complete
            CompletableFuture.allOf(serverSync, certSync, repoSync).join();
            
            // Check sync results
            if (!serverSync.join().isSuccess() || 
                !certSync.join().isSuccess() || 
                !repoSync.join().getRepositoriesFound() > 0) {
                throw new SyncException("One or more sync operations failed");
            }
            
            // Step 2: Connection testing
            ConnectionTestResult connectionResult = connectionTestService
                .testServerConnections(csi.getCsiId()).join();
            
            // Step 3: Module-specific operations for connected servers
            List<Server> connectedServers = connectionResult.getConnectedServers();
            
            if (!connectedServers.isEmpty()) {
                executeModuleOperations(masterJob, csi, connectedServers);
            }
            
            // Step 4: Generate consolidated report
            ClmReport report = reportService.generateOnboardingReport(masterJob);
            
            // Step 5: Send notification
            notificationService.sendOnboardingCompleteNotification(csi, report);
            
            completeJob(masterJob.getJobId(), report.getReportId());
            
        } catch (Exception e) {
            log.error("Onboarding flow failed for CSI: {}", csi.getCsiId(), e);
            failJob(masterJob.getJobId(), e.getMessage());
        }
    }
    
    /**
     * Executes module-specific operations based on subscriptions
     */
    private void executeModuleOperations(ClmJob masterJob, CsiDetails csi, 
                                       List<Server> connectedServers) {
        log.info("Executing module operations for {} connected servers", connectedServers.size());
        
        List<CompletableFuture<ModuleExecutionResult>> moduleJobs = new ArrayList<>();
        
        for (ModuleSubscription subscription : csi.getModuleSubscriptions()) {
            if (!subscription.isSelected()) {
                continue;
            }
            
            ModuleExecutor executor = moduleExecutors.get(subscription.getModule());
            if (executor != null) {
                CompletableFuture<ModuleExecutionResult> moduleJob = CompletableFuture
                    .supplyAsync(() -> executor.execute(csi, connectedServers, subscription));
                
                moduleJobs.add(moduleJob);
            }
        }
        
        // Wait for all module operations to complete
        CompletableFuture.allOf(moduleJobs.toArray(new CompletableFuture[0])).join();
        
        // Collect results
        List<ModuleExecutionResult> results = moduleJobs.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
        
        // Update master job with module results
        masterJob.getJobParameters().put("moduleResults", results);
    }
    
    /**
     * Handles scheduled batch processing for all modules
     */
    public void executeScheduledBatch(LocalDateTime scheduledTime) {
        log.info("Executing scheduled batch at: {}", scheduledTime);
        
        // Get all active CSIs
        List<CsiDetails> activeCsis = csiRepository.findByActiveTrue();
        
        // Group by module subscriptions for efficient processing
        Map<ModuleType, List<CsiDetails>> csisByModule = groupCsisByModule(activeCsis);
        
        // Execute batch for each module
        csisByModule.forEach((module, csiList) -> {
            String batchJobId = createBatchJob(module, csiList, scheduledTime);
            executeBatchJob(batchJobId, module, csiList);
        });
    }
    
    /**
     * Handles adhoc scan with module selection
     */
    public String performAdhocScan(String csiId, List<ModuleType> modules, String triggeredBy) {
        log.info("Performing adhoc scan for CSI: {} with modules: {}", csiId, modules);
        
        CsiDetails csi = csiRepository.findByCsiId(csiId)
            .orElseThrow(() -> new CsiNotFoundException("CSI not found: " + csiId));
        
        // Create adhoc job
        ClmJob adhocJob = new ClmJob();
        adhocJob.setJobId(UUID.randomUUID().toString());
        adhocJob.setCsiId(csiId);
        adhocJob.setJobType(JobType.ADHOC_SCAN);
        adhocJob.setTriggeredBy(triggeredBy);
        adhocJob.setStartTime(LocalDateTime.now());
        
        Map<String, Object> parameters = new HashMap<>();
        parameters.put("modules", modules);
        adhocJob.setJobParameters(parameters);
        
        jobRepository.save(adhocJob);
        
        // Execute asynchronously
        CompletableFuture.runAsync(() -> executeAdhocScan(adhocJob, csi, modules));
        
        return adhocJob.getJobId();
    }
}

/**
 * Interface for module executors
 */
public interface ModuleExecutor {
    ModuleExecutionResult execute(CsiDetails csi, List<Server> servers, 
                                 ModuleSubscription subscription);
    ModuleType getModuleType();
}

/**
 * CLM Module Executor
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class ClmModuleExecutor implements ModuleExecutor {
    
    private final VmDiscoveryService vmDiscoveryService;
    private final ContainerDiscoveryService containerDiscoveryService;
    
    @Override
    public ModuleExecutionResult execute(CsiDetails csi, List<Server> servers, 
                                       ModuleSubscription subscription) {
        log.info("Executing CLM module for CSI: {}", csi.getCsiId());
        
        ModuleConfiguration config = subscription.getConfiguration();
        ClmResults results = new ClmResults();
        
        // VM Discovery
        if (config.isVmDiscoveryEnabled()) {
            DiscoveryResults vmResults = vmDiscoveryService
                .discoverCertificates(csi.getCsiId(), servers);
            results.setVmDiscoveryResults(vmResults);
        }
        
        // Container Discovery
        if (config.isContainerDiscoveryEnabled()) {
            DiscoveryResults containerResults = containerDiscoveryService
                .discoverCertificates(csi);
            results.setContainerDiscoveryResults(containerResults);
        }
        
        // Generate inventory
        CertificateInventory inventory = generateInventory(results);
        results.setInventory(inventory);
        
        return ModuleExecutionResult.builder()
            .module(ModuleType.CLM)
            .success(true)
            .results(results)
            .build();
    }
    
    @Override
    public ModuleType getModuleType() {
        return ModuleType.CLM;
    }
}

/**
 * BCM Module Executor
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class BcmModuleExecutor implements ModuleExecutor {
    
    private final BcmScanService bcmScanService;
    private final BcmRemediationService remediationService;
    
    @Override
    public ModuleExecutionResult execute(CsiDetails csi, List<Server> servers, 
                                       ModuleSubscription subscription) {
        log.info("Executing BCM module for CSI: {}", csi.getCsiId());
        
        ModuleConfiguration config = subscription.getConfiguration();
        BcmResults results = new BcmResults();
        
        // Run BCM scan
        BcmScanResult scanResult = bcmScanService.performBcmScan(csi.getCsiId(), servers);
        results.setServersScanned(scanResult.getServersScanned());
        results.setExceptionsFound(scanResult.getExceptionsFound());
        results.setExceptionsByType(scanResult.getExceptionsByType());
        results.setExceptions(scanResult.getExceptions());
        
        // Auto-remediation if enabled
        if (config.getBcmMode() == BcmMode.WITH_AUTO_REMEDIATION && 
            config.isBcmAutoRemediationEnabled()) {
            
            List<RemediationResult> remediationResults = remediationService
                .performAutoRemediation(csi.getCsiId(), scanResult.getExceptions());
            
            results.setRemediationAttempted(remediationResults.size());
            results.setRemediationSuccessful(
                (int) remediationResults.stream()
                    .filter(RemediationResult::isSuccess)
                    .count()
            );
            results.setRemediationResults(remediationResults);
        }
        
        return ModuleExecutionResult.builder()
            .module(ModuleType.BCM)
            .success(true)
            .results(results)
            .build();
    }
    
    @Override
    public ModuleType getModuleType() {
        return ModuleType.BCM;
    }
}

/**
 * EOVS Module Executor
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class EovsModuleExecutor implements ModuleExecutor {
    
    private final EovsInventoryService inventoryService;
    private final EovsRemediationService remediationService;
    
    @Override
    public ModuleExecutionResult execute(CsiDetails csi, List<Server> servers, 
                                       ModuleSubscription subscription) {
        log.info("Executing EOVS module for CSI: {}", csi.getCsiId());
        
        ModuleConfiguration config = subscription.getConfiguration();
        EovsResults results = new EovsResults();
        
        // EOVS uses existing inventory
        EovsInventoryResult inventory = inventoryService.getEovsInventory(csi.getCsiId());
        results.setVulnerabilitiesFound(inventory.getTotalVulnerabilities());
        results.setVulnerabilitiesBySeverity(inventory.getVulnerabilitiesBySeverity());
        results.setVulnerabilities(inventory.getVulnerabilities());
        
        // Handle user-acknowledged remediation if enabled
        if (config.getEovsMode() == EovsMode.WITH_USER_ACKNOWLEDGED_REMEDIATION) {
            List<RemediationRequest> requests = remediationService
                .getPendingRemediations(csi.getCsiId());
            
            results.setRemediationRequests(requests);
            results.setPendingAcknowledgment(requests.size());
        }
        
        return ModuleExecutionResult.builder()
            .module(ModuleType.EOVS)
            .success(true)
            .results(results)
            .build();
    }
    
    @Override
    public ModuleType getModuleType() {
        return ModuleType.EOVS;
    }
}
4. Enhanced Controllers with Module Support
javapackage com.citi.cert_management.container.controller;

import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import javax.validation.Valid;
import java.util.*;

/**
 * Enhanced REST controller with module subscription support
 */
@Slf4j
@RestController
@RequestMapping("/api/v1/clm")
@RequiredArgsConstructor
@Tag(name = "CLM API", description = "Certificate Lifecycle Management with module support")
public class EnhancedClmController {
    
    private final CsiService csiService;
    private final EnhancedJobOrchestrationService jobOrchestrationService;
    private final ReportService reportService;
    
    /**
     * Enhanced CSI onboarding with module subscriptions
     * 
     * @param request CSI onboarding request with module selections
     * @return ResponseEntity with onboarding result and job ID
     */
    @PostMapping("/csi/onboard")
    @Operation(summary = "Onboard CSI with module subscriptions",
              description = "Creates CSI profile with selected modules and initiates sync operations")
    public ResponseEntity<OnboardingResponse> onboardCsi(@Valid @RequestBody EnhancedCsiOnboardingRequest request) {
        log.info("Onboarding CSI: {} with modules: {}", request.getCsiId(), 
                request.getModuleSubscriptions().stream()
                    .filter(ModuleSubscriptionRequest::isSelected)
                    .map(m -> m.getModule())
                    .collect(Collectors.toList()));
        
        validateMandatoryFields(request);
        validateModuleSubscriptions(request.getModuleSubscriptions());
        
        // Create CSI with module subscriptions
        CsiDetails csi = csiService.createCsiWithModules(request);
        
        // Initiate onboarding flow
        String masterJobId = jobOrchestrationService.initiateOnboardingFlow(
            csi.getCsiId(), 
            request.getRequestorSoeid()
        );
        
        return ResponseEntity.ok(OnboardingResponse.builder()
            .csiId(csi.getCsiId())
            .jobId(masterJobId)
            .message("CSI onboarded successfully. Sync and discovery operations initiated.")
            .subscribedModules(getSubscribedModules(csi))
            .build());
    }
    
    /**
     * Gets sync status for CSI
     */
    @GetMapping("/csi/{csiId}/sync-status")
    @Operation(summary = "Get synchronization status",
              description = "Returns status of server, certificate, and repository sync operations")
    public ResponseEntity<SyncStatusResponse> getSyncStatus(@PathVariable String csiId) {
        
        CsiDetails csi = csiService.getCsi(csiId);
        
        return ResponseEntity.ok(SyncStatusResponse.builder()
            .csiId(csiId)
            .serverSyncStatus(csi.getServerSyncStatus())
            .certificateSyncStatus(csi.getCertificateSyncStatus())
            .repositorySyncStatus(csi.getRepositorySyncStatus())
            .lastSuccessfulSync(csi.getLastSuccessfulSync())
            .canProceedWithDiscovery(canProceedWithDiscovery(csi))
            .build());
    }
    
    /**
     * Triggers module-specific operations
     */
    @PostMapping("/csi/{csiId}/module/{moduleType}/execute")
    @Operation(summary = "Execute specific module operations",
              description = "Triggers operations for a specific module (CLM, BCM, or EOVS)")
    public ResponseEntity<JobResponse> executeModuleOperation(
            @PathVariable String csiId,
            @PathVariable ModuleType moduleType,
            @RequestParam(required = false) String triggeredBy,
            @RequestBody(required = false) ModuleOperationRequest request) {
        
        // Validate module subscription
        CsiDetails csi = csiService.getCsi(csiId);
        if (!isModuleSubscribed(csi, moduleType)) {
            return ResponseEntity.badRequest()
                .body(new JobResponse(null, "Module not subscribed: " + moduleType));
        }
        
        String jobId = jobOrchestrationService.executeModuleOperation(
            csiId, moduleType, triggeredBy, request);
        
        return ResponseEntity.ok(new JobResponse(jobId, 
            moduleType + " operation initiated for CSI: " + csiId));
    }
    
    /**
     * Updates module configuration
     */
    @PutMapping("/csi/{csiId}/module/{moduleType}")
    @Operation(summary = "Update module configuration",
              description = "Updates configuration for a specific module")
    public ResponseEntity<ModuleSubscription> updateModuleConfiguration(
            @PathVariable String csiId,
            @PathVariable ModuleType moduleType,
            @Valid @RequestBody ModuleConfigurationRequest request) {
        
        ModuleSubscription updated = csiService.updateModuleConfiguration(
            csiId, moduleType, request);
        
        return ResponseEntity.ok(updated);
    }
    
    /**
     * Gets consolidated report across all modules
     */
    @GetMapping("/csi/{csiId}/consolidated-report")
    @Operation(summary = "Get consolidated report",
              description = "Returns consolidated report across all subscribed modules")
    public ResponseEntity<ConsolidatedReport> getConsolidatedReport(
            @PathVariable String csiId,
            @RequestParam(required = false) String jobId,
            @RequestParam(defaultValue = "JSON") ReportFormat format) {
        
        ClmReport report;
        if (jobId != null) {
            report = reportService.getReportByJobId(jobId, format);
        } else {
            report = reportService.getLatestReport(csiId, format);
        }
        
        ConsolidatedReport consolidated = buildConsolidatedReport(report);
        
        return ResponseEntity.ok(consolidated);
    }
    
    /**
     * Performs adhoc scan with module selection
     */
    @PostMapping("/csi/{csiId}/adhoc-scan")
    @Operation(summary = "Perform adhoc scan",
              description = "Executes adhoc scan for selected modules")
    public ResponseEntity<JobResponse> performAdhocScan(
            @PathVariable String csiId,
            @RequestBody AdhocScanRequest request) {
        
        // Validate selected modules are subscribed
        CsiDetails csi = csiService.getCsi(csiId);
        List<ModuleType> validModules = validateSelectedModules(csi, request.getModules());
        
        if (validModules.isEmpty()) {
            return ResponseEntity.badRequest()
                .body(new JobResponse(null, "No valid modules selected for scan"));
        }
        
        String jobId = jobOrchestrationService.performAdhocScan(
            csiId, validModules, request.getTriggeredBy());
        
        return ResponseEntity.ok(new JobResponse(jobId, 
            "Adhoc scan initiated for modules: " + validModules));
    }
    
    /**
     * BCM-specific: Triggers remediation
     */
    @PostMapping("/csi/{csiId}/bcm/remediate")
    @Operation(summary = "Trigger BCM remediation",
              description = "Initiates remediation for BCM exceptions")
    public ResponseEntity<RemediationResponse> triggerBcmRemediation(
            @PathVariable String csiId,
            @RequestBody BcmRemediationRequest request) {
        
        // Ensure BCM is subscribed with remediation enabled
        CsiDetails csi = csiService.getCsi(csiId);
        ModuleSubscription bcmSubscription = getModuleSubscription(csi, ModuleType.BCM);
        
        if (bcmSubscription == null || 
            bcmSubscription.getConfiguration().getBcmMode() != BcmMode.WITH_AUTO_REMEDIATION) {
            return ResponseEntity.badRequest()
                .body(RemediationResponse.builder()
                    .success(false)
                    .message("BCM auto-remediation not enabled")
                    .build());
        }
        
        RemediationResult result = bcmService.performRemediation(csiId, request);
        
        return ResponseEntity.ok(RemediationResponse.builder()
            .success(result.isSuccess())
            .remediationId(result.getRemediationId())
            .message(result.getMessage())
            .build());
    }
    
    /**
     * EOVS-specific: Acknowledge remediation request
     */
    @PostMapping("/csi/{csiId}/eovs/acknowledge")
    @Operation(summary = "Acknowledge EOVS remediation",
              description = "User acknowledgment for EOVS remediation requests")
    public ResponseEntity<AcknowledgmentResponse> acknowledgeEovsRemediation(
            @PathVariable String csiId,
            @RequestBody EovsAcknowledgmentRequest request) {
        
        // Ensure EOVS is subscribed with user acknowledgment enabled
        CsiDetails csi = csiService.getCsi(csiId);
        ModuleSubscription eovsSubscription = getModuleSubscription(csi, ModuleType.EOVS);
        
        if (eovsSubscription == null || 
            eovsSubscription.getConfiguration().getEovsMode() != 
            EovsMode.WITH_USER_ACKNOWLEDGED_REMEDIATION) {
            return ResponseEntity.badRequest()
                .body(AcknowledgmentResponse.builder()
                    .success(false)
                    .message("EOVS user acknowledgment not enabled")
                    .build());
        }
        
        AcknowledgmentResult result = eovsService.acknowledgeRemediation(csiId, request);
        
        return ResponseEntity.ok(AcknowledgmentResponse.builder()
            .success(result.isSuccess())
            .acknowledgedCount(result.getAcknowledgedCount())
            .message(result.getMessage())
            .build());
    }
    
    private void validateModuleSubscriptions(List<ModuleSubscriptionRequest> subscriptions) {
        // Ensure at least one module is selected
        boolean hasSelection = subscriptions.stream()
            .anyMatch(ModuleSubscriptionRequest::isSelected);
        
        if (!hasSelection) {
            throw new ValidationException("At least one module must be selected");
        }
        
        // Validate module-specific configurations
        for (ModuleSubscriptionRequest subscription : subscriptions) {
            if (subscription.isSelected()) {
                validateModuleConfiguration(subscription);
            }
        }
    }
}
5. Enhanced Request/Response DTOsMapped out Thymeleaf context and DTO enhancement strategy.The user wants me to continue with the enhanced request/response DTOs and also answer their question about Thymeleaf Context. Let me address both.
Yes, when using Thymeleaf's Context and TemplateEngine, it can return HTML content with styles. The styles can be:

Inline styles directly in the HTML
Internal styles in <style> tags
External stylesheets referenced in the template

I'll continue with the DTOs and then provide the enhanced email templates and documentation.javapackage com.citi.cert_management.container.dto;

import lombok.Data;
import lombok.Builder;
import javax.validation.constraints.NotNull;
import java.util.*;

/**
 * Enhanced CSI onboarding request with module subscriptions
 */
@Data
public class EnhancedCsiOnboardingRequest {
    @NotNull
    private String csiId;
    
    @NotNull
    private String requestorSoeid;
    
    @NotNull
    private String emailDl;
    
    @NotNull
    private String email;
    
    @NotNull
    private String preferredRenewal;
    
    private List<String> notificationEmailList;
    private List<String> escalationMatrix;
    
    private String bitbucketProjectUrl;
    private String githubProjectUrl;
    
    // Module subscriptions
    @NotNull
    private List<ModuleSubscriptionRequest> moduleSubscriptions;
    
    private Map<String, Object> additionalConfig;
}

/**
 * Module subscription request
 */
@Data
public class ModuleSubscriptionRequest {
    @NotNull
    private ModuleType module;
    
    private boolean isSelected;
    
    private ModuleConfigurationRequest configuration;
}

/**
 * Module configuration request
 */
@Data
public class ModuleConfigurationRequest {
    // CLM specific
    private boolean vmDiscoveryEnabled;
    private boolean containerDiscoveryEnabled;
    private boolean autoDeploymentEnabled;
    
    // BCM specific
    private BcmMode bcmMode;
    private List<String> bcmExceptionTypes;
    
    // EOVS specific
    private EovsMode eovsMode;
    private List<String> eovsChecks;
}

/**
 * Sync result
 */
@Data
@Builder
public class SyncResult {
    private boolean success;
    private int recordsProcessed;
    private int recordsFailed;
    private String errorMessage;
    private String syncType;
    private LocalDateTime completedAt;
}

/**
 * Connection test result
 */
@Data
@Builder
public class ConnectionTestResult {
    private int totalServers;
    private int successfulConnections;
    private int failedConnections;
    private List<Server> servers;
    private List<Server> connectedServers;
    
    public List<Server> getConnectedServers() {
        return servers.stream()
            .filter(s -> s.getConnectionStatus() == ConnectionStatus.SUCCESS)
            .collect(Collectors.toList());
    }
}

/**
 * Repository scan result
 */
@Data
public class RepositoryScanResult {
    private int repositoriesFound;
    private List<Repository> repositories;
    private List<CertificateLocation> certificateLocations;
    private LocalDateTime scanCompletedAt;
}

/**
 * Module execution result
 */
@Data
@Builder
public class ModuleExecutionResult {
    private ModuleType module;
    private boolean success;
    private Object results;
    private String errorMessage;
    private LocalDateTime executedAt;
}

/**
 * Consolidated report across modules
 */
@Data
@Builder
public class ConsolidatedReport {
    private String reportId;
    private String csiId;
    private LocalDateTime generatedAt;
    
    // Module results
    private ClmReportSection clmSection;
    private BcmReportSection bcmSection;
    private EovsReportSection eovsSection;
    
    // Summary
    private ReportSummary summary;
    private List<ActionItem> consolidatedActionItems;
    
    // Report metadata
    private List<ModuleType> includedModules;
    private ReportFormat format;
}

/**
 * CLM report section
 */
@Data
public class ClmReportSection {
    private boolean included;
    private DiscoverySummary vmDiscovery;
    private DiscoverySummary containerDiscovery;
    private CertificateInventorySummary inventory;
    private int criticalCertificates;
    private int expiringIn30Days;
}

/**
 * BCM report section
 */
@Data
public class BcmReportSection {
    private boolean included;
    private int serversScanned;
    private int exceptionsFound;
    private Map<String, Integer> exceptionsByType;
    private int remediationAttempted;
    private int remediationSuccessful;
    private List<BcmExceptionSummary> topExceptions;
}

/**
 * EOVS report section
 */
@Data
public class EovsReportSection {
    private boolean included;
    private int vulnerabilitiesFound;
    private Map<String, Integer> vulnerabilitiesBySeverity;
    private int pendingAcknowledgment;
    private int userAcknowledged;
    private List<EovsVulnerabilitySummary> criticalVulnerabilities;
}

/**
 * Sync status response
 */
@Data
@Builder
public class SyncStatusResponse {
    private String csiId;
    private SyncStatus serverSyncStatus;
    private SyncStatus certificateSyncStatus;
    private SyncStatus repositorySyncStatus;
    private LocalDateTime lastSuccessfulSync;
    private boolean canProceedWithDiscovery;
}

/**
 * BCM remediation request
 */
@Data
public class BcmRemediationRequest {
    private List<String> exceptionIds;
    private boolean forceRemediation;
    private String approvedBy;
    private String reason;
}

/**
 * EOVS acknowledgment request
 */
@Data
public class EovsAcknowledgmentRequest {
    private List<String> vulnerabilityIds;
    private String acknowledgedBy;
    private String businessJustification;
    private LocalDateTime acknowledgmentExpiry;
}
6. Enhanced Email Templates with Styles
Yes, Thymeleaf Context returns HTML with styles!
The Thymeleaf template engine can include:

Inline CSS styles
Internal <style> blocks
External CSS references

Here's an example template:
html<!-- clm-report-email.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>CLM Report - [[${csiId}]]</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            color: #333;
            margin: 0;
            padding: 0;
            background-color: #f4f4f4;
        }
        .container {
            max-width: 800px;
            margin: 20px auto;
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .header {
            background-color: #004080;
            color: white;
            padding: 20px;
            border-radius: 8px 8px 0 0;
            margin: -20px -20px 20px -20px;
        }
        .module-section {
            margin: 20px 0;
            padding: 15px;
            border-left: 4px solid #004080;
            background-color: #f9f9f9;
        }
        .summary-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 15px;
            margin: 20px 0;
        }
        .summary-card {
            background: #e8f0fe;
            padding: 15px;
            border-radius: 4px;
            text-align: center;
        }
        .summary-card .number {
            font-size: 2em;
            font-weight: bold;
            color: #004080;
        }
        .critical {
            background-color: #ffebee;
            border-color: #d32f2f;
        }
        .warning {
            background-color: #fff3e0;
            border-color: #f57c00;
        }
        .success {
            background-color: #e8f5e9;
            border-color: #388e3c;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin: 15px 0;
        }
        th, td {
            padding: 10px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }
        th {
            background-color: #004080;
            color: white;
        }
        .action-required {
            background-color: #ffebee;
            padding: 15px;
            border-radius: 4px;
            margin: 15px 0;
        }
        .footer {
            text-align: center;
            margin-top: 30px;
            padding-top: 20px;
            border-top: 1px solid #ddd;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Certificate Lifecycle Management Report</h1>
            <p>CSI: [[${csiId}]] - [[${csiName}]]</p>
            <p>Report Date: [[${#temporals.format(reportDate, 'MMM dd, yyyy HH:mm')}]]</p>
        </div>
        
        <!-- Summary Section -->
        <h2>Executive Summary</h2>
        <div class="summary-grid">
            <div class="summary-card">
                <div class="number">[[${summary.totalCertificates}]]</div>
                <div>Total Certificates</div>
            </div>
            <div class="summary-card" th:classappend="${summary.expiringIn30Days > 0} ? 'critical'">
                <div class="number">[[${summary.expiringIn30Days}]]</div>
                <div>Expiring in 30 Days</div>
            </div>
            <div class="summary-card">
                <div class="number">[[${summary.totalActions}]]</div>
                <div>Action Items</div>
            </div>
            <div class="summary-card" th:classappend="${summary.criticalActions > 0} ? 'warning'">
                <div class="number">[[${summary.criticalActions}]]</div>
                <div>Critical Actions</div>
            </div>
        </div>
        
        <!-- CLM Module Section -->
        <div class="module-section" th:if="${clmSection != null && clmSection.included}">
            <h3>📋 Certificate Lifecycle Management (CLM)</h3>
            
            <h4>Discovery Results</h4>
            <table>
                <tr>
                    <th>Type</th>
                    <th>Servers Scanned</th>
                    <th>Certificates Found</th>
                    <th>Status</th>
                </tr>
                <tr th:if="${vmDiscovery != null}">
                    <td>VM Discovery</td>
                    <td>[[${vmDiscovery.totalServersScanned}]]</td>
                    <td>[[${vmDiscovery.totalCertsFound}]]</td>
                    <td><span class="success">✓ Complete</span></td>
                </tr>
                <tr th:if="${containerDiscovery != null}">
                    <td>Container Discovery</td>
                    <td>[[${containerDiscovery.totalServersScanned}]]</td>
                    <td>[[${containerDiscovery.totalCertsFound}]]</td>
                    <td><span class="success">✓ Complete</span></td>
                </tr>
            </table>
            
            <h4>Certificate Inventory by Expiry</h4>
            <table>
                <tr>
                    <th>Category</th>
                    <th>Count</th>
                    <th>Action Required</th>
                </tr>
                <tr th:each="entry : ${inventory.certsByExpiryCategory}">
                    <td>[[${entry.key}]]</td>
                    <td>[[${entry.value}]]</td>
                    <td th:switch="${entry.key}">
                        <span th:case="'EXPIRED'" style="color: #d32f2f;">Immediate renewal required</span>
                        <span th:case="'30_DAYS'" style="color: #f57c00;">Urgent renewal needed</span>
                        <span th:case="'60_DAYS'" style="color: #fbc02d;">Plan renewal</span>
                        <span th:case="*">Monitor</span>
                    </td>
                </tr>
            </table>
        </div>
        
        <!-- BCM Module Section -->
        <div class="module-section" th:if="${bcmSection != null && bcmSection.included}">
            <h3>🛡️ Baseline Configuration Monitoring (BCM)</h3>
            
            <p><strong>Servers Scanned:</strong> [[${bcmSection.serversScanned}]]</p>
            <p><strong>Total Exceptions:</strong> [[${bcmSection.exceptionsFound}]]</p>
            
            <div th:if="${bcmSection.remediationAttempted > 0}">
                <h4>Auto-Remediation Results</h4>
                <p>Attempted: [[${bcmSection.remediationAttempted}]]</p>
                <p>Successful: [[${bcmSection.remediationSuccessful}]]</p>
            </div>
            
            <h4>Top Exceptions by Type</h4>
            <table>
                <tr>
                    <th>Exception Type</th>
                    <th>Count</th>
                </tr>
                <tr th:each="exception : ${bcmSection.exceptionsByType}">
                    <td>[[${exception.key}]]</td>
                    <td>[[${exception.value}]]</td>
                </tr>
            </table>
        </div>
        
        <!-- EOVS Module Section -->
        <div class="module-section" th:if="${eovsSection != null && eovsSection.included}">
            <h3>🔍 Enterprise Operating Vulnerability Scanner (EOVS)</h3>
            
            <p><strong>Total Vulnerabilities:</strong> [[${eovsSection.vulnerabilitiesFound}]]</p>
            
            <h4>Vulnerabilities by Severity</h4>
            <div class="summary-grid">
                <div class="summary-card critical" th:if="${eovsSection.vulnerabilitiesBySeverity['CRITICAL'] != null}">
                    <div class="number">[[${eovsSection.vulnerabilitiesBySeverity['CRITICAL']}]]</div>
                    <div>Critical</div>
                </div>
                <div class="summary-card warning" th:if="${eovsSection.vulnerabilitiesBySeverity['HIGH'] != null}">
                    <div class="number">[[${eovsSection.vulnerabilitiesBySeverity['HIGH']}]]</div>
                    <div>High</div>
                </div>
                <div class="summary-card" th:if="${eovsSection.vulnerabilitiesBySeverity['MEDIUM'] != null}">
                    <div class="number">[[${eovsSection.vulnerabilitiesBySeverity['MEDIUM']}]]</div>
                    <div>Medium</div>
                </div>
            </div>
            
            <div th:if="${eovsSection.pendingAcknowledgment > 0}" class="action-required">
                <strong>⚠️ User Acknowledgment Required:</strong> 
                [[${eovsSection.pendingAcknowledgment}]] vulnerabilities pending acknowledgment
            </div>
        </div>
        
        <!-- Action Items Section -->
        <div th:if="${!#lists.isEmpty(actionItems)}" class="action-required">
            <h3>⚡ Action Items Required</h3>
            <table>
                <tr>
                    <th>Priority</th>
                    <th>Type</th>
                    <th>Description</th>
                    <th>Due Date</th>
                </tr>
                <tr th:each="item : ${actionItems}" 
                    th:classappend="${item.priority == 'CRITICAL'} ? 'critical-row'">
                    <td>[[${item.priority}]]</td>
                    <td>[[${item.type}]]</td>
                    <td>[[${item.description}]]</td>
                    <td>[[${#temporals.format(item.dueDate, 'MMM dd, yyyy')}]]</td>
                </tr>
            </table>
        </div>
        
        <div class="footer">
            <p>This is an automated report generated by the CLM system.</p>
            <p>For questions, please contact the CLM support team.</p>
        </div>
    </div>
</body>
</html>
7. Detailed Documentation
markdown# Certificate Lifecycle Management System - Complete Documentation

## Table of Contents
1. [System Overview](#system-overview)
2. [Architecture](#architecture)
3. [Module System](#module-system)
4. [Onboarding Flow](#onboarding-flow)
5. [API Reference](#api-reference)
6. [Integration Points](#integration-points)
7. [Configuration Guide](#configuration-guide)
8. [Deployment](#deployment)
9. [Troubleshooting](#troubleshooting)

## System Overview

The Certificate Lifecycle Management (CLM) system is a modular platform that manages certificates across VM and container environments. It supports three primary modules:

- **CLM (Certificate Lifecycle Management)**: Core certificate discovery, renewal, and deployment
- **BCM (Baseline Configuration Monitoring)**: Security compliance scanning and remediation
- **EOVS (Enterprise Operating Vulnerability Scanner)**: Vulnerability detection and management

### Key Features

1. **Modular Architecture**: Subscribe to only the modules you need
2. **Automated Synchronization**: Pulls data from drift, CAS, and repositories
3. **Intelligent Orchestration**: Ensures proper execution order
4. **Multi-Environment Support**: Works across VMs and containers
5. **Comprehensive Reporting**: HTML emails and JSON APIs
6. **Escalation Management**: Automatic escalation for unresolved issues

## Architecture

### High-Level Architecture
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   UI (Angular)  │────▶│   Spring Boot   │────▶│    MongoDB      │
└─────────────────┘     │   Application   │     └─────────────────┘
└────────┬────────┘
│
┌────────────┼────────────┐
▼            ▼            ▼
┌─────────────┐ ┌─────────┐ ┌─────────────┐
│   Ansible   │ │  Vault  │ │ Git Repos   │
│  Playbooks  │ │   API   │ │ BB/GitHub   │
└─────────────┘ └─────────┘ └─────────────┘
│
┌───────┴────────┐
▼                ▼
┌─────────────┐  ┌─────────────┐
│    Drift    │  │     CAS     │
│   (Servers) │  │   (Certs)   │
└─────────────┘  └─────────────┘

### Component Breakdown

#### 1. Core Services
- **CsiService**: Manages CSI profiles and module subscriptions
- **EnhancedJobOrchestrationService**: Handles job execution and dependencies
- **Module Executors**: CLM, BCM, and EOVS specific implementations

#### 2. Sync Services
- **ServerSyncService**: Synchronizes server data from drift
- **CertificateSyncService**: Pulls certificates from CAS
- **RepositorySyncService**: Scans and indexes repositories

#### 3. Discovery Services
- **VmDiscoveryService**: Ansible-based VM certificate discovery
- **ContainerDiscoveryService**: Repository and vault-based discovery

#### 4. Support Services
- **NotificationService**: Email generation and delivery
- **ReportGenerationService**: Report compilation across modules
- **DeploymentService**: Certificate deployment to repositories

## Module System

### CLM Module

**Purpose**: Core certificate lifecycle management

**Features**:
- VM certificate discovery via Ansible
- Container certificate discovery via repositories
- Certificate inventory management
- Auto-renewal capabilities
- Repository deployment

**Configuration Options**:
```json
{
  "module": "CLM",
  "isSelected": true,
  "configuration": {
    "vmDiscoveryEnabled": true,
    "containerDiscoveryEnabled": true,
    "autoDeploymentEnabled": false
  }
}
BCM Module
Purpose: Baseline configuration monitoring and compliance
Features:

Server configuration scanning
Exception detection
Auto-remediation (optional)
Compliance reporting

Modes:

Discovery Only: Scans and reports exceptions
With Auto-Remediation: Automatically fixes known exceptions

Configuration Options:
json{
  "module": "BCM",
  "isSelected": true,
  "configuration": {
    "bcmMode": "WITH_AUTO_REMEDIATION",
    "bcmExceptionTypes": ["file_permissions", "service_config"]
  }
}
EOVS Module
Purpose: Vulnerability scanning and management
Features:

Vulnerability assessment from existing inventory
Severity categorization
User-acknowledged remediation workflow
Risk reporting

Modes:

Discovery Only: Reports vulnerabilities
With User-Acknowledged Remediation: Requires approval for fixes

Configuration Options:
json{
  "module": "EOVS",
  "isSelected": true,
  "configuration": {
    "eovsMode": "WITH_USER_ACKNOWLEDGED_REMEDIATION",
    "eovsChecks": ["CVE_CHECK", "PATCH_LEVEL"]
  }
}
Onboarding Flow
Execution Order
mermaidgraph TD
    A[CSI Onboarding] --> B[Parallel Sync Operations]
    B --> B1[Server Sync from Drift]
    B --> B2[Certificate Sync from CAS]
    B --> B3[Repository Retrieval]
    
    B1 --> C[Ansible Connection Test]
    B2 --> C
    B3 --> D[Prescan Certificates]
    
    C --> E{Connection Success?}
    E -->|Yes| F[Module Operations]
    E -->|No| G[Log Failed Servers]
    
    F --> F1[CLM: VM/Container Discovery]
    F --> F2[BCM: Configuration Scan]
    F --> F3[EOVS: Vulnerability Assessment]
    
    F1 --> H[Generate Report]
    F2 --> H
    F3 --> H
    
    H --> I[Send Email Notification]
Sync Operations Detail
1. Server Sync from Drift

Fetches server inventory from drift system
Stores server metadata in MongoDB
Marks servers for connection testing

2. Certificate Sync from CAS

Retrieves certificate data from Certificate Authority Service
Categorizes certificates (VM/Container)
Stores in appropriate storage (MongoDB/Vault)

3. Repository Sync

Clones configured repositories
Scans for certificate files
Indexes certificate locations

Connection Testing

Tests Ansible connectivity to each server
Only servers with successful connections proceed to discovery
Failed connections are logged for manual intervention

API Reference
Core Endpoints
CSI Onboarding
httpPOST /api/v1/clm/csi/onboard
Content-Type: application/json

{
  "csiId": "CSI123",
  "requestorSoeid": "abc123",
  "emailDl": "team-dl@company.com",
  "email": "contact@company.com",
  "preferredRenewal": "AUTO",
  "notificationEmailList": ["notify1@company.com"],
  "escalationMatrix": ["escalate@company.com"],
  "bitbucketProjectUrl": "https://bitbucket.company.com/project/repo",
  "githubProjectUrl": "https://github.com/company/repo",
  "moduleSubscriptions": [
    {
      "module": "CLM",
      "isSelected": true,
      "configuration": {
        "vmDiscoveryEnabled": true,
        "containerDiscoveryEnabled": true,
        "autoDeploymentEnabled": false
      }
    },
    {
      "module": "BCM",
      "isSelected": true,
      "configuration": {
        "bcmMode": "DISCOVERY_ONLY"
      }
    },
    {
      "module": "EOVS",
      "isSelected": false
    }
  ]
}
Get Sync Status
httpGET /api/v1/clm/csi/{csiId}/sync-status
Response:
json{
  "csiId": "CSI123",
  "serverSyncStatus": {
    "state": "COMPLETED",
    "lastSuccess": "2025-09-22T10:00:00",
    "recordsProcessed": 50
  },
  "certificateSyncStatus": {
    "state": "COMPLETED",
    "recordsProcessed": 25
  },
  "repositorySyncStatus": {
    "state": "COMPLETED",
    "recordsProcessed": 3
  },
  "canProceedWithDiscovery": true
}
Execute Module Operation
httpPOST /api/v1/clm/csi/{csiId}/module/{moduleType}/execute
Get Consolidated Report
httpGET /api/v1/clm/csi/{csiId}/consolidated-report?format=JSON
Module-Specific Endpoints
BCM Remediation
httpPOST /api/v1/clm/csi/{csiId}/bcm/remediate
EOVS Acknowledgment
httpPOST /api/v1/clm/csi/{csiId}/eovs/acknowledge
Job Management
Get Job Status
httpGET /api/v1/clm/jobs/{jobId}/status
Perform Adhoc Scan
httpPOST /api/v1/clm/csi/{csiId}/adhoc-scan
Integration Points
External Systems
1. Drift API

Purpose: Server inventory management
Authentication: API key based
Endpoints:

GET /api/servers/by-csi/{csiId}


Data Format: JSON

2. CAS (Certificate Authority Service)

Purpose: Certificate repository
Authentication: OAuth 2.0
Endpoints:

GET /api/certificates/csi/{csiId}


Data Format: JSON with X.509 certificate data

3. Ansible

Purpose: Server operations and scanning
Authentication: SSH key based
Playbooks:

vm_clm_scan.yml: VM certificate discovery
bcm_scan.yml: Configuration compliance check
send_notification_email.yml: Email delivery



4. Vault

Purpose: Container certificate storage
Authentication: Token based
Paths:

/v1/secret/clm/{csiId}/certificates



5. Git Repositories

Bitbucket: REST API v2.0
GitHub: GraphQL and REST API v3
Operations: Clone, commit, pull request creation

Database Schema
Collections
csi_details:

Stores CSI configuration
Module subscriptions
Sync status tracking

clm_jobs:

Job execution tracking
State management
Dependencies

clm_reports:

Generated reports
Multi-module results
3-month retention

servers:

Server inventory from drift
Connection status
Metadata

Configuration Guide
Application Properties
yamlspring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/clm
      
clm:
  ansible:
    base-url: http://ansible-api:8080
    timeout: 300s
  
  vault:
    address: http://vault:8200
    token: ${VAULT_TOKEN}
  
  cas:
    base-url: http://cas-api:8080
    client-id: ${CAS_CLIENT_ID}
    
  drift:
    base-url: http://drift-api:8080
    api-key: ${DRIFT_API_KEY}
    
  batch:
    schedule: "0 0 2 * * SUN"  # Weekly on Sunday 2 AM
    max-batch-size: 100
    
  retention:
    reports: 90d
    jobs: 30d
Environment Variables
bash# Required
VAULT_TOKEN=your-vault-token
CAS_CLIENT_ID=cas-client-id
DRIFT_API_KEY=drift-api-key
MONGODB_URI=mongodb://user:pass@host:port/db

# Optional
ANSIBLE_TIMEOUT=300
BATCH_SIZE=50
REPORT_RETENTION_DAYS=90
Deployment
ECS Deployment
Task Definition
json{
  "family": "clm-service",
  "taskRoleArn": "arn:aws:iam::account:role/clm-task-role",
  "executionRoleArn": "arn:aws:iam::account:role/clm-execution-role",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "clm-api",
      "image": "clm-service:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "SPRING_PROFILES_ACTIVE",
          "value": "ecs"
        }
      ],
      "secrets": [
        {
          "name": "VAULT_TOKEN",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:clm/vault-token"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/clm-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
Service Configuration

Desired Count: 3 (for HA)
Load Balancer: Application Load Balancer
Health Check: /actuator/health
Auto Scaling: Based on CPU/Memory metrics

Database Indexes
javascript// Optimize query performance
db.csi_details.createIndex({ "csiId": 1 }, { unique: true });
db.csi_details.createIndex({ "active": 1, "lastScanDate": 1 });

db.clm_jobs.createIndex({ "jobId": 1 }, { unique: true });
db.clm_jobs.createIndex({ "csiId": 1, "status": 1 });
db.clm_jobs.createIndex({ "startTime": -1 });

db.clm_reports.createIndex({ "reportId": 1 }, { unique: true });
db.clm_reports.createIndex({ "csiId": 1, "generatedAt": -1 });
db.clm_reports.createIndex({ "expiresAt": 1 }, { expireAfterSeconds: 0 });

db.servers.createIndex({ "csiId": 1 });
db.servers.createIndex({ "serverId": 1 }, { unique: true });
Troubleshooting
Common Issues
1. Sync Failures
Symptom: Server/Certificate sync stuck in FAILED state
Causes:

Drift/CAS API unavailable
Authentication issues
Network connectivity

Resolution:
bash# Check sync status
GET /api/v1/clm/csi/{csiId}/sync-status

# Retry sync manually
POST /api/v1/clm/csi/{csiId}/sync/retry
2. Connection Test Failures
Symptom: Ansible connection tests failing
Causes:

SSH key issues
Firewall rules
Server offline

Resolution:

Verify SSH keys in Ansible
Check network connectivity
Review server status in drift

3. Module Execution Errors
Symptom: Module operations failing
Debug Steps:

Check job status: GET /api/v1/clm/jobs/{jobId}/status
Review job steps for specific failures
Check module configuration
Verify prerequisites (sync completion)

Monitoring
Key Metrics

Sync Success Rate: Track sync failures
Job Completion Time: Monitor performance
Connection Success Rate: Identify problematic servers
Module Error Rate: Track module-specific issues

Health Checks
httpGET /actuator/health
GET /actuator/health/custom/sync
GET /actuator/health/custom/jobs
Logging

Log Levels: Configure per package
Structured Logging: JSON format for ELK
Correlation IDs: Track requests across services

Support
For issues or questions:

Check documentation first
Review logs for error details
Contact CLM support team
Include CSI ID and Job ID in tickets

Future Enhancements
Planned Features

Additional Modules:

Secrets Management Module
Compliance Reporting Module


Enhanced Automation:

ML-based certificate categorization
Predictive renewal scheduling


Integration Expansion:

ServiceNow integration
Slack notifications


UI Improvements:

Real-time job progress
Interactive dashboards




This comprehensive implementation provides:

1. **Modular architecture** supporting CLM, BCM, and EOVS
2. **Proper execution sequencing** with sync operations first
3. **Flexible module configuration** during onboarding
4. **Comprehensive reporting** across all modules
5. **Styled HTML emails** using Thymeleaf
6. **Extensible design** for future modules
7. **Complete documentation** for operations and troubleshooting





package com.citi.cert_management.container.service;

import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.util.*;

/**
 * Service for handling email notifications via Ansible playbooks
 * Integrates with existing notification infrastructure
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class NotificationService {
    
    private final AnsibleService ansibleService;
    private final EmailTemplateService templateService;
    private final ClmReportRepository reportRepository;
    
    /**
     * Sends job completion notification with CLM report
     * 
     * @param job Completed CLM job
     */
    public void sendJobCompletionNotification(ClmJob job) {
        log.info("Sending completion notification for job: {}", job.getJobId());
        
        try {
            // Get report
            ClmReport report = reportRepository.findByJobId(job.getJobId())
                .orElseThrow(() -> new ReportNotFoundException("Report not found for job: " + job.getJobId()));
            
            // Get CSI details
            CsiDetails csi = csiRepository.findByCsiId(job.getCsiId())
                .orElseThrow(() -> new CsiNotFoundException("CSI not found: " + job.getCsiId()));
            
            // Generate email content
            String htmlContent = templateService.generateReportEmail(report, csi);
            
            // Prepare email parameters
            Map<String, Object> emailParams = new HashMap<>();
            emailParams.put("to", csi.getEmailDl());
            emailParams.put("cc", String.join(",", csi.getNotificationEmailList()));
            emailParams.put("subject", generateSubject(job, report));
            emailParams.put("body", htmlContent);
            emailParams.put("contentType", "text/html");
            
            // Send via Ansible playbook
            sendEmail(emailParams);
            
            // Mark report as emailed
            report.setEmailSent(true);
            report.setEmailSentAt(LocalDateTime.now());
            reportRepository.save(report);
            
        } catch (Exception e) {
            log.error("Failed to send completion notification for job: {}", job.getJobId(), e);
        }
    }
    
    /**
     * Sends escalation email for unresolved action items
     * 
     * @param csi CSI details
     * @param unresolvedItems List of unresolved action items
     */
    public void sendEscalationEmail(CsiDetails csi, List<ActionItem> unresolvedItems) {
        log.info("Sending escalation email for CSI: {} with {} unresolved items", 
                csi.getCsiId(), unresolvedItems.size());
        
        try {
            // Generate escalation email content
            String htmlContent = templateService.generateEscalationEmail(csi, unresolvedItems);
            
            // Prepare recipients - include escalation matrix
            List<String> toRecipients = new ArrayList<>();
            toRecipients.add(csi.getEmailDl());
            toRecipients.addAll(csi.getEscalationMatrix());
            
            Map<String, Object> emailParams = new HashMap<>();
            emailParams.put("to", String.join(",", toRecipients));
            emailParams.put("cc", String.join(",", csi.getNotificationEmailList()));
            emailParams.put("subject", "ESCALATION: Unresolved CLM Action Items - " + csi.getCsiId());
            emailParams.put("body", htmlContent);
            emailParams.put("contentType", "text/html");
            emailParams.put("priority", "high");
            
            // Send via Ansible playbook
            sendEmail(emailParams);
            
        } catch (Exception e) {
            log.error("Failed to send escalation email for CSI: {}", csi.getCsiId(), e);
        }
    }
    
    /**
     * Sends critical alert notification
     */
    public void sendCriticalAlert(CsiDetails csi, String alertMessage, ActionItem criticalItem) {
        if (!csi.isCriticalAlertsOnly()) {
            return;
        }
        
        log.info("Sending critical alert for CSI: {}", csi.getCsiId());
        
        Map<String, Object> emailParams = new HashMap<>();
        emailParams.put("to", csi.getEmailDl());
        emailParams.put("cc", String.join(",", csi.getNotificationEmailList()));
        emailParams.put("subject", "CRITICAL: " + alertMessage + " - " + csi.getCsiId());
        emailParams.put("body", generateCriticalAlertBody(csi, criticalItem));
        emailParams.put("contentType", "text/html");
        emailParams.put("priority", "urgent");
        
        sendEmail(emailParams);
    }
    
    /**
     * Core email sending method using Ansible playbook
     * Based on the attached notification service pattern
     */
    private void sendEmail(Map<String, Object> params) {
        try {
            // Prepare Ansible playbook parameters
            Map<String, Object> playbookParams = new HashMap<>();
            playbookParams.put("email_to", params.get("to"));
            playbookParams.put("email_cc", params.get("cc"));
            playbookParams.put("email_subject", params.get("subject"));
            playbookParams.put("email_body", params.get("body"));
            playbookParams.put("email_content_type", params.get("contentType"));
            
            // Execute Ansible playbook
            AnsibleResponse response = ansibleService.executePlaybook(
                "send_notification_email.yml", 
                playbookParams
            );
            
            if (!response.isSuccess()) {
                throw new NotificationException("Ansible playbook execution failed: " + 
                    response.getErrorMessage());
            }
            
            log.info("Email sent successfully via Ansible playbook");
            
        } catch (Exception e) {
            log.error("Failed to send email notification", e);
            throw new NotificationException("Email sending failed", e);
        }
    }
    
    private String generateSubject(ClmJob job, ClmReport report) {
        String prefix = job.getJobType() == JobType.ONBOARDING_SCAN ? "Onboarding Complete" : "CLM Report";
        int actionCount = report.getActionItems().size();
        
        if (actionCount > 0) {
            return String.format("%s - %s - %d Action Items Required", 
                prefix, job.getCsiId(), actionCount);
        }
        
        return String.format("%s - %s", prefix, job.getCsiId());
    }
}


package com.citi.cert_management.container.service;

import org.springframework.stereotype.Service;
import org.thymeleaf.TemplateEngine;
import org.thymeleaf.context.Context;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.util.*;

/**
 * Service for generating HTML email content for CLM reports
 * Uses Thymeleaf templates for formatting
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class EmailTemplateService {
    
    private final TemplateEngine templateEngine;
    
    /**
     * Generates HTML email content for CLM report
     * 
     * @param report CLM report data
     * @param csi CSI details
     * @return HTML content for email
     */
    public String generateReportEmail(ClmReport report, CsiDetails csi) {
        Context context = new Context();
        
        // Set basic information
        context.setVariable("csiId", csi.getCsiId());
        context.setVariable("csiName", csi.getCsiName());
        context.setVariable("reportDate", report.getGeneratedAt());
        context.setVariable("reportType", report.getReportType());
        
        // Set discovery results
        context.setVariable("vmDiscovery", report.getVmDiscoveryResults());
        context.setVariable("containerDiscovery", report.getContainerDiscoveryResults());
        
        // Set certificate inventory
        context.setVariable("inventory", report.getInventory());
        context.setVariable("expiringCerts", getExpiringCertificates(report));
        
        // Set assessment results
        context.setVariable("impactAssessment", report.getImpactAssessment());
        context.setVariable("riskAssessment", report.getRiskAssessment());
        
        // Set action items
        context.setVariable("actionItems", report.getActionItems());
        context.setVariable("criticalActions", getCriticalActions(report));
        
        // Generate summary statistics
        context.setVariable("summary", generateSummary(report));
        
        return templateEngine.process("clm-report-email", context);
    }
    
    /**
     * Generates escalation email for unresolved items
     * 
     * @param csi CSI details
     * @param unresolvedItems List of unresolved action items
     * @return HTML content for escalation email
     */
    public String generateEscalationEmail(CsiDetails csi, List<ActionItem> unresolvedItems) {
        Context context = new Context();
        
        context.setVariable("csiId", csi.getCsiId());
        context.setVariable("csiName", csi.getCsiName());
        context.setVariable("escalationDate", LocalDateTime.now());
        context.setVariable("unresolvedItems", unresolvedItems);
        context.setVariable("itemCount", unresolvedItems.size());
        context.setVariable("oldestItem", getOldestItem(unresolvedItems));
        
        // Group items by type
        Map<ActionType, List<ActionItem>> itemsByType = unresolvedItems.stream()
            .collect(Collectors.groupingBy(ActionItem::getType));
        context.setVariable("itemsByType", itemsByType);
        
        return templateEngine.process("escalation-email", context);
    }
    
    /**
     * Generates onboarding completion email
     */
    public String generateOnboardingEmail(CsiDetails csi, ClmReport report) {
        Context context = new Context();
        
        context.setVariable("csi", csi);
        context.setVariable("onboardingDate", LocalDateTime.now());
        context.setVariable("discoveryResults", combineDiscoveryResults(report));
        context.setVariable("nextSteps", generateNextSteps(csi, report));
        
        return templateEngine.process("onboarding-complete-email", context);
    }
    
    private Map<String, Object> generateSummary(ClmReport report) {
        Map<String, Object> summary = new HashMap<>();
        
        // Certificate counts
        summary.put("totalCertificates", report.getInventory().getTotalCertificates());
        summary.put("expiringIn30Days", report.getInventory().getCertsByExpiryCategory().get("30_DAYS"));
        summary.put("expiringIn60Days", report.getInventory().getCertsByExpiryCategory().get("60_DAYS"));
        summary.put("expiringIn90Days", report.getInventory().getCertsByExpiryCategory().get("90_DAYS"));
        
        // Discovery statistics
        summary.put("vmServersScanned", report.getVmDiscoveryResults().getTotalServersScanned());
        summary.put("containersScanned", report.getContainerDiscoveryResults().getTotalServersScanned());
        
        // Action items
        long criticalActions = report.getActionItems().stream()
            .filter(item -> item.getPriority() == Priority.CRITICAL)
            .count();
        summary.put("criticalActions", criticalActions);
        summary.put("totalActions", report.getActionItems().size());
        
        return summary;
    }
}


@Data
public class DiscoveryResults {
    private int totalServersScanned;
    private int serversWithCerts;
    private int totalCertsFound;
    private Map<String, Integer> certsByType;
    private List<CertificateLocation> locations;
    private List<String> scanErrors;
    private Map<String, Object> metadata;
}

/**
 * Certificate inventory details
 */
@Data
public class CertificateInventory {
    private int totalCertificates;
    private Map<String, Integer> certsByExpiryCategory;
    private List<CertificateDetail> certificates;
    private int truststoreExpiry;
    private Map<String, Integer> certsByEnvironment;
    private Map<String, Integer> certsByPurpose;
}


public enum JobStatus {
    PENDING, RUNNING, COMPLETED, FAILED, CANCELLED, RETRY
}

public enum StepStatus {
    PENDING, IN_PROGRESS, COMPLETED, FAILED, SKIPPED
}

public enum TriggerType {
    MANUAL, SCHEDULED, API, ONBOARDING
}

public enum ReportType {
    ONBOARDING, WEEKLY, ADHOC, RENEWAL
}

public enum ReportFormat {
    JSON, HTML
}

public enum ActionType {
    CERTIFICATE_EXPIRING, CONNECTION_FAILED, ANSIBLE_FAILED, 
    CONTAINER_NOT_FOUND, RISK_IDENTIFIED, MANUAL_RENEWAL_NEEDED
}

public enum Priority {
    CRITICAL, HIGH, MEDIUM, LOW
}

public enum ActionStatus {
    OPEN, IN_PROGRESS, RESOLVED, ESCALATED
}

