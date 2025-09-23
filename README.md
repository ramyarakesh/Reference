CSI Onboarding and Report Generation System - Complete Implementation
Architecture Overview
The system is designed to handle long-running asynchronous jobs with proper tracking and reporting. It consists of:

Job orchestration with state management
Async job execution with progress tracking
Daily report generation
Weekly sync cycles
Email notifications


package com.citi.cert_management.entity;

import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import javax.validation.constraints.NotNull;
import java.util.Date;
import java.util.List;

@Document(collection = "CSIDetails")
@Data
public class CsiDetails {
    @Id
    private String id;
    
    @NotNull
    private Integer csi;
    
    @NotNull
    private String email;
    
    @NotNull
    private String ownerTeamEmailDlName;
    
    // Basic CSI Information - from archcenter
    private String csiManagerSOEID;
    private Long csiManagerGEID;
    private String businessOwner;
    private String businessOwnerId;
    private String cioName;
    private String cioId;
    private String appName;
    private String cioDescription;
    private String director;
    private String directorId;
    
    // Optional parameters
    private String cmpRequesterSoeId;
    private List<String> notificationEmails;
    private List<String> escalationContacts;
    private String primaryContact;
    
    // Certificate preferences
    private String preferredRenewalProvider; // PP, CMP, ACM
    private Boolean autoRenewalEnabled;
    private Integer customExpiryThresholdDays;
    
    // Repository configuration
    private List<String> githubRepositories;
    private List<String> bitbucketProjectUrls;
    private List<String> bitbucketRepositories;
    
    // Notification preferences
    private boolean dailyDigestEnabled;
    private boolean criticalAlertsOnly;
    
    // Audit fields
    private Date onboardedAt;
    private String onboardedBy;
    private Date lastModified;
    private boolean active;
    
    // Module Subscriptions
    private List<ModuleSubscription> moduleSubscriptions;
    
    // Sync Status
    private SyncStatus serverSyncStatus;
    private SyncStatus certificateSyncStatus;
    private SyncStatus repositorySyncStatus;
    private Date lastSuccessfulSync;
    
    // Job tracking
    private String currentJobId;
    private JobStatus jobStatus;
    private Date lastJobExecutionTime;
    private Date lastReportGenerationTime;
    private String lastReportId;
}

package com.citi.cert_management.entity;

import lombok.Data;
import java.time.LocalDateTime;

@Data
public class ModuleSubscription {
    private ModuleType module;
    private boolean isSelected;
    private ModuleConfiguration configuration;
    private LocalDateTime subscribedAt;
    private String subscribedBy;
}


package com.citi.cert_management.entity;

import lombok.Data;
import java.util.List;

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
    private List<String> includePaths;
    private List<String> excludePaths;
    private boolean eovsUserAcknowledgmentRequired;
    private List<String> eovsChecks;
}

package com.citi.cert_management.entity;

import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Map;

@Document(collection = "CSIJobExecutions")
@Data
public class CsiJobExecution {
    @Id
    private String jobId;
    private Integer csi;
    private JobType jobType; // ONBOARDING, WEEKLY_SYNC, MANUAL_SYNC
    private JobStatus overallStatus;
    private Date startTime;
    private Date endTime;
    private Date lastUpdated;
    private List<JobStep> steps = new ArrayList<>();
    private Map<String, Object> metadata;
    private String triggeredBy;
    private List<String> errors = new ArrayList<>();
    private List<String> warnings = new ArrayList<>();
    private Integer completionPercentage = 0;
    private boolean reportGenerationEnabled = false;
    private Date reportGenerationStartDate;
}

package com.citi.cert_management.entity;

import lombok.Data;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Map;

@Data
public class JobStep {
    private String stepId;
    private StepType stepType;
    private StepStatus status = StepStatus.PENDING;
    private Date startTime;
    private Date endTime;
    private Date lastChecked;
    private Integer progress = 0; // 0-100
    private String message;
    private Map<String, Object> results;
    private List<String> subTaskIds = new ArrayList<>(); // For tracking ansible jobs
    private List<String> dependsOn = new ArrayList<>(); // Step dependencies
    private Integer retryCount = 0;
    private Integer maxRetries = 3;
}

package com.citi.cert_management.entity;

import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;

@Document(collection = "CSIReports")
@Data
public class CsiReport {
    @Id
    private String reportId;
    private Integer csi;
    private String jobId;
    private Date generatedAt;
    private JobStatus jobStatus;
    private Integer completionPercentage;
    private boolean isFinalReport;
    
    // Server statistics
    private Integer totalServers = 0;
    private Integer vmCount = 0;
    private Integer windowsCount = 0;
    private Integer containersCount = 0;
    private String vmScanStatus;
    private String windowsScanStatus;
    private Integer successfulScans = 0;
    private Integer failedScans = 0;
    
    // Certificate statistics
    private Integer totalCertificates = 0;
    private Integer expiredCerts = 0;
    private Integer expiringIn30Days = 0;
    private Integer expiringIn60Days = 0;
    private Integer expiringIn90Days = 0;
    private Integer trustStoreExpiry = 0;
    
    // Repository statistics
    private Integer totalRepositories = 0;
    private Integer scannedRepositories = 0;
    private Integer certificatesInCode = 0;
    
    // BCM statistics
    private Integer bcmExceptions = 0;
    private Integer criticalExceptions = 0;
    
    // Impact assessment
    private List<String> highRiskItems = new ArrayList<>();
    private List<String> mediumRiskItems = new ArrayList<>();
    private List<String> actionItems = new ArrayList<>();
    
    // Progress details
    private List<StepProgress> stepProgressList = new ArrayList<>();
}

package com.citi.cert_management.enums;

public enum JobType {
    ONBOARDING, WEEKLY_SYNC, MANUAL_SYNC
}

public enum JobStatus {
    PENDING, IN_PROGRESS, COMPLETED, FAILED, PARTIALLY_COMPLETED
}

public enum StepType {
    SERVER_SYNC, 
    CERTIFICATE_SYNC, 
    REPOSITORY_SYNC,
    ANSIBLE_CONNECTIVITY_TEST, 
    VM_CERT_SCAN, 
    WINDOWS_CERT_SCAN,
    BCM_EXCEPTION_SCAN, 
    REPO_CERT_USAGE_SCAN, 
    IMPACT_ASSESSMENT
}

public enum StepStatus {
    PENDING, IN_PROGRESS, COMPLETED, FAILED, PARTIALLY_COMPLETED, SKIPPED
}

public enum SyncStatus {
    NOT_STARTED, IN_PROGRESS, COMPLETED, FAILED
}

public enum ModuleType {
    CLM, BCM, EOVS
}

public enum BcmMode {
    MONITOR_ONLY, AUTO_REMEDIATE
}

public enum EovsMode {
    SCAN_ONLY, SCAN_AND_NOTIFY, AUTO_REMEDIATE
}


package com.citi.cert_management.service;

import com.citi.cert_management.entity.*;
import com.citi.cert_management.enums.*;
import com.citi.cert_management.repository.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@Service
@Slf4j
public class CsiJobOrchestrator {
    
    @Autowired
    private CsiJobExecutionRepository jobExecutionRepository;
    
    @Autowired
    private CsiDetailsRepository csiDetailsRepository;
    
    @Autowired
    private CasIntegrationService casIntegrationService;
    
    @Autowired
    private VaultIntegrationService vaultIntegrationService;
    
    @Autowired
    private BitbucketIntegrationService bitbucketIntegrationService;
    
    @Autowired
    private AnsibleIntegrationService ansibleIntegrationService;
    
    @Autowired
    private CertificateScanningService certificateScanningService;
    
    @Autowired
    private ImpactAssessmentService impactAssessmentService;
    
    @Autowired
    private JobProgressTracker jobProgressTracker;
    
    private final ExecutorService executorService = Executors.newFixedThreadPool(10);
    
    /**
     * Initiates CSI onboarding job
     */
    public String initiateOnboardingJob(Integer csi, String triggeredBy) {
        log.info("Initiating onboarding job for CSI: {}", csi);
        
        // Create job execution record
        CsiJobExecution jobExecution = createJobExecution(csi, JobType.ONBOARDING, triggeredBy);
        
        // Initialize job steps
        initializeJobSteps(jobExecution);
        
        // Save initial job state
        jobExecutionRepository.save(jobExecution);
        
        // Update CSI details
        CsiDetails csiDetails = csiDetailsRepository.findByCsi(csi);
        if (csiDetails != null) {
            csiDetails.setCurrentJobId(jobExecution.getJobId());
            csiDetails.setJobStatus(JobStatus.IN_PROGRESS);
            csiDetails.setLastJobExecutionTime(new Date());
            csiDetailsRepository.save(csiDetails);
        }
        
        // Start job execution asynchronously
        executeJobAsync(jobExecution.getJobId());
        
        return jobExecution.getJobId();
    }
    
    /**
     * Executes job asynchronously
     */
    @Async
    public void executeJobAsync(String jobId) {
        try {
            CsiJobExecution jobExecution = jobExecutionRepository.findById(jobId)
                .orElseThrow(() -> new RuntimeException("Job not found: " + jobId));
            
            jobExecution.setStartTime(new Date());
            jobExecution.setOverallStatus(JobStatus.IN_PROGRESS);
            jobExecutionRepository.save(jobExecution);
            
            // Phase 1: Execute parallel sync operations
            executePhase1SyncOperations(jobExecution);
            
            // Phase 2: Execute dependent operations (will check completion in background)
            schedulePhase2Operations(jobExecution);
            
            // Enable report generation after 2 days
            scheduleReportGeneration(jobExecution);
            
        } catch (Exception e) {
            log.error("Error executing job: " + jobId, e);
            handleJobError(jobId, e);
        }
    }
    
    /**
     * Phase 1: Execute sync operations in parallel
     */
    private void executePhase1SyncOperations(CsiJobExecution jobExecution) {
        log.info("Starting Phase 1 sync operations for job: {}", jobExecution.getJobId());
        
        // Execute server sync
        CompletableFuture<Void> serverSyncFuture = CompletableFuture.runAsync(() -> {
            JobStep serverSyncStep = findStepByType(jobExecution, StepType.SERVER_SYNC);
            executeServerSync(jobExecution, serverSyncStep);
        }, executorService);
        
        // Execute certificate sync
        CompletableFuture<Void> certSyncFuture = CompletableFuture.runAsync(() -> {
            JobStep certSyncStep = findStepByType(jobExecution, StepType.CERTIFICATE_SYNC);
            executeCertificateSync(jobExecution, certSyncStep);
        }, executorService);
        
        // Execute repository sync
        CompletableFuture<Void> repoSyncFuture = CompletableFuture.runAsync(() -> {
            JobStep repoSyncStep = findStepByType(jobExecution, StepType.REPOSITORY_SYNC);
            executeRepositorySync(jobExecution, repoSyncStep);
        }, executorService);
        
        // Don't wait for completion - let them run asynchronously
    }
    
    /**
     * Execute server sync from CAS
     */
    private void executeServerSync(CsiJobExecution jobExecution, JobStep step) {
        try {
            step.setStatus(StepStatus.IN_PROGRESS);
            step.setStartTime(new Date());
            step.setMessage("Initiating server sync from CAS");
            updateJobStep(jobExecution, step);
            
            // Call CAS integration service
            String syncJobId = casIntegrationService.initiateServerSync(jobExecution.getCsi());
            step.getSubTaskIds().add(syncJobId);
            
            // Update progress
            step.setProgress(50);
            step.setMessage("Server sync initiated. Job ID: " + syncJobId);
            updateJobStep(jobExecution, step);
            
            // Poll for completion (this could be done in a separate scheduled job)
            boolean completed = pollForCompletion(() -> 
                casIntegrationService.checkSyncStatus(syncJobId), 
                30, // max attempts
                60000 // 1 minute interval
            );
            
            if (completed) {
                step.setStatus(StepStatus.COMPLETED);
                step.setProgress(100);
                step.setMessage("Server sync completed successfully");
                
                // Retrieve sync results
                Map<String, Object> results = casIntegrationService.getSyncResults(syncJobId);
                step.setResults(results);
            } else {
                step.setStatus(StepStatus.PARTIALLY_COMPLETED);
                step.setProgress(75);
                step.setMessage("Server sync timed out - will check in background");
            }
            
        } catch (Exception e) {
            log.error("Error in server sync for job: " + jobExecution.getJobId(), e);
            step.setStatus(StepStatus.FAILED);
            step.setMessage("Server sync failed: " + e.getMessage());
        } finally {
            step.setEndTime(new Date());
            updateJobStep(jobExecution, step);
        }
    }
    
    /**
     * Execute certificate sync from Vault
     */
    private void executeCertificateSync(CsiJobExecution jobExecution, JobStep step) {
        try {
            step.setStatus(StepStatus.IN_PROGRESS);
            step.setStartTime(new Date());
            step.setMessage("Initiating certificate sync from Vault");
            updateJobStep(jobExecution, step);
            
            // Call Vault integration service
            Map<String, Object> syncResult = vaultIntegrationService.syncCertificates(jobExecution.getCsi());
            
            step.setStatus(StepStatus.COMPLETED);
            step.setProgress(100);
            step.setResults(syncResult);
            step.setMessage("Certificate sync completed. Synced " + 
                syncResult.getOrDefault("certificateCount", 0) + " certificates");
            
        } catch (Exception e) {
            log.error("Error in certificate sync for job: " + jobExecution.getJobId(), e);
            step.setStatus(StepStatus.FAILED);
            step.setMessage("Certificate sync failed: " + e.getMessage());
        } finally {
            step.setEndTime(new Date());
            updateJobStep(jobExecution, step);
        }
    }
    
    /**
     * Execute repository sync from Bitbucket/GitHub
     */
    private void executeRepositorySync(CsiJobExecution jobExecution, JobStep step) {
        try {
            step.setStatus(StepStatus.IN_PROGRESS);
            step.setStartTime(new Date());
            step.setMessage("Initiating repository sync");
            updateJobStep(jobExecution, step);
            
            CsiDetails csiDetails = csiDetailsRepository.findByCsi(jobExecution.getCsi());
            
            List<String> allRepos = new ArrayList<>();
            allRepos.addAll(csiDetails.getBitbucketRepositories() != null ? 
                csiDetails.getBitbucketRepositories() : Collections.emptyList());
            allRepos.addAll(csiDetails.getGithubRepositories() != null ? 
                csiDetails.getGithubRepositories() : Collections.emptyList());
            
            Map<String, Object> syncResult = bitbucketIntegrationService.syncRepositories(
                jobExecution.getCsi(), allRepos);
            
            step.setStatus(StepStatus.COMPLETED);
            step.setProgress(100);
            step.setResults(syncResult);
            step.setMessage("Repository sync completed. Synced " + 
                syncResult.getOrDefault("repositoryCount", 0) + " repositories");
            
        } catch (Exception e) {
            log.error("Error in repository sync for job: " + jobExecution.getJobId(), e);
            step.setStatus(StepStatus.FAILED);
            step.setMessage("Repository sync failed: " + e.getMessage());
        } finally {
            step.setEndTime(new Date());
            updateJobStep(jobExecution, step);
        }
    }
    
    /**
     * Schedule Phase 2 operations to run after dependencies are met
     */
    private void schedulePhase2Operations(CsiJobExecution jobExecution) {
        // Schedule ansible connectivity test (depends on server sync)
        jobProgressTracker.scheduleStepExecution(
            jobExecution.getJobId(),
            StepType.ANSIBLE_CONNECTIVITY_TEST,
            () -> executeAnsibleConnectivityTest(jobExecution)
        );
        
        // Schedule repository scans (depends on cert and repo sync)
        jobProgressTracker.scheduleStepExecution(
            jobExecution.getJobId(),
            StepType.REPO_CERT_USAGE_SCAN,
            () -> executeRepositoryScans(jobExecution)
        );
    }
    
    /**
     * Execute Ansible connectivity test
     */
    private void executeAnsibleConnectivityTest(CsiJobExecution jobExecution) {
        JobStep step = findStepByType(jobExecution, StepType.ANSIBLE_CONNECTIVITY_TEST);
        
        try {
            // Check if server sync is completed
            JobStep serverSyncStep = findStepByType(jobExecution, StepType.SERVER_SYNC);
            if (serverSyncStep.getStatus() != StepStatus.COMPLETED) {
                step.setStatus(StepStatus.SKIPPED);
                step.setMessage("Skipped - Server sync not completed");
                updateJobStep(jobExecution, step);
                return;
            }
            
            step.setStatus(StepStatus.IN_PROGRESS);
            step.setStartTime(new Date());
            updateJobStep(jobExecution, step);
            
            // Get servers from inventory
            List<FEInventoryReportNew> servers = inventoryRepository.findByCsi(jobExecution.getCsi());
            
            // Test ansible connectivity
            Map<String, Object> connectivityResults = 
                ansibleIntegrationService.testConnectivity(servers);
            
            step.setStatus(StepStatus.COMPLETED);
            step.setProgress(100);
            step.setResults(connectivityResults);
            step.setMessage("Ansible connectivity test completed");
            
            // Schedule server scans for successful connections
            scheduleServerScans(jobExecution, connectivityResults);
            
        } catch (Exception e) {
            log.error("Error in ansible connectivity test", e);
            step.setStatus(StepStatus.FAILED);
            step.setMessage("Ansible connectivity test failed: " + e.getMessage());
        } finally {
            step.setEndTime(new Date());
            updateJobStep(jobExecution, step);
        }
    }
    
    /**
     * Schedule server scans (VM and Windows)
     */
    private void scheduleServerScans(CsiJobExecution jobExecution, 
                                   Map<String, Object> connectivityResults) {
        // Schedule VM scans
        jobProgressTracker.scheduleStepExecution(
            jobExecution.getJobId(),
            StepType.VM_CERT_SCAN,
            () -> executeVmCertScan(jobExecution, connectivityResults)
        );
        
        // Schedule Windows scans
        jobProgressTracker.scheduleStepExecution(
            jobExecution.getJobId(),
            StepType.WINDOWS_CERT_SCAN,
            () -> executeWindowsCertScan(jobExecution, connectivityResults)
        );
        
        // Schedule BCM exception scan
        jobProgressTracker.scheduleStepExecution(
            jobExecution.getJobId(),
            StepType.BCM_EXCEPTION_SCAN,
            () -> executeBcmExceptionScan(jobExecution, connectivityResults)
        );
    }
    
    /**
     * Execute VM certificate scan
     */
    private void executeVmCertScan(CsiJobExecution jobExecution, 
                                  Map<String, Object> connectivityResults) {
        JobStep step = findStepByType(jobExecution, StepType.VM_CERT_SCAN);
        
        try {
            step.setStatus(StepStatus.IN_PROGRESS);
            step.setStartTime(new Date());
            updateJobStep(jobExecution, step);
            
            // Get successful VM servers
            List<String> vmServers = extractSuccessfulServers(connectivityResults, "VM");
            
            if (vmServers.isEmpty()) {
                step.setStatus(StepStatus.SKIPPED);
                step.setMessage("No VM servers available for scanning");
                updateJobStep(jobExecution, step);
                return;
            }
            
            // Initiate ansible playbook for VM cert scan
            String playbookJobId = ansibleIntegrationService.initiateVmCertScan(
                jobExecution.getCsi(), vmServers);
            step.getSubTaskIds().add(playbookJobId);
            
            step.setProgress(25);
            step.setMessage("VM certificate scan initiated. Ansible job: " + playbookJobId);
            updateJobStep(jobExecution, step);
            
            // This will complete asynchronously - progress will be tracked separately
            
        } catch (Exception e) {
            log.error("Error in VM cert scan", e);
            step.setStatus(StepStatus.FAILED);
            step.setMessage("VM cert scan failed: " + e.getMessage());
            step.setEndTime(new Date());
            updateJobStep(jobExecution, step);
        }
    }
    
    /**
     * Execute repository certificate usage scan
     */
    private void executeRepositoryScans(CsiJobExecution jobExecution) {
        JobStep step = findStepByType(jobExecution, StepType.REPO_CERT_USAGE_SCAN);
        
        try {
            // Check dependencies
            JobStep certSyncStep = findStepByType(jobExecution, StepType.CERTIFICATE_SYNC);
            JobStep repoSyncStep = findStepByType(jobExecution, StepType.REPOSITORY_SYNC);
            
            if (certSyncStep.getStatus() != StepStatus.COMPLETED || 
                repoSyncStep.getStatus() != StepStatus.COMPLETED) {
                step.setStatus(StepStatus.SKIPPED);
                step.setMessage("Skipped - Dependencies not completed");
                updateJobStep(jobExecution, step);
                return;
            }
            
            step.setStatus(StepStatus.IN_PROGRESS);
            step.setStartTime(new Date());
            updateJobStep(jobExecution, step);
            
            // Scan repositories for certificate usage
            Map<String, Object> scanResults = 
                certificateScanningService.scanRepositoriesForCertUsage(jobExecution.getCsi());
            
            step.setStatus(StepStatus.COMPLETED);
            step.setProgress(100);
            step.setResults(scanResults);
            step.setMessage("Repository scan completed. Found " + 
                scanResults.getOrDefault("certificatesInCode", 0) + " certificates in code");
            
        } catch (Exception e) {
            log.error("Error in repository scan", e);
            step.setStatus(StepStatus.FAILED);
            step.setMessage("Repository scan failed: " + e.getMessage());
        } finally {
            step.setEndTime(new Date());
            updateJobStep(jobExecution, step);
        }
    }
    
    /**
     * Schedule report generation to start after 2 days
     */
    private void scheduleReportGeneration(CsiJobExecution jobExecution) {
        Calendar cal = Calendar.getInstance();
        cal.add(Calendar.DAY_OF_MONTH, 2);
        
        jobExecution.setReportGenerationEnabled(true);
        jobExecution.setReportGenerationStartDate(cal.getTime());
        jobExecutionRepository.save(jobExecution);
        
        log.info("Report generation scheduled to start on {} for job {}", 
            cal.getTime(), jobExecution.getJobId());
    }
    
    /**
     * Initialize job steps with dependencies
     */
    private void initializeJobSteps(CsiJobExecution jobExecution) {
        List<JobStep> steps = new ArrayList<>();
        
        // Phase 1 - Parallel sync operations
        steps.add(createJobStep(StepType.SERVER_SYNC, null));
        steps.add(createJobStep(StepType.CERTIFICATE_SYNC, null));
        steps.add(createJobStep(StepType.REPOSITORY_SYNC, null));
        
        // Phase 2 - Dependent operations
        steps.add(createJobStep(StepType.ANSIBLE_CONNECTIVITY_TEST, 
            Arrays.asList(StepType.SERVER_SYNC.name())));
        
        steps.add(createJobStep(StepType.REPO_CERT_USAGE_SCAN, 
            Arrays.asList(StepType.CERTIFICATE_SYNC.name(), StepType.REPOSITORY_SYNC.name())));
        
        // Phase 3 - Server scans (depends on ansible connectivity)
        steps.add(createJobStep(StepType.VM_CERT_SCAN, 
            Arrays.asList(StepType.ANSIBLE_CONNECTIVITY_TEST.name())));
        steps.add(createJobStep(StepType.WINDOWS_CERT_SCAN, 
            Arrays.asList(StepType.ANSIBLE_CONNECTIVITY_TEST.name())));
        steps.add(createJobStep(StepType.BCM_EXCEPTION_SCAN, 
            Arrays.asList(StepType.ANSIBLE_CONNECTIVITY_TEST.name())));
        
        // Phase 4 - Impact assessment (depends on all scans)
        steps.add(createJobStep(StepType.IMPACT_ASSESSMENT, 
            Arrays.asList(StepType.VM_CERT_SCAN.name(), 
                         StepType.WINDOWS_CERT_SCAN.name(),
                         StepType.REPO_CERT_USAGE_SCAN.name())));
        
        jobExecution.setSteps(steps);
    }
    
    private JobStep createJobStep(StepType type, List<String> dependencies) {
        JobStep step = new JobStep();
        step.setStepId(UUID.randomUUID().toString());
        step.setStepType(type);
        step.setStatus(StepStatus.PENDING);
        if (dependencies != null) {
            step.setDependsOn(dependencies);
        }
        return step;
    }
    
    private CsiJobExecution createJobExecution(Integer csi, JobType jobType, String triggeredBy) {
        CsiJobExecution jobExecution = new CsiJobExecution();
        jobExecution.setJobId(UUID.randomUUID().toString());
        jobExecution.setCsi(csi);
        jobExecution.setJobType(jobType);
        jobExecution.setOverallStatus(JobStatus.PENDING);
        jobExecution.setTriggeredBy(triggeredBy);
        jobExecution.setMetadata(new HashMap<>());
        return jobExecution;
    }
    
    private void updateJobStep(CsiJobExecution jobExecution, JobStep step) {
        step.setLastChecked(new Date());
        jobExecution.setLastUpdated(new Date());
        
        // Update completion percentage
        updateJobCompletionPercentage(jobExecution);
        
        jobExecutionRepository.save(jobExecution);
    }
    
    private void updateJobCompletionPercentage(CsiJobExecution jobExecution) {
        List<JobStep> steps = jobExecution.getSteps();
        if (steps.isEmpty()) {
            jobExecution.setCompletionPercentage(0);
            return;
        }
        
        int totalProgress = steps.stream()
            .mapToInt(JobStep::getProgress)
            .sum();
        
        jobExecution.setCompletionPercentage(totalProgress / steps.size());
    }
    
    private JobStep findStepByType(CsiJobExecution jobExecution, StepType type) {
        return jobExecution.getSteps().stream()
            .filter(step -> step.getStepType() == type)
            .findFirst()
            .orElseThrow(() -> new RuntimeException("Step not found: " + type));
    }
    
    private boolean pollForCompletion(Supplier<Boolean> checkFunction, 
                                    int maxAttempts, long intervalMs) {
        for (int i = 0; i < maxAttempts; i++) {
            if (checkFunction.get()) {
                return true;
            }
            try {
                Thread.sleep(intervalMs);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            }
        }
        return false;
    }
    
    private void handleJobError(String jobId, Exception e) {
        CsiJobExecution jobExecution = jobExecutionRepository.findById(jobId).orElse(null);
        if (jobExecution != null) {
            jobExecution.setOverallStatus(JobStatus.FAILED);
            jobExecution.getErrors().add(e.getMessage());
            jobExecution.setEndTime(new Date());
            jobExecutionRepository.save(jobExecution);
        }
    }
    
    @Autowired
    private FEInventoryReportNewRepository inventoryRepository;
    
    private List<String> extractSuccessfulServers(Map<String, Object> connectivityResults, 
                                                 String serverType) {
        // Extract successful servers based on type
        // Implementation depends on connectivity results structure
        return new ArrayList<>();
    }
    
    private void executeWindowsCertScan(CsiJobExecution jobExecution, 
                                      Map<String, Object> connectivityResults) {
        // Similar to VM scan but for Windows servers
    }
    
    private void executeBcmExceptionScan(CsiJobExecution jobExecution, 
                                       Map<String, Object> connectivityResults) {
        // BCM exception scanning logic
    }
}


package com.citi.cert_management.service;

import com.citi.cert_management.entity.*;
import com.citi.cert_management.enums.*;
import com.citi.cert_management.repository.CsiJobExecutionRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class JobProgressTracker {
    
    @Autowired
    private CsiJobExecutionRepository jobExecutionRepository;
    
    @Autowired
    private AnsibleIntegrationService ansibleIntegrationService;
    
    private final ScheduledExecutorService scheduler = new ScheduledThreadPoolExecutor(5);
    private final ConcurrentHashMap<String, Runnable> pendingSteps = new ConcurrentHashMap<>();
    
    /**
     * Schedule a step to be executed when dependencies are met
     */
    public void scheduleStepExecution(String jobId, StepType stepType, Runnable execution) {
        String key = jobId + "-" + stepType;
        pendingSteps.put(key, execution);
        
        // Schedule immediate check and then periodic checks
        scheduler.scheduleAtFixedRate(() -> {
            checkAndExecuteStep(jobId, stepType);
        }, 0, 5, TimeUnit.MINUTES);
    }
    
    /**
     * Check if step dependencies are met and execute if ready
     */
    private void checkAndExecuteStep(String jobId, StepType stepType) {
        try {
            CsiJobExecution jobExecution = jobExecutionRepository.findById(jobId).orElse(null);
            if (jobExecution == null) {
                log.warn("Job not found: {}", jobId);
                return;
            }
            
            JobStep step = findStepByType(jobExecution, stepType);
            if (step.getStatus() != StepStatus.PENDING) {
                // Step already processed
                removePendingStep(jobId, stepType);
                return;
            }
            
            // Check dependencies
            if (areDependenciesMet(jobExecution, step)) {
                String key = jobId + "-" + stepType;
                Runnable execution = pendingSteps.get(key);
                if (execution != null) {
                    log.info("Executing step {} for job {}", stepType, jobId);
                    execution.run();
                    removePendingStep(jobId, stepType);
                }
            }
        } catch (Exception e) {
            log.error("Error checking step execution for job {} step {}", jobId, stepType, e);
        }
    }
    
    /**
     * Check progress of long-running ansible jobs
     */
    @Scheduled(fixedDelay = 300000) // Every 5 minutes
    public void checkAnsibleJobProgress() {
        log.debug("Checking ansible job progress");
        
        // Find all jobs with ansible tasks in progress
        List<CsiJobExecution> activeJobs = jobExecutionRepository.findByOverallStatus(JobStatus.IN_PROGRESS);
        
        for (CsiJobExecution job : activeJobs) {
            for (JobStep step : job.getSteps()) {
                if (step.getStatus() == StepStatus.IN_PROGRESS && !step.getSubTaskIds().isEmpty()) {
                    checkAnsibleTaskProgress(job, step);
                }
            }
        }
    }
    
    /**
     * Check progress of specific ansible task
     */
    private void checkAnsibleTaskProgress(CsiJobExecution job, JobStep step) {
        try {
            for (String ansibleJobId : step.getSubTaskIds()) {
                Map<String, Object> status = ansibleIntegrationService.getJobStatus(ansibleJobId);
                
                String jobStatus = (String) status.get("status");
                Integer progress = (Integer) status.get("progress");
                
                if ("completed".equals(jobStatus)) {
                    step.setProgress(100);
                    step.setStatus(StepStatus.COMPLETED);
                    step.setMessage("Ansible job completed successfully");
                    step.setEndTime(new Date());
                } else if ("failed".equals(jobStatus)) {
                    step.setStatus(StepStatus.FAILED);
                    step.setMessage("Ansible job failed: " + status.get("error"));
                    step.setEndTime(new Date());
                } else {
                    // Update progress
                    if (progress != null) {
                        step.setProgress(progress);
                    }
                    step.setMessage("Ansible job in progress: " + progress + "%");
                }
                
                step.setLastChecked(new Date());
                job.setLastUpdated(new Date());
                jobExecutionRepository.save(job);
            }
        } catch (Exception e) {
            log.error("Error checking ansible job progress", e);
        }
    }
    
    /**
     * Update overall job status based on step statuses
     */
    @Scheduled(fixedDelay = 600000) // Every 10 minutes
    public void updateJobStatuses() {
        List<CsiJobExecution> activeJobs = jobExecutionRepository
            .findByOverallStatusIn(Arrays.asList(JobStatus.IN_PROGRESS, JobStatus.PENDING));
        
        for (CsiJobExecution job : activeJobs) {
            updateJobStatus(job);
        }
    }
    
    private void updateJobStatus(CsiJobExecution job) {
        List<JobStep> steps = job.getSteps();
        
        long completedCount = steps.stream()
            .filter(s -> s.getStatus() == StepStatus.COMPLETED)
            .count();
        
        long failedCount = steps.stream()
            .filter(s -> s.getStatus() == StepStatus.FAILED)
            .count();
        
        long inProgressCount = steps.stream()
            .filter(s -> s.getStatus() == StepStatus.IN_PROGRESS)
            .count();
        
        if (completedCount == steps.size()) {
            job.setOverallStatus(JobStatus.COMPLETED);
            job.setCompletionPercentage(100);
            job.setEndTime(new Date());
        } else if (failedCount > 0 && inProgressCount == 0) {
            job.setOverallStatus(JobStatus.FAILED);
        } else if (completedCount > 0 || failedCount > 0) {
            job.setOverallStatus(JobStatus.PARTIALLY_COMPLETED);
        }
        
        jobExecutionRepository.save(job);
    }
    
    private boolean areDependenciesMet(CsiJobExecution job, JobStep step) {
        if (step.getDependsOn() == null || step.getDependsOn().isEmpty()) {
            return true;
        }
        
        for (String dependency : step.getDependsOn()) {
            JobStep depStep = job.getSteps().stream()
                .filter(s -> s.getStepType().name().equals(dependency))
                .findFirst()
                .orElse(null);
            
            if (depStep == null || depStep.getStatus() != StepStatus.COMPLETED) {
                return false;
            }
        }
        
        return true;
    }
    
    private JobStep findStepByType(CsiJobExecution job, StepType type) {
        return job.getSteps().stream()
            .filter(step -> step.getStepType() == type)
            .findFirst()
            .orElse(null);
    }
    
    private void removePendingStep(String jobId, StepType stepType) {
        String key = jobId + "-" + stepType;
        pendingSteps.remove(key);
    }
}



package com.citi.cert_management.service;

import com.citi.cert_management.entity.*;
import com.citi.cert_management.enums.*;
import com.citi.cert_management.repository.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.thymeleaf.TemplateEngine;
import org.thymeleaf.context.Context;

import java.text.SimpleDateFormat;
import java.util.*;
import java.util.stream.Collectors;

@Service
@Slf4j
public class CsiReportGenerationService {
    
    @Autowired
    private CsiJobExecutionRepository jobExecutionRepository;
    
    @Autowired
    private CsiDetailsRepository csiDetailsRepository;
    
    @Autowired
    private CsiReportRepository reportRepository;
    
    @Autowired
    private FEInventoryReportNewRepository inventoryRepository;
    
    @Autowired
    private CertificateRepository certificateRepository;
    
    @Autowired
    private BcmExceptionRepository bcmExceptionRepository;
    
    @Autowired
    private TemplateEngine templateEngine;
    
    @Autowired
    private NotificationService notificationService;
    
    /**
     * Daily report generation job
     */
    @Scheduled(cron = "0 0 8 * * *") // Daily at 8 AM
    public void generateDailyReports() {
        log.info("Starting daily report generation");
        
        // Find jobs eligible for report generation
        Date currentDate = new Date();
        List<CsiJobExecution> eligibleJobs = jobExecutionRepository
            .findByReportGenerationEnabledTrueAndReportGenerationStartDateBefore(currentDate);
        
        for (CsiJobExecution job : eligibleJobs) {
            try {
                generateReportForJob(job);
            } catch (Exception e) {
                log.error("Error generating report for job: " + job.getJobId(), e);
            }
        }
    }
    
    /**
     * Generate report for specific job
     */
    public CsiReport generateReportForJob(CsiJobExecution job) {
        log.info("Generating report for job: {} CSI: {}", job.getJobId(), job.getCsi());
        
        CsiReport report = new CsiReport();
        report.setReportId(UUID.randomUUID().toString());
        report.setCsi(job.getCsi());
        report.setJobId(job.getJobId());
        report.setGeneratedAt(new Date());
        report.setJobStatus(job.getOverallStatus());
        report.setCompletionPercentage(job.getCompletionPercentage());
        
        // Populate server statistics
        populateServerStatistics(job.getCsi(), report);
        
        // Populate certificate statistics
        populateCertificateStatistics(job.getCsi(), report);
        
        // Populate repository statistics
        populateRepositoryStatistics(job, report);
        
        // Populate BCM statistics
        populateBcmStatistics(job.getCsi(), report);
        
        // Generate step progress details
        populateStepProgress(job, report);
        
        // Generate action items
        generateActionItems(job, report);
        
        // Check if job is complete
        report.setFinalReport(job.getCompletionPercentage() == 100);
        
        // Save report
        report = reportRepository.save(report);
        
        // Update CSI details
        updateCsiDetailsWithReport(job.getCsi(), report);
        
        // Send email notification
        sendReportEmail(job.getCsi(), report);
        
        return report;
    }
    
    /**
     * Populate server statistics
     */
    private void populateServerStatistics(Integer csi, CsiReport report) {
        List<FEInventoryReportNew> servers = inventoryRepository.findByCsi(csi);
        
        report.setTotalServers(servers.size());
        
        // Count by type
        report.setVmCount((int) servers.stream()
            .filter(s -> "VM".equals(s.getServerType()))
            .count());
        
        report.setWindowsCount((int) servers.stream()
            .filter(s -> "WINDOWS".equals(s.getServerType()))
            .count());
        
        report.setContainersCount((int) servers.stream()
            .filter(s -> "CONTAINER".equals(s.getServerType()))
            .count());
        
        // Scan status
        Map<String, List<FEInventoryReportNew>> scanStatusMap = servers.stream()
            .collect(Collectors.groupingBy(s -> 
                s.getScanResult() != null ? s.getScanResult().getStatus() : "NOT_SCANNED"));
        
        report.setSuccessfulScans(scanStatusMap.getOrDefault("SUCCESS", Collections.emptyList()).size());
        report.setFailedScans(scanStatusMap.getOrDefault("FAILED", Collections.emptyList()).size());
        
        // Set scan statuses
        if (report.getVmCount() > 0) {
            long vmScanned = servers.stream()
                .filter(s -> "VM".equals(s.getServerType()) && s.getScanResult() != null)
                .count();
            report.setVmScanStatus(String.format("%d/%d scanned", vmScanned, report.getVmCount()));
        }
        
        if (report.getWindowsCount() > 0) {
            long windowsScanned = servers.stream()
                .filter(s -> "WINDOWS".equals(s.getServerType()) && s.getScanResult() != null)
                .count();
            report.setWindowsScanStatus(String.format("%d/%d scanned", windowsScanned, report.getWindowsCount()));
        }
    }
    
    /**
     * Populate certificate statistics
     */
    private void populateCertificateStatistics(Integer csi, CsiReport report) {
        List<Certificate> certificates = certificateRepository.findByCsi(csi);
        
        report.setTotalCertificates(certificates.size());
        
        Date now = new Date();
        Calendar cal = Calendar.getInstance();
        
        // Count certificates by expiry
        for (Certificate cert : certificates) {
            if (cert.getExpiryDate() == null) continue;
            
            if (cert.getExpiryDate().before(now)) {
                report.setExpiredCerts(report.getExpiredCerts() + 1);
            } else {
                long daysUntilExpiry = (cert.getExpiryDate().getTime() - now.getTime()) / (1000 * 60 * 60 * 24);
                
                if (daysUntilExpiry <= 30) {
                    report.setExpiringIn30Days(report.getExpiringIn30Days() + 1);
                } else if (daysUntilExpiry <= 60) {
                    report.setExpiringIn60Days(report.getExpiringIn60Days() + 1);
                } else if (daysUntilExpiry <= 90) {
                    report.setExpiringIn90Days(report.getExpiringIn90Days() + 1);
                }
            }
            
            // Check truststore expiry
            if (cert.isTrustStore() && daysUntilExpiry <= 60) {
                report.setTrustStoreExpiry(report.getTrustStoreExpiry() + 1);
            }
        }
    }
    
    /**
     * Populate repository statistics
     */
    private void populateRepositoryStatistics(CsiJobExecution job, CsiReport report) {
        CsiDetails csiDetails = csiDetailsRepository.findByCsi(job.getCsi());
        
        int totalRepos = 0;
        if (csiDetails.getBitbucketRepositories() != null) {
            totalRepos += csiDetails.getBitbucketRepositories().size();
        }
        if (csiDetails.getGithubRepositories() != null) {
            totalRepos += csiDetails.getGithubRepositories().size();
        }
        
        report.setTotalRepositories(totalRepos);
        
        // Get repository scan results from job step
        JobStep repoScanStep = job.getSteps().stream()
            .filter(s -> s.getStepType() == StepType.REPO_CERT_USAGE_SCAN)
            .findFirst()
            .orElse(null);
        
        if (repoScanStep != null && repoScanStep.getResults() != null) {
            report.setScannedRepositories((Integer) repoScanStep.getResults()
                .getOrDefault("scannedRepositories", 0));
            report.setCertificatesInCode((Integer) repoScanStep.getResults()
                .getOrDefault("certificatesInCode", 0));
        }
    }
    
    /**
     * Populate BCM statistics
     */
    private void populateBcmStatistics(Integer csi, CsiReport report) {
        List<BcmException> exceptions = bcmExceptionRepository.findByCsi(csi);
        
        report.setBcmExceptions(exceptions.size());
        report.setCriticalExceptions((int) exceptions.stream()
            .filter(e -> "CRITICAL".equals(e.getSeverity()))
            .count());
    }
    
    /**
     * Populate step progress
     */
    private void populateStepProgress(CsiJobExecution job, CsiReport report) {
        List<StepProgress> progressList = new ArrayList<>();
        
        for (JobStep step : job.getSteps()) {
            StepProgress progress = new StepProgress();
            progress.setStepType(step.getStepType().name());
            progress.setStatus(step.getStatus().name());
            progress.setProgress(step.getProgress());
            progress.setMessage(step.getMessage());
            progress.setStartTime(step.getStartTime());
            progress.setEndTime(step.getEndTime());
            
            progressList.add(progress);
        }
        
        report.setStepProgressList(progressList);
    }
    
    /**
     * Generate action items based on findings
     */
    private void generateActionItems(CsiJobExecution job, CsiReport report) {
        List<String> actionItems = new ArrayList<>();
        List<String> highRiskItems = new ArrayList<>();
        List<String> mediumRiskItems = new ArrayList<>();
        
        // Check for expired certificates
        if (report.getExpiredCerts() > 0) {
            highRiskItems.add(String.format("%d expired certificates require immediate renewal", 
                report.getExpiredCerts()));
        }
        
        // Check for soon-to-expire certificates
        if (report.getExpiringIn30Days() > 0) {
            highRiskItems.add(String.format("%d certificates expiring within 30 days", 
                report.getExpiringIn30Days()));
        }
        
        // Check for failed scans
        if (report.getFailedScans() > 0) {
            mediumRiskItems.add(String.format("%d servers failed certificate scanning", 
                report.getFailedScans()));
        }
        
        // Check for ansible connectivity issues
        JobStep ansibleStep = job.getSteps().stream()
            .filter(s -> s.getStepType() == StepType.ANSIBLE_CONNECTIVITY_TEST)
            .findFirst()
            .orElse(null);
        
        if (ansibleStep != null && ansibleStep.getStatus() == StepStatus.FAILED) {
            highRiskItems.add("Ansible connectivity test failed - manual intervention required");
        }
        
        // Check for certificates in code
        if (report.getCertificatesInCode() > 0) {
            actionItems.add(String.format("%d certificates found in code repositories - review required", 
                report.getCertificatesInCode()));
        }
        
        // Check BCM exceptions
        if (report.getCriticalExceptions() > 0) {
            highRiskItems.add(String.format("%d critical BCM exceptions detected", 
                report.getCriticalExceptions()));
        }
        
        report.setHighRiskItems(highRiskItems);
        report.setMediumRiskItems(mediumRiskItems);
        report.setActionItems(actionItems);
    }
    
    /**
     * Update CSI details with latest report
     */
    private void updateCsiDetailsWithReport(Integer csi, CsiReport report) {
        CsiDetails csiDetails = csiDetailsRepository.findByCsi(csi);
        if (csiDetails != null) {
            csiDetails.setLastReportGenerationTime(report.getGeneratedAt());
            csiDetails.setLastReportId(report.getReportId());
            
            if (report.isFinalReport() && report.getJobStatus() == JobStatus.COMPLETED) {
                csiDetails.setJobStatus(JobStatus.COMPLETED);
                csiDetails.setLastSuccessfulSync(new Date());
            }
            
            csiDetailsRepository.save(csiDetails);
        }
    }
    
    /**
     * Send report email
     */
    private void sendReportEmail(Integer csi, CsiReport report) {
        CsiDetails csiDetails = csiDetailsRepository.findByCsi(csi);
        if (csiDetails == null) {
            log.warn("CSI details not found for CSI: {}", csi);
            return;
        }
        
        // Prepare email recipients
        Map<String, List<String>> recipients = new HashMap<>();
        recipients.put("to", Arrays.asList(csiDetails.getEmail()));
        
        if (csiDetails.getNotificationEmails() != null && !csiDetails.getNotificationEmails().isEmpty()) {
            recipients.put("cc", csiDetails.getNotificationEmails());
        }
        
        // Generate HTML content
        String htmlContent = generateHtmlReport(report, csiDetails);
        
        // Prepare email data
        Map<String, Object> emailData = new HashMap<>();
        emailData.put("recipients", recipients);
        emailData.put("subject", String.format("CSI %d - Certificate Management Report - %s", 
            csi, new SimpleDateFormat("yyyy-MM-dd").format(report.getGeneratedAt())));
        emailData.put("body", htmlContent);
        emailData.put("priority", report.getHighRiskItems().isEmpty() ? "NORMAL" : "HIGH");
        
        try {
            notificationService.sendEmail(emailData);
            log.info("Report email sent for CSI: {}", csi);
        } catch (Exception e) {
            log.error("Error sending report email for CSI: " + csi, e);
        }
    }
    
    /**
     * Generate HTML report content
     */
    public String generateHtmlReport(CsiReport report, CsiDetails csiDetails) {
        Context context = new Context();
        context.setVariable("report", report);
        context.setVariable("csiDetails", csiDetails);
        context.setVariable("generatedDate", new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
            .format(report.getGeneratedAt()));
        context.setVariable("isPartialReport", !report.isFinalReport());
        context.setVariable("completionClass", getCompletionClass(report.getCompletionPercentage()));
        
        return templateEngine.process("csi-report-template", context);
    }
    
    private String getCompletionClass(Integer percentage) {
        if (percentage >= 100) return "complete";
        if (percentage >= 75) return "high-progress";
        if (percentage >= 50) return "medium-progress";
        if (percentage >= 25) return "low-progress";
        return "minimal-progress";
    }
}


package com.citi.cert_management.scheduler;

import com.citi.cert_management.entity.*;
import com.citi.cert_management.enums.*;
import com.citi.cert_management.repository.*;
import com.citi.cert_management.service.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.util.Calendar;
import java.util.Date;
import java.util.List;

@Component
@Slf4j
public class JobScheduler {
    
    @Autowired
    private CsiDetailsRepository csiDetailsRepository;
    
    @Autowired
    private CsiJobExecutionRepository jobExecutionRepository;
    
    @Autowired
    private CsiJobOrchestrator jobOrchestrator;
    
    @Autowired
    private CsiReportGenerationService reportService;
    
    /**
     * Weekly sync job - runs every Sunday at 2 AM
     */
    @Scheduled(cron = "0 0 2 * * SUN")
    public void weeklySync() {
        log.info("Starting weekly sync job");
        
        // Get all active CSIs
        List<CsiDetails> activeCsis = csiDetailsRepository.findByActiveTrue();
        
        for (CsiDetails csi : activeCsis) {
            try {
                // Check if last sync was more than 6 days ago
                if (shouldRunWeeklySync(csi)) {
                    log.info("Initiating weekly sync for CSI: {}", csi.getCsi());
                    jobOrchestrator.initiateOnboardingJob(csi.getCsi(), "WEEKLY_SYNC_SCHEDULER");
                }
            } catch (Exception e) {
                log.error("Error in weekly sync for CSI: " + csi.getCsi(), e);
            }
        }
    }
    
    /**
     * Check if weekly sync should run for CSI
     */
    private boolean shouldRunWeeklySync(CsiDetails csi) {
        if (csi.getLastJobExecutionTime() == null) {
            return true;
        }
        
        Calendar cal = Calendar.getInstance();
        cal.add(Calendar.DAY_OF_MONTH, -6);
        
        return csi.getLastJobExecutionTime().before(cal.getTime());
    }
    
    /**
     * Cleanup old reports - runs daily at 3 AM
     */
    @Scheduled(cron = "0 0 3 * * *")
    public void cleanupOldReports() {
        log.info("Starting cleanup of old reports");
        
        Calendar cal = Calendar.getInstance();
        cal.add(Calendar.DAY_OF_MONTH, -30); // Keep reports for 30 days
        Date cutoffDate = cal.getTime();
        
        int deletedCount = reportRepository.deleteByGeneratedAtBefore(cutoffDate);
        log.info("Deleted {} old reports", deletedCount);
    }
    
    /**
     * Check stalled jobs - runs every hour
     */
    @Scheduled(cron = "0 0 * * * *")
    public void checkStalledJobs() {
        log.info("Checking for stalled jobs");
        
        Calendar cal = Calendar.getInstance();
        cal.add(Calendar.HOUR_OF_DAY, -24); // Jobs older than 24 hours
        Date cutoffDate = cal.getTime();
        
        List<CsiJobExecution> stalledJobs = jobExecutionRepository
            .findByOverallStatusAndStartTimeBefore(JobStatus.IN_PROGRESS, cutoffDate);
        
        for (CsiJobExecution job : stalledJobs) {
            log.warn("Found stalled job: {} for CSI: {}", job.getJobId(), job.getCsi());
            
            // Check if job has any recent activity
            if (job.getLastUpdated() != null && job.getLastUpdated().after(cutoffDate)) {
                continue; // Job has recent activity
            }
            
            // Mark job as partially completed
            job.setOverallStatus(JobStatus.PARTIALLY_COMPLETED);
            job.getWarnings().add("Job marked as partially completed due to inactivity");
            jobExecutionRepository.save(job);
            
            // Generate final report
            reportService.generateReportForJob(job);
        }
    }
    
    @Autowired
    private CsiReportRepository reportRepository;
}



package com.citi.cert_management.service;

import org.springframework.stereotype.Service;
import java.util.Map;

@Service
public interface CasIntegrationService {
    String initiateServerSync(Integer csi);
    boolean checkSyncStatus(String syncJobId);
    Map<String, Object> getSyncResults(String syncJobId);
}


package com.citi.cert_management.service;

import org.springframework.stereotype.Service;
import java.util.Map;

@Service
public interface VaultIntegrationService {
    Map<String, Object> syncCertificates(Integer csi);
}

package com.citi.cert_management.service;

import com.citi.cert_management.entity.FEInventoryReportNew;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Map;

@Service
public interface AnsibleIntegrationService {
    Map<String, Object> testConnectivity(List<FEInventoryReportNew> servers);
    String initiateVmCertScan(Integer csi, List<String> servers);
    String initiateWindowsCertScan(Integer csi, List<String> servers);
    Map<String, Object> getJobStatus(String jobId);
}

package com.citi.cert_management.repository;

import com.citi.cert_management.entity.CsiJobExecution;
import com.citi.cert_management.enums.JobStatus;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import java.util.Date;
import java.util.List;

@Repository
public interface CsiJobExecutionRepository extends MongoRepository<CsiJobExecution, String> {
    List<CsiJobExecution> findByOverallStatus(JobStatus status);
    List<CsiJobExecution> findByOverallStatusIn(List<JobStatus> statuses);
    List<CsiJobExecution> findByReportGenerationEnabledTrueAndReportGenerationStartDateBefore(Date date);
    List<CsiJobExecution> findByOverallStatusAndStartTimeBefore(JobStatus status, Date date);
    CsiJobExecution findTopByCsiOrderByStartTimeDesc(Integer csi);
}


package com.citi.cert_management.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
@EnableScheduling
public class AsyncConfiguration {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("CSI-Async-");
        executor.initialize();
        return executor;
    }
}


7. Enhancement Recommendations for Existing Methods
7.1 Enhanced CsiMetadataService.java
// Add these methods to existing CsiMetadataService

public class CsiMetadataService {
    
    /**
     * Enhanced method to fetch CSI details from Archcenter with retry logic
     */
    public CsiDetails fetchCsiDetailsFromArchcenter(Integer csi) {
        int maxRetries = 3;
        int retryCount = 0;
        
        while (retryCount < maxRetries) {
            try {
                // Existing archcenter call logic
                Document doc = parseXml(response.getBody());
                CsiDetails details = extractCsiDetails(csi, doc);
                
                // Validate required fields
                validateCsiDetails(details);
                
                return details;
            } catch (Exception e) {
                retryCount++;
                if (retryCount >= maxRetries) {
                    throw new RuntimeException("Failed to fetch CSI details after " + maxRetries + " attempts", e);
                }
                log.warn("Retry {} for CSI {}: {}", retryCount, csi, e.getMessage());
                try {
                    Thread.sleep(1000 * retryCount); // Exponential backoff
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
            }
        }
        return null;
    }
    
    /**
     * Enhanced extraction with null safety
     */
    private void extractCsiDetails(Integer csiId, CsiDetails csi, NodeList nodeList, String email) {
        // Existing extraction logic with enhanced null checks
        for (int itr = 0; itr < nodeList.getLength(); itr++) {
            Node node = nodeList.item(itr);
            if (node.getNodeType() == Node.ELEMENT_NODE) {
                Element eElement = (Element) node;
                String key = eElement.getElementsByTagName("key").item(0).getTextContent();
                String value = getElementValue(eElement, "value");
                
                // Use safe extraction
                safeSetCsiField(csi, key, value);
            }
        }
    }
    
    private String getElementValue(Element element, String tagName) {
        NodeList nodeList = element.getElementsByTagName(tagName);
        if (nodeList != null && nodeList.getLength() > 0) {
            Node node = nodeList.item(0);
            if (node != null && node.getTextContent() != null) {
                return node.getTextContent().trim();
            }
        }
        return "";
    }
    
    private void safeSetCsiField(CsiDetails csi, String key, String value) {
        if (key == null || value == null || value.isEmpty()) {
            return;
        }
        
        try {
            switch (key) {
                case "MANAGER":
                    csi.setCsiManagerSOEID(value);
                    break;
                case "MANAGERGEID":
                    csi.setCsiManagerGEID(parseLongSafe(value));
                    break;
                // ... other cases
                default:
                    log.debug("Unknown field: {} = {}", key, value);
            }
        } catch (Exception e) {
            log.warn("Error setting field {} with value {}: {}", key, value, e.getMessage());
        }
    }
    
    private Long parseLongSafe(String value) {
        try {
            return Long.parseLong(value);
        } catch (NumberFormatException e) {
            log.warn("Failed to parse long: {}", value);
            return null;
        }
    }
}


// Add these fields to track scan progress

@Document(collection = "FE_Inventory")
public class FEInventoryReportNew {
    // Existing fields...
    
    // Enhanced scan tracking
    private ScanResult scanResult;
    private Date lastScanAttempt;
    private Integer scanRetryCount = 0;
    private String ansibleJobId;
    private Map<String, Object> scanMetadata;
    
    @Data
    public static class ScanResult {
        private String status; // SUCCESS, FAILED, IN_PROGRESS, NOT_STARTED
        private Date scanDate;
        private Integer certificatesFound = 0;
        private List<String> errors = new ArrayList<>();
        private Map<String, Object> scanDetails = new HashMap<>();
    }
}

8. API Controllers
8.1 CsiJobController.java

   package com.citi.cert_management.controller;

import com.citi.cert_management.entity.*;
import com.citi.cert_management.service.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/csi-jobs")
public class CsiJobController {
    
    @Autowired
    private CsiJobOrchestrator jobOrchestrator;
    
    @Autowired
    private CsiJobExecutionRepository jobExecutionRepository;
    
    @Autowired
    private CsiReportGenerationService reportService;
    
    @PostMapping("/{csi}/initiate")
    public ResponseEntity<Map<String, Object>> initiateJob(
            @PathVariable Integer csi,
            @RequestParam String triggeredBy) {
        
        String jobId = jobOrchestrator.initiateOnboardingJob(csi, triggeredBy);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("jobId", jobId);
        response.put("message", "Job initiated successfully");
        
        return ResponseEntity.ok(response);
    }
    
    @GetMapping("/{jobId}/status")
    public ResponseEntity<CsiJobExecution> getJobStatus(@PathVariable String jobId) {
        CsiJobExecution job = jobExecutionRepository.findById(jobId)
            .orElseThrow(() -> new RuntimeException("Job not found: " + jobId));
        
        return ResponseEntity.ok(job);
    }
    
    @PostMapping("/{jobId}/generate-report")
    public ResponseEntity<Map<String, Object>> generateReport(@PathVariable String jobId) {
        CsiJobExecution job = jobExecutionRepository.findById(jobId)
            .orElseThrow(() -> new RuntimeException("Job not found: " + jobId));
        
        CsiReport report = reportService.generateReportForJob(job);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("reportId", report.getReportId());
        response.put("message", "Report generated successfully");
        
        return ResponseEntity.ok(response);
    }
}

Documentation Summary
System Architecture

Asynchronous Job Execution: Jobs are initiated and run asynchronously with proper state management
Progress Tracking: Each step tracks its progress independently with retry logic
Daily Report Generation: Reports are generated daily starting 2 days after job initiation
Weekly Sync: Automatic weekly synchronization to keep data current
Email Notifications: HTML email reports sent to configured recipients

Key Features

Parallel Execution: Steps 2.1, 2.2, and 2.3 run in parallel
Dependency Management: Steps wait for their dependencies before execution
Long-Running Job Support: Ansible jobs tracked separately with progress polling
Comprehensive Reporting: Detailed reports with risk analysis and action items
Scalable Architecture: Thread pool executors for concurrent job processing

Usage Flow

CSI Onboarding initiated via API or scheduler
Job created with all steps initialized
Phase 1 sync operations start in parallel
Phase 2 operations scheduled based on dependencies
Progress tracked continuously
Reports generated daily after 2-day waiting period
Weekly sync maintains data freshness




1. Detailed Sequence Diagram

   sequenceDiagram
    participant User
    participant API as CsiJobController
    participant JO as CsiJobOrchestrator
    participant JPT as JobProgressTracker
    participant CAS as CasIntegrationService
    participant Vault as VaultIntegrationService
    participant BB as BitbucketIntegrationService
    participant Ansible as AnsibleIntegrationService
    participant Scan as CertificateScanningService
    participant IA as ImpactAssessmentService
    participant RGS as ReportGenerationService
    participant Email as NotificationService
    participant DB as MongoDB
    participant Scheduler

    %% Onboarding Initiation
    User->>API: POST /api/csi-jobs/{csi}/initiate
    API->>JO: initiateOnboardingJob(csi, triggeredBy)
    JO->>DB: Create CsiJobExecution with all steps
    JO->>DB: Update CsiDetails with jobId
    JO-->>API: Return jobId
    API-->>User: {success: true, jobId: xxx}

    %% Async Job Execution
    JO->>JO: executeJobAsync(jobId)
    activate JO
    
    %% Phase 1: Parallel Sync Operations
    par Server Sync
        JO->>CAS: initiateServerSync(csi)
        CAS-->>JO: syncJobId
        JO->>DB: Update step with IN_PROGRESS
        loop Poll for completion (30 attempts)
            JO->>CAS: checkSyncStatus(syncJobId)
            CAS-->>JO: status
        end
        JO->>CAS: getSyncResults(syncJobId)
        CAS-->>JO: {serverCount, syncedServers, errors}
        JO->>DB: Update step COMPLETED/PARTIALLY_COMPLETED
    and Certificate Sync
        JO->>Vault: syncCertificates(csi)
        Vault-->>JO: {certificateCount, certificates, expiryInfo}
        JO->>DB: Update step with results
    and Repository Sync
        JO->>BB: syncRepositories(csi, repos)
        BB-->>JO: {repositoryCount, syncedRepos}
        JO->>DB: Update step with results
    end

    %% Schedule Phase 2 Operations
    JO->>JPT: scheduleStepExecution(jobId, ANSIBLE_CONNECTIVITY_TEST)
    JO->>JPT: scheduleStepExecution(jobId, REPO_CERT_USAGE_SCAN)
    JO->>DB: Enable report generation (start date = now + 2 days)
    deactivate JO

    %% JobProgressTracker Monitoring
    loop Every 5 minutes
        JPT->>DB: Check pending steps
        JPT->>JPT: checkAndExecuteStep()
        
        %% Ansible Connectivity Test (depends on Server Sync)
        alt Server Sync Completed
            JPT->>JO: executeAnsibleConnectivityTest()
            activate JO
            JO->>DB: Get servers from inventory
            JO->>Ansible: testConnectivity(servers)
            Ansible-->>JO: {successfulServers, failedServers}
            JO->>DB: Update step with results
            
            %% Schedule Server Scans
            JO->>JPT: scheduleStepExecution(jobId, VM_CERT_SCAN)
            JO->>JPT: scheduleStepExecution(jobId, WINDOWS_CERT_SCAN)
            JO->>JPT: scheduleStepExecution(jobId, BCM_EXCEPTION_SCAN)
            deactivate JO
        end

        %% Repository Scan (depends on Cert & Repo Sync)
        alt Cert Sync and Repo Sync Completed
            JPT->>JO: executeRepositoryScans()
            activate JO
            JO->>Scan: scanRepositoriesForCertUsage(csi)
            Scan-->>JO: {certificatesInCode, locations}
            JO->>DB: Update step with results
            deactivate JO
        end

        %% Server Scans (depends on Ansible Test)
        alt Ansible Test Completed
            JPT->>JO: executeVmCertScan()
            activate JO
            JO->>Ansible: initiateVmCertScan(csi, vmServers)
            Ansible-->>JO: ansibleJobId
            JO->>DB: Update step with ansibleJobId
            deactivate JO
            
            %% Similar for Windows and BCM scans
        end
    end

    %% Ansible Job Progress Monitoring
    loop Every 5 minutes
        JPT->>DB: Find jobs with ansible tasks IN_PROGRESS
        JPT->>Ansible: getJobStatus(ansibleJobId)
        Ansible-->>JPT: {status, progress}
        JPT->>DB: Update step progress
    end

    %% Impact Assessment (depends on all scans)
    alt All Scans Completed
        JPT->>IA: performImpactAssessment(csi, scanResults)
        IA-->>JPT: {riskScore, findings, recommendations}
        JPT->>DB: Update step with results
    end

    %% Daily Report Generation
    Scheduler->>RGS: generateDailyReports() [Daily at 8 AM]
    activate RGS
    RGS->>DB: Find eligible jobs (reportGenerationEnabled && startDate < now)
    
    loop For each eligible job
        RGS->>RGS: generateReportForJob(job)
        RGS->>DB: Get job execution details
        RGS->>DB: Get server inventory
        RGS->>DB: Get certificates
        RGS->>DB: Get BCM exceptions
        RGS->>RGS: Calculate statistics
        RGS->>RGS: Generate action items
        RGS->>DB: Save CsiReport
        RGS->>RGS: generateHtmlReport(report)
        RGS->>Email: sendEmail(recipients, htmlContent)
        Email-->>RGS: Email sent
        RGS->>DB: Update CsiDetails with lastReportId
    end
    deactivate RGS

    %% Weekly Sync
    Scheduler->>Scheduler: weeklySync() [Sunday 2 AM]
    Scheduler->>DB: Find active CSIs
    loop For each CSI needing sync
        Scheduler->>JO: initiateOnboardingJob(csi, "WEEKLY_SYNC")
    end

    %% User Check Status
    User->>API: GET /api/csi-jobs/{jobId}/status
    API->>DB: Find job by ID
    DB-->>API: CsiJobExecution
    API-->>User: Job details with step progress

   2.1 Missing Entity Classes

   // Certificate.java
package com.citi.cert_management.entity;

import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import java.util.Date;

@Document(collection = "certificates")
@Data
public class Certificate {
    @Id
    private String id;
    private Integer csi;
    private String certificateName;
    private String certificateType;
    private Date expiryDate;
    private String issuer;
    private String subject;
    private boolean isTrustStore;
    private String environment;
    private String serverId;
    private String repositoryLocation;
    private Date lastUpdated;
}

// BcmException.java
package com.citi.cert_management.entity;

import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import java.util.Date;

@Document(collection = "bcm_exceptions")
@Data
public class BcmException {
    @Id
    private String id;
    private Integer csi;
    private String exceptionType;
    private String severity; // CRITICAL, HIGH, MEDIUM, LOW
    private String serverId;
    private String description;
    private Date detectedDate;
    private String status;
    private String remediationAction;
}

// StepProgress.java (for report)
package com.citi.cert_management.dto;

import lombok.Data;
import java.util.Date;

@Data
public class StepProgress {
    private String stepType;
    private String status;
    private Integer progress;
    private String message;
    private Date startTime;
    private Date endTime;
}

2.2 Missing Repository Classes

// CertificateRepository.java
package com.citi.cert_management.repository;

import com.citi.cert_management.entity.Certificate;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface CertificateRepository extends MongoRepository<Certificate, String> {
    List<Certificate> findByCsi(Integer csi);
    List<Certificate> findByCsiAndEnvironment(Integer csi, String environment);
}

// BcmExceptionRepository.java
package com.citi.cert_management.repository;

import com.citi.cert_management.entity.BcmException;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface BcmExceptionRepository extends MongoRepository<BcmException, String> {
    List<BcmException> findByCsi(Integer csi);
    List<BcmException> findByCsiAndSeverity(Integer csi, String severity);
}

// CsiReportRepository.java
package com.citi.cert_management.repository;

import com.citi.cert_management.entity.CsiReport;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;
import java.util.Date;

@Repository
public interface CsiReportRepository extends MongoRepository<CsiReport, String> {
    CsiReport findTopByCsiOrderByGeneratedAtDesc(Integer csi);
    int deleteByGeneratedAtBefore(Date date);
}

// FEInventoryReportNewRepository.java
package com.citi.cert_management.repository;

import com.citi.cert_management.entity.FEInventoryReportNew;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface FEInventoryReportNewRepository extends MongoRepository<FEInventoryReportNew, String> {
    List<FEInventoryReportNew> findByCsi(Integer csi);
    List<FEInventoryReportNew> findByCsiAndServerType(Integer csi, String serverType);
}

2.3 Missing Service Implementations

// BitbucketIntegrationService.java
package com.citi.cert_management.service;

import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Map;

@Service
public interface BitbucketIntegrationService {
    Map<String, Object> syncRepositories(Integer csi, List<String> repositories);
    Map<String, Object> scanRepositoryForCertificates(String repositoryUrl);
}

// CertificateScanningService.java
package com.citi.cert_management.service;

import org.springframework.stereotype.Service;
import java.util.Map;

@Service
public interface CertificateScanningService {
    Map<String, Object> scanRepositoriesForCertUsage(Integer csi);
    Map<String, Object> analyzeCertificateUsage(String certificateId, List<String> repositories);
}

// ImpactAssessmentService.java
package com.citi.cert_management.service;

import org.springframework.stereotype.Service;
import java.util.Map;

@Service
public interface ImpactAssessmentService {
    Map<String, Object> performImpactAssessment(Integer csi, Map<String, Object> scanResults);
    Map<String, Object> calculateRiskScore(Integer csi);
}

// NotificationService.java
package com.citi.cert_management.service;

import org.springframework.stereotype.Service;
import java.util.Map;

@Service
public interface NotificationService {
    void sendEmail(Map<String, Object> emailData);
    void sendAlert(String recipient, String message, String priority);
}

4. Expected Result Formats for Each Step
Each step should return results in a specific format for proper report generation:
4.1 Server Sync (CAS Integration)

   // Expected result format from CasIntegrationService.getSyncResults()
{
    "serverCount": 150,
    "syncedServers": 148,
    "failedServers": 2,
    "serverTypes": {
        "VM": 100,
        "WINDOWS": 45,
        "CONTAINER": 5
    },
    "errors": [
        "Server XYZ123: Connection timeout",
        "Server ABC456: Authentication failed"
    ],
    "syncDuration": 3600000, // milliseconds
    "dataSource": "CAS"
}

4.2 Certificate Sync (Vault Integration)


// Expected result format from VaultIntegrationService.syncCertificates()
{
    "certificateCount": 250,
    "certificates": [
        {
            "certificateId": "cert123",
            "name": "app.example.com",
            "expiryDate": "2025-12-31T23:59:59Z",
            "daysUntilExpiry": 90,
            "type": "SSL",
            "environment": "PROD"
        }
    ],
    "expiryBreakdown": {
        "expired": 5,
        "expiringIn30Days": 12,
        "expiringIn60Days": 25,
        "expiringIn90Days": 40
    },
    "trustStoreCerts": 15,
    "syncDuration": 45000
}

4.3 Repository Sync

// Expected result format from BitbucketIntegrationService.syncRepositories()
{
    "repositoryCount": 25,
    "syncedRepositories": 24,
    "failedRepositories": 1,
    "repositoryTypes": {
        "bitbucket": 20,
        "github": 5
    },
    "branches": {
        "master": 25,
        "develop": 20,
        "feature": 45
    },
    "errors": ["repo123: Access denied"],
    "syncDuration": 120000
}

4.4 Ansible Connectivity Test

// Expected result format from AnsibleIntegrationService.testConnectivity()
{
    "totalServers": 150,
    "successfulConnections": 145,
    "failedConnections": 5,
    "successfulServers": [
        {"serverId": "srv001", "hostname": "app01.example.com", "type": "VM"},
        // ... more servers
    ],
    "failedServers": [
        {
            "serverId": "srv100",
            "hostname": "db01.example.com",
            "error": "SSH timeout",
            "retryable": true
        }
    ],
    "avgConnectionTime": 2500 // milliseconds
}

4.5 VM/Windows Certificate Scan

// Expected result format from scan operations
{
    "scannedServers": 100,
    "successfulScans": 98,
    "failedScans": 2,
    "certificatesFound": 450,
    "certificatesByType": {
        "SSL": 300,
        "CODE_SIGNING": 50,
        "CA": 100
    },
    "certificatesByLocation": {
        "/etc/ssl/certs": 200,
        "Java Keystore": 150,
        "Windows Store": 100
    },
    "ansibleJobId": "job_2025_001",
    "scanDuration": 7200000,
    "errors": ["srv050: Certificate store access denied"]
}

4.6 Repository Certificate Usage Scan

// Expected result format from CertificateScanningService.scanRepositoriesForCertUsage()
{
    "scannedRepositories": 24,
    "certificatesInCode": 15,
    "certificateLocations": [
        {
            "repository": "app-repo",
            "branch": "master",
            "file": "src/main/resources/cert.pem",
            "lineNumber": 1,
            "certificateType": "PEM",
            "lastModified": "2025-01-15T10:30:00Z"
        }
    ],
    "hardcodedPasswords": 3,
    "privateKeysFound": 2,
    "scanDuration": 180000
}

4.7 BCM Exception Scan

// Expected result format for BCM scanning
{
    "totalExceptions": 45,
    "criticalExceptions": 5,
    "highExceptions": 12,
    "mediumExceptions": 20,
    "lowExceptions": 8,
    "exceptionsByType": {
        "MISSING_PATCH": 15,
        "WEAK_CIPHER": 10,
        "OUTDATED_SSL": 8,
        "INSECURE_PROTOCOL": 12
    },
    "remediationRequired": 25,
    "autoRemediatable": 15
}

4.8 Impact Assessment

// Expected result format from ImpactAssessmentService.performImpactAssessment()
{
    "overallRiskScore": 75, // 0-100
    "riskLevel": "HIGH",
    "criticalFindings": [
        {
            "type": "EXPIRED_CERT_IN_PROD",
            "count": 3,
            "impact": "Service unavailability",
            "recommendation": "Immediate renewal required"
        }
    ],
    "recommendations": [
        {
            "priority": "HIGH",
            "action": "Renew 3 expired certificates",
            "deadline": "2025-01-20"
        },
        {
            "priority": "MEDIUM",
            "action": "Update 15 certificates expiring in 30 days",
            "deadline": "2025-02-15"
        }
    ],
    "complianceStatus": {
        "compliant": false,
        "violations": ["CERT_EXPIRY_POLICY", "BCM_THRESHOLD"]
    }
}

4.9 Standardized Result Storage
To ensure consistency, implement a result wrapper:

@Data
public class StepResult {
    private boolean success;
    private String status; // SUCCESS, PARTIAL_SUCCESS, FAILURE
    private Map<String, Object> data;
    private List<String> errors;
    private List<String> warnings;
    private Long duration; // milliseconds
    private Date timestamp;
    
    public static StepResult success(Map<String, Object> data) {
        StepResult result = new StepResult();
        result.setSuccess(true);
        result.setStatus("SUCCESS");
        result.setData(data);
        result.setTimestamp(new Date());
        return result;
    }
    
    public static StepResult failure(String error) {
        StepResult result = new StepResult();
        result.setSuccess(false);
        result.setStatus("FAILURE");
        result.setErrors(Arrays.asList(error));
        result.setTimestamp(new Date());
        return result;
    }
}

