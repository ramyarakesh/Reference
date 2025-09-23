
private void validateMandatoryFields(CsiOnboardingRequest request) { if (request.getCsiId() == null || request.getRequestorSoeid() == null || request.getEmailDl() == null || request.getEmail() == null || request.getPreferredRenewal() == null || request.getNotificationEmailList() == null) { throw new ValidationException("Missing mandatory fields for CSI onboarding"); } }

private void validateModuleSubscriptions(List<ModuleSubscriptionRequest> subscriptions) { // Ensure at least one module is selected boolean hasSelection = subscriptions.stream() .anyMatch(ModuleSubscriptionRequest::isSelected); if (!hasSelection) { throw new ValidationException("At least one module must be selected"); } // Validate module-specific configurations for (ModuleSubscriptionRequest subscription : subscriptions) { if (subscription.isSelected()) { validateModuleConfiguration(subscription); } } }

package com.citi.cert_management.container.dto;

import lombok.Data;
import lombok.Builder;
import javax.validation.constraints.NotNull;
import java.util.*;

/**
 * Request DTOs
 */
@Data
public class CsiOnboardingRequest {
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
   
    private boolean vmDiscoveryEnabled;
    private boolean containerDiscoveryEnabled;
    private boolean autoDeploymentEnabled;
   
    private Map<String, Object> additionalConfig;
}

@Data
public class RenewalRequest {
    @NotNull
    private String certificateId;
   
    private String renewalProvider;
    private boolean autoRenewal;
    private Map<String, String> certificateAttributes;
    private LocalDateTime requestedExpiryDate;
}

@Data
public class DeploymentRequest {
    private List<String> targetPaths;
    private String branchName;
    private String commitMessage;
    private boolean createPullRequest;
    private List<String> reviewers;
}

/**
 * Response DTOs
 */
@Data
@Builder
public class OnboardingResponse {
    private String csiId;
    private String jobId;
    private String message;
    private LocalDateTime timestamp;
}

@Data
@Builder
public class JobResponse {
    private String jobId;
    private String message;
    private JobStatus status;
    private LocalDateTime estimatedCompletion;
}

@Data
@Builder
public class JobStatusResponse {
    private String jobId;
    private JobStatus status;
    private double progress;
    private String currentStep;
    private LocalDateTime startTime;
    private LocalDateTime estimatedCompletion;
    private List<StepProgress> steps;
    private Map<String, Object> metadata;
}

@Data
@Builder
public class RenewalResponse {
    private String renewalId;
    private String status;
    private String message;
    private String newCertificateId;
    private LocalDateTime expiryDate;
}

@Data
@Builder
public class ClmReportResponse {
    private String reportId;
    private Object content;
    private ReportFormat format;
    private LocalDateTime generatedAt;
}

@Data
@Builder
public class InventoryReport {
    private String csiId;
    private int totalCertificates;
    private Map<String, Integer> certsByCategory;
    private List<CertificateSummary> expiringCertificates;
    private List<ActionItemSummary> actionItems;
    private LocalDateTime reportDate;
}

@Data
@Builder
public class DeploymentResult {
    private boolean success;
    private String pullRequestUrl;
    private String pullRequestId;
    private String message;
    private String errorMessage;
    private Map<String, Object> metadata;
}

package com.citi.cert_management.container.controller;

import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import javax.validation.Valid;
import java.util.*;

/**
 * REST controller for Certificate Lifecycle Management operations
 * Provides endpoints for CSI onboarding, discovery, assessment, and reporting
 */
@Slf4j
@RestController
@RequestMapping("/api/v1/clm")
@RequiredArgsConstructor
@Tag(name = "CLM API", description = "Certificate Lifecycle Management operations")
public class ClmController {
    
    private final ClmService clmService;
    private final CsiService csiService;
    private final ReportService reportService;
    
    /**
     * Enhanced CSI onboarding with CLM configuration
     * 
     * @param request CSI onboarding request with mandatory fields
     * @return ResponseEntity with onboarding result and job ID
     */
    @PostMapping("/csi/onboard")
    @Operation(summary = "Onboard CSI with CLM configuration",
              description = "Creates CSI profile and initiates onboarding scan")
    public ResponseEntity<OnboardingResponse> onboardCsi(@Valid @RequestBody CsiOnboardingRequest request) {
        log.info("Onboarding CSI: {}", request.getCsiId());
        
        validateMandatoryFields(request);
        
        // Save CSI details
        CsiDetails csi = csiService.createCsi(request);
        
        // Initiate onboarding scan
        String jobId = clmService.performOnboardingScan(csi.getCsiId(), request.getRequestorSoeid());
        
        return ResponseEntity.ok(OnboardingResponse.builder()
            .csiId(csi.getCsiId())
            .jobId(jobId)
            .message("CSI onboarded successfully. Scan initiated.")
            .build());
    }
    
    /**
     * Triggers VM discovery for a CSI
     */
    @PostMapping("/csi/{csiId}/vm-discovery")
    @Operation(summary = "Trigger VM certificate discovery")
    public ResponseEntity<JobResponse> triggerVmDiscovery(
            @PathVariable String csiId,
            @RequestParam(required = false) String triggeredBy) {
        
        String jobId = clmService.initiateJob(csiId, JobType.VM_DISCOVERY, 
                                            triggeredBy, Collections.emptyMap());
        
        return ResponseEntity.ok(new JobResponse(jobId, "VM discovery initiated"));
    }
    
    /**
     * Triggers container discovery for a CSI
     */
    @PostMapping("/csi/{csiId}/container-discovery")
    @Operation(summary = "Trigger container certificate discovery")
    public ResponseEntity<JobResponse> triggerContainerDiscovery(
            @PathVariable String csiId,
            @RequestParam(required = false) String triggeredBy) {
        
        String jobId = clmService.initiateJob(csiId, JobType.CONTAINER_DISCOVERY, 
                                            triggeredBy, Collections.emptyMap());
        
        return ResponseEntity.ok(new JobResponse(jobId, "Container discovery initiated"));
    }
    
    /**
     * Performs impact assessment on discovered certificates
     */
    @PostMapping("/csi/{csiId}/impact-assessment")
    @Operation(summary = "Run certificate impact assessment")
    public ResponseEntity<JobResponse> runImpactAssessment(
            @PathVariable String csiId,
            @RequestParam String jobId) {
        
        Map<String, Object> params = new HashMap<>();
        params.put("discoveryJobId", jobId);
        
        String assessmentJobId = clmService.initiateJob(csiId, JobType.IMPACT_ASSESSMENT, 
                                                      "system", params);
        
        return ResponseEntity.ok(new JobResponse(assessmentJobId, "Impact assessment initiated"));
    }
    
    /**
     * Initiates certificate renewal process
     */
    @PostMapping("/csi/{csiId}/renewal")
    @Operation(summary = "Initiate certificate renewal")
    public ResponseEntity<RenewalResponse> initiateCertificateRenewal(
            @PathVariable String csiId,
            @Valid @RequestBody RenewalRequest request) {
        
        RenewalResult result = clmService.processCertificateRenewal(csiId, 
                                                                   request.getCertificateId(), 
                                                                   request);
        
        return ResponseEntity.ok(RenewalResponse.builder()
            .renewalId(result.getRenewalId())
            .status(result.getStatus())
            .message(result.getMessage())
            .build());
    }
    
    /**
     * Generates inventory report for a CSI
     */
    @GetMapping("/csi/{csiId}/inventory-report")
    @Operation(summary = "Generate certificate inventory report")
    public ResponseEntity<InventoryReport> getInventoryReport(
            @PathVariable String csiId,
            @RequestParam(defaultValue = "JSON") ReportFormat format) {
        
        ClmReport report = reportService.generateInventoryReport(csiId, format);
        
        return ResponseEntity.ok(convertToInventoryReport(report));
    }
    
    /**
     * Retrieves job status and progress
     */
    @GetMapping("/jobs/{jobId}/status")
    @Operation(summary = "Get job execution status")
    public ResponseEntity<JobStatusResponse> getJobStatus(@PathVariable String jobId) {
        
        JobStatusResponse status = clmService.getJobStatus(jobId);
        
        return ResponseEntity.ok(status);
    }
    
    /**
     * Retrieves CLM report by job ID
     */
    @GetMapping("/reports/{jobId}")
    @Operation(summary = "Get CLM report for completed job")
    public ResponseEntity<ClmReportResponse> getReport(
            @PathVariable String jobId,
            @RequestParam(defaultValue = "JSON") ReportFormat format) {
        
        ClmReport report = reportService.getReportByJobId(jobId, format);
        
        if (format == ReportFormat.HTML) {
            return ResponseEntity.ok()
                .header("Content-Type", "text/html")
                .body(new ClmReportResponse(report.getReportId(), report.getHtmlContent()));
        }
        
        return ResponseEntity.ok(new ClmReportResponse(report.getReportId(), report.getRawData()));
    }
    
    /**
     * Triggers adhoc scan for a CSI
     */
    @PostMapping("/csi/{csiId}/adhoc-scan")
    @Operation(summary = "Perform adhoc certificate scan")
    public ResponseEntity<JobResponse> performAdhocScan(
            @PathVariable String csiId,
            @RequestParam String triggeredBy) {
        
        String jobId = clmService.performAdhocScan(csiId, triggeredBy);
        
        return ResponseEntity.ok(new JobResponse(jobId, "Adhoc scan initiated"));
    }
    
    /**
     * Searches historical reports
     */
    @GetMapping("/reports/search")
    @Operation(summary = "Search historical CLM reports")
    public ResponseEntity<List<ReportSummary>> searchReports(
            @RequestParam(required = false) String csiId,
            @RequestParam(required = false) LocalDateTime startDate,
            @RequestParam(required = false) LocalDateTime endDate,
            @RequestParam(required = false) ReportType reportType) {
        
        List<ClmReport> reports = reportService.searchReports(csiId, startDate, endDate, reportType);
        
        return ResponseEntity.ok(convertToReportSummaries(reports));
    }
    
    /**
     * Updates CSI configuration
     */
    @PutMapping("/csi/{csiId}")
    @Operation(summary = "Update CSI CLM configuration")
    public ResponseEntity<CsiDetails> updateCsiConfig(
            @PathVariable String csiId,
            @Valid @RequestBody CsiUpdateRequest request) {
        
        CsiDetails updated = csiService.updateCsi(csiId, request);
        
        return ResponseEntity.ok(updated);
    }
    
    /**
     * Gets action items for a CSI
     */
    @GetMapping("/csi/{csiId}/action-items")
    @Operation(summary = "Get pending action items for CSI")
    public ResponseEntity<List<ActionItem>> getActionItems(
            @PathVariable String csiId,
            @RequestParam(defaultValue = "OPEN") ActionStatus status) {
        
        List<ActionItem> items = clmService.getActionItems(csiId, status);
        
        return ResponseEntity.ok(items);
    }
    
    private void validateMandatoryFields(CsiOnboardingRequest request) {
        if (request.getCsiId() == null || request.getRequestorSoeid() == null ||
            request.getEmailDl() == null || request.getEmail() == null ||
            request.getPreferredRenewal() == null || request.getNotificationEmailList() == null) {
            throw new ValidationException("Missing mandatory fields for CSI onboarding");
        }
    }
}


package com.citi.cert_management.container.service;

import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.util.*;
import java.time.LocalDateTime;

/**
 * Main service for CLM operations orchestration
 * Coordinates certificate lifecycle management activities
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class ClmService {
    
    private final ClmJobOrchestrationService jobOrchestrationService;
    private final DiscoveryService discoveryService;
    private final AssessmentService assessmentService;
    private final RenewalService renewalService;
    private final DeploymentService deploymentService;
    private final ReportGenerationService reportService;
    private final NotificationService notificationService;
    private final CsiMetadataRepository csiRepository;
    
    /**
     * Performs complete onboarding scan for newly onboarded CSI
     * 
     * @param csiId CSI identifier
     * @param userId User who triggered onboarding
     * @return Job ID for tracking
     */
    public String performOnboardingScan(String csiId, String userId) {
        log.info("Starting onboarding scan for CSI: {}", csiId);
        
        CsiDetails csi = csiRepository.findByCsiId(csiId)
            .orElseThrow(() -> new CsiNotFoundException("CSI not found: " + csiId));
        
        Map<String, Object> parameters = buildOnboardingParameters(csi);
        
        return jobOrchestrationService.initiateJob(
            csiId, 
            JobType.ONBOARDING_SCAN,
            userId,
            parameters
        );
    }
    
    /**
     * Executes VM discovery operations
     */
    public DiscoveryResults performVmDiscovery(String csiId, Map<String, Object> params) {
        log.info("Performing VM discovery for CSI: {}", csiId);
        
        try {
            // Trigger Ansible playbook for VM CLM scan
            AnsibleResponse vmScanResponse = discoveryService.triggerVmClmScan(csiId, params);
            
            // Wait for completion and collect results
            DiscoveryResults results = discoveryService.collectVmResults(vmScanResponse.getJobId());
            
            // Store results in MongoDB
            discoveryService.saveVmDiscoveryResults(csiId, results);
            
            return results;
        } catch (Exception e) {
            log.error("VM discovery failed for CSI: {}", csiId, e);
            throw new DiscoveryException("VM discovery failed", e);
        }
    }
    
    /**
     * Executes container discovery operations
     */
    public DiscoveryResults performContainerDiscovery(String csiId, Map<String, Object> params) {
        log.info("Performing container discovery for CSI: {}", csiId);
        
        CsiDetails csi = csiRepository.findByCsiId(csiId)
            .orElseThrow(() -> new CsiNotFoundException("CSI not found: " + csiId));
        
        try {
            DiscoveryResults results = new DiscoveryResults();
            
            // If repositories provided, scan them
            if (csi.getRepositories() != null && !csi.getRepositories().isEmpty()) {
                results = discoveryService.scanRepositories(csi);
            }
            
            // Query vault for container certificates
            List<ContainerCertificate> vaultCerts = discoveryService.queryVaultCertificates(csiId);
            results.setTotalCertsFound(vaultCerts.size());
            
            return results;
        } catch (Exception e) {
            log.error("Container discovery failed for CSI: {}", csiId, e);
            throw new DiscoveryException("Container discovery failed", e);
        }
    }
    
    /**
     * Performs impact assessment on discovered certificates
     */
    public ImpactAssessment performImpactAssessment(String csiId, String jobId) {
        log.info("Performing impact assessment for CSI: {}", csiId);
        
        try {
            // Get discovery results
            DiscoveryResults vmResults = discoveryService.getVmResults(csiId, jobId);
            DiscoveryResults containerResults = discoveryService.getContainerResults(csiId, jobId);
            
            // Analyze certificate usage and dependencies
            ImpactAssessment assessment = assessmentService.analyzeCertificateImpact(
                csiId, vmResults, containerResults);
            
            // Identify risks
            assessment.setRisks(assessmentService.identifyRisks(assessment));
            
            return assessment;
        } catch (Exception e) {
            log.error("Impact assessment failed for CSI: {}", csiId, e);
            throw new AssessmentException("Impact assessment failed", e);
        }
    }
    
    /**
     * Generates comprehensive CLM report
     */
    public ClmReport generateReport(String jobId, ReportFormat format) {
        log.info("Generating report for job: {} in format: {}", jobId, format);
        
        ClmJob job = jobRepository.findById(jobId)
            .orElseThrow(() -> new JobNotFoundException("Job not found: " + jobId));
        
        ClmReport report = reportService.generateReport(job, format);
        
        // Set retention
        report.setExpiresAt(LocalDateTime.now().plusMonths(3));
        
        reportRepository.save(report);
        
        return report;
    }
    
    /**
     * Handles certificate renewal process
     */
    public RenewalResult processCertificateRenewal(String csiId, String certificateId, 
                                                  RenewalRequest request) {
        log.info("Processing renewal for certificate: {} in CSI: {}", certificateId, csiId);
        
        CsiDetails csi = csiRepository.findByCsiId(csiId)
            .orElseThrow(() -> new CsiNotFoundException("CSI not found: " + csiId));
        
        try {
            // Check if auto-renewal is enabled
            if (csi.isAutoDeploymentEnabled() && request.isAutoRenewalEligible()) {
                return renewalService.performAutoRenewal(csiId, certificateId, request);
            } else {
                return renewalService.performManualRenewal(csiId, certificateId, request);
            }
        } catch (Exception e) {
            log.error("Renewal failed for certificate: {}", certificateId, e);
            throw new RenewalException("Certificate renewal failed", e);
        }
    }
    
    /**
     * Handles certificate deployment to repositories
     */
    public DeploymentResult deployCertificate(String csiId, String certificateId, 
                                            DeploymentRequest request) {
        log.info("Deploying certificate: {} for CSI: {}", certificateId, csiId);
        
        CsiDetails csi = csiRepository.findByCsiId(csiId)
            .orElseThrow(() -> new CsiNotFoundException("CSI not found: " + csiId));
        
        DeploymentResult result = new DeploymentResult();
        
        // Deploy to Bitbucket if configured
        if (csi.getBitbucketProjectUrl() != null) {
            result.setBitbucketResult(
                deploymentService.deployToBitbucket(csi, certificateId, request));
        }
        
        // Deploy to GitHub if configured
        if (csi.getGithubProjectUrl() != null) {
            result.setGithubResult(
                deploymentService.deployToGitHub(csi, certificateId, request));
        }
        
        return result;
    }
    
    /**
     * Handles escalation for unresolved action items
     */
    public void processEscalations() {
        log.info("Processing escalations for unresolved action items");
        
        LocalDateTime oneMonthAgo = LocalDateTime.now().minusMonths(1);
        
        // Find unresolved action items older than 1 month
        List<ActionItem> escalationItems = actionItemRepository
            .findByStatusAndCreatedAtBefore(ActionStatus.OPEN, oneMonthAgo);
        
        // Group by CSI
        Map<String, List<ActionItem>> itemsByCsi = escalationItems.stream()
            .collect(Collectors.groupingBy(ActionItem::getCsiId));
        
        itemsByCsi.forEach((csiId, items) -> {
            CsiDetails csi = csiRepository.findByCsiId(csiId).orElse(null);
            if (csi != null && csi.getEscalationMatrix() != null) {
                notificationService.sendEscalationEmail(csi, items);
                
                // Mark items as escalated
                items.forEach(item -> {
                    item.setEscalated(true);
                    item.setStatus(ActionStatus.ESCALATED);
                });
                actionItemRepository.saveAll(items);
            }
        });
    }
}


package com.citi.cert_management.container.service;

import org.springframework.statemachine.StateMachine;
import org.springframework.statemachine.config.StateMachineBuilder;
import org.springframework.statemachine.config.StateMachineFactory;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.util.*;
import java.time.LocalDateTime;
import java.util.concurrent.CompletableFuture;

/**
 * Central job orchestration service for CLM operations
 * Manages asynchronous job execution with state tracking
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class ClmJobOrchestrationService {
    
    private final ClmJobRepository jobRepository;
    private final StateMachineFactory<JobStatus, JobEvent> stateMachineFactory;
    private final Map<JobType, JobExecutor> jobExecutors;
    private final NotificationService notificationService;
    
    /**
     * Initiates a new CLM job with state tracking
     * 
     * @param csiId CSI identifier
     * @param jobType Type of job to execute
     * @param triggeredBy User or system that triggered the job
     * @param parameters Additional job parameters
     * @return Job ID for tracking
     */
    public String initiateJob(String csiId, JobType jobType, String triggeredBy, 
                            Map<String, Object> parameters) {
        log.info("Initiating {} job for CSI: {}", jobType, csiId);
        
        // Create job record
        ClmJob job = new ClmJob();
        job.setJobId(UUID.randomUUID().toString());
        job.setCsiId(csiId);
        job.setJobType(jobType);
        job.setStatus(JobStatus.PENDING);
        job.setStartTime(LocalDateTime.now());
        job.setTriggeredBy(triggeredBy);
        job.setTriggerType(determineT
iggerType(triggeredBy));
        job.setJobParameters(parameters);
        job.setSteps(initializeJobSteps(jobType));
        
        jobRepository.save(job);
        
        // Execute asynchronously
        CompletableFuture.runAsync(() -> executeJob(job))
            .exceptionally(throwable -> {
                handleJobFailure(job.getJobId(), throwable);
                return null;
            });
        
        return job.getJobId();
    }
    
    /**
     * Executes job with state management
     */
    private void executeJob(ClmJob job) {
        StateMachine<JobStatus, JobEvent> stateMachine = 
            stateMachineFactory.getStateMachine(job.getJobId());
        
        try {
            stateMachine.start();
            stateMachine.sendEvent(JobEvent.START);
            
            // Update job status
            updateJobStatus(job.getJobId(), JobStatus.RUNNING);
            
            // Execute job based on type
            JobExecutor executor = jobExecutors.get(job.getJobType());
            if (executor == null) {
                throw new IllegalArgumentException("No executor found for job type: " + job.getJobType());
            }
            
            JobResult result = executor.execute(job);
            
            if (result.isSuccess()) {
                completeJob(job.getJobId(), result);
                stateMachine.sendEvent(JobEvent.COMPLETE);
            } else {
                failJob(job.getJobId(), result.getErrorMessage());
                stateMachine.sendEvent(JobEvent.FAIL);
            }
            
        } catch (Exception e) {
            log.error("Job execution failed for job: {}", job.getJobId(), e);
            failJob(job.getJobId(), e.getMessage());
            stateMachine.sendEvent(JobEvent.FAIL);
        }
    }
    
    /**
     * Marks job as completed and triggers notifications
     */
    private void completeJob(String jobId, JobResult result) {
        ClmJob job = jobRepository.findById(jobId)
            .orElseThrow(() -> new RuntimeException("Job not found: " + jobId));
        
        job.setStatus(JobStatus.COMPLETED);
        job.setEndTime(LocalDateTime.now());
        job.setReportId(result.getReportId());
        
        jobRepository.save(job);
        
        // Trigger notification for completed jobs
        if (shouldSendNotification(job)) {
            notificationService.sendJobCompletionNotification(job);
        }
        
        log.info("Job {} completed successfully", jobId);
    }
    
    /**
     * Retrieves job status and progress
     */
    public JobStatusResponse getJobStatus(String jobId) {
        ClmJob job = jobRepository.findById(jobId)
            .orElseThrow(() -> new RuntimeException("Job not found: " + jobId));
        
        return JobStatusResponse.builder()
            .jobId(job.getJobId())
            .status(job.getStatus())
            .progress(calculateProgress(job))
            .currentStep(getCurrentStep(job))
            .startTime(job.getStartTime())
            .estimatedCompletion(estimateCompletion(job))
            .build();
    }
    
    /**
     * Batch job scheduling for weekly scans
     */
    public void scheduleBatchJobs(LocalDateTime scheduledTime) {
        log.info("Scheduling batch jobs for: {}", scheduledTime);
        
        // Get all active CSIs for batch processing
        List<CsiDetails> activeCsis = csiRepository.findByActiveTrue();
        
        // Group by batch criteria
        Map<String, List<CsiDetails>> batches = groupIntoBatches(activeCsis);
        
        batches.forEach((batchId, csiList) -> {
            String batchJobId = initiateBatchJob(batchId, csiList, scheduledTime);
            log.info("Batch job {} created for {} CSIs", batchJobId, csiList.size());
        });
    }
    
    /**
     * Handles adhoc scan requests
     */
    public String performAdhocScan(String csiId, String triggeredBy) {
        Map<String, Object> parameters = new HashMap<>();
        parameters.put("scanType", "ADHOC");
        parameters.put("fullScan", true);
        
        return initiateJob(csiId, JobType.ADHOC_SCAN, triggeredBy, parameters);
    }
    
    // State Machine Events
    public enum JobEvent {
        START, COMPLETE, FAIL, RETRY, CANCEL
    }
}

/**
 * Interface for job executors
 */
public interface JobExecutor {
    JobResult execute(ClmJob job);
    JobType getJobType();
}

/**
 * Job execution result
 */
@Data
@Builder
public class JobResult {
    private boolean success;
    private String reportId;
    private String errorMessage;
    private Map<String, Object> metadata;
}

package com.citi.cert_management.container.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import lombok.Data;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

/**
 * Enhanced CSI Details entity with CLM functionality support
 * Stores CSI onboarding information and configuration for certificate lifecycle management
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
    private boolean impactAssessmentApplicable;
    private String appName;
    
    // User mandatory inputs
    private String requestorSoeid;
    private String emailDl;
    private String preferredRenewal;
    private String bitbucketProjectUrl;
    private String githubProjectUrl;
    private List<String> notificationEmailList;
    private List<String> escalationMatrix;
    
    // CLM Configuration
    private boolean vmDiscoveryEnabled;
    private boolean containerDiscoveryEnabled;
    private boolean autoDeploymentEnabled;
    private String deploymentSchedule;
    private Map<String, Boolean> clmFeatures;
    
    // Discovery Settings
    private DiscoveryConfig vmDiscoveryConfig;
    private DiscoveryConfig containerDiscoveryConfig;
    
    // Job Tracking
    private String lastJobId;
    private LocalDateTime lastScanDate;
    private LocalDateTime nextScheduledScan;
    private JobStatus lastJobStatus;
}

/**
 * Configuration for discovery operations
 */
@Data
public class DiscoveryConfig {
    private boolean enabled;
    private String schedule;
    private List<String> includePaths;
    private List<String> excludePaths;
    private Map<String, String> additionalParams;
}

/**
 * Job execution tracking entity
 */
@Data
@Document(collection = "clm_jobs")
public class ClmJob {
    @Id
    private String jobId;
    private String csiId;
    private JobType jobType;
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
}

/**
 * Individual job step tracking
 */
@Data
public class JobStep {
    private String stepName;
    private StepStatus status;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private String output;
    private String errorDetails;
    private Map<String, Object> metadata;
}

/**
 * CLM Report entity for storing scan results
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
    
    // Discovery Results
    private DiscoveryResults vmDiscoveryResults;
    private DiscoveryResults containerDiscoveryResults;
    
    // Certificate Inventory
    private CertificateInventory inventory;
    
    // Assessment Results
    private ImpactAssessment impactAssessment;
    private RiskAssessment riskAssessment;
    
    // Action Items
    private List<ActionItem> actionItems;
    
    // Raw data for UI
    private Map<String, Object> rawData;
    
    // Report metadata
    private LocalDateTime expiresAt;
    private boolean emailSent;
    private LocalDateTime emailSentAt;
}

/**
 * Discovery results structure
 */
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

/**
 * Individual certificate details
 */
@Data
public class CertificateDetail {
    private String certificateId;
    private String commonName;
    private String issuer;
    private LocalDateTime notBefore;
    private LocalDateTime notAfter;
    private int daysToExpiry;
    private String location;
    private String format;
    private String purpose;
    private boolean autoRenewEligible;
    private String lastRenewalStatus;
}

/**
 * Action items for follow-up
 */
@Data
public class ActionItem {
    private String actionId;
    private ActionType type;
    private Priority priority;
    private String description;
    private String targetCertificate;
    private LocalDateTime dueDate;
    private ActionStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime resolvedAt;
    private boolean escalated;
}

// Enums
public enum JobType {
    ONBOARDING_SCAN, SCHEDULED_SCAN, ADHOC_SCAN, RENEWAL, DEPLOYMENT
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
