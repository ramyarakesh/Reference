<!-- clm-report-email.html -->
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



GET /api/v1/clm/csi/{csiId}/sync-status

{
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
POST /api/v1/clm/csi/{csiId}/module/{moduleType}/execute
Get Consolidated Report
GET /api/v1/clm/csi/{csiId}/consolidated-report?format=JSON

Job Management
Get Job Status
GET /api/v1/clm/jobs/{jobId}/status
Perform Adhoc Scan
POST /api/v1/clm/csi/{csiId}/adhoc-scan



# CLM (Certificate Lifecycle Management) System Architecture

## Overview
The CLM system provides comprehensive certificate lifecycle management for both VM and container environments. It automates discovery, assessment, renewal, and deployment of certificates while maintaining audit trails and generating reports.

## Key Components

### 1. Job Orchestration
- **Spring State Machine**: Manages job state transitions
- **Async Processing**: Non-blocking job execution
- **Database Tracking**: Persistent job status storage
- **ECS Compatible**: Designed for multi-instance deployment

### 2. Discovery Services
- **VM Discovery**: Ansible playbook integration for server scanning
- **Container Discovery**: Repository scanning and Vault integration
- **Batch Processing**: Weekly scheduled scans
- **Adhoc Scanning**: On-demand discovery operations

### 3. Certificate Management
- **Multiple Formats**: Support for all certificate formats
- **Vault Integration**: Container certificate storage
- **MongoDB Storage**: VM certificate persistence
- **Auto-renewal**: Configurable automatic renewal

### 4. Repository Integration
- **Bitbucket Support**: Clone, commit, PR creation
- **GitHub Support**: Full Git operations
- **Certificate Deployment**: Automated certificate updates
- **Version Control**: Complete audit trail

### 5. Reporting & Notifications
- **HTML Email Reports**: Comprehensive scan results
- **JSON API Reports**: Programmatic access
- **Escalation Management**: Automatic escalation after 1 month
- **Historical Reports**: 3-month retention minimum

## API Endpoints

### CSI Management
- `POST /api/v1/clm/csi/onboard` - Onboard new CSI
- `PUT /api/v1/clm/csi/{csiId}` - Update CSI configuration

### Discovery Operations
- `POST /api/v1/clm/csi/{csiId}/vm-discovery` - Trigger VM discovery
- `POST /api/v1/clm/csi/{csiId}/container-discovery` - Trigger container discovery
- `POST /api/v1/clm/csi/{csiId}/adhoc-scan` - Perform adhoc scan

### Certificate Operations
- `POST /api/v1/clm/csi/{csiId}/renewal` - Initiate renewal
- `POST /api/v1/clm/csi/{csiId}/impact-assessment` - Run assessment

### Reporting
- `GET /api/v1/clm/reports/{jobId}` - Get job report
- `GET /api/v1/clm/csi/{csiId}/inventory-report` - Get inventory
- `GET /api/v1/clm/reports/search` - Search historical reports

### Job Tracking
- `GET /api/v1/clm/jobs/{jobId}/status` - Get job status
- `GET /api/v1/clm/csi/{csiId}/action-items` - Get action items

## Data Flow

1. **Onboarding**: CSI registered → Initial scan triggered
2. **Discovery**: VMs/Containers scanned → Certificates identified
3. **Assessment**: Impact analyzed → Risks identified
4. **Action Items**: Issues tracked → Escalations managed
5. **Renewal**: Certificates renewed → Deployed to repos
6. **Reporting**: Results compiled → Emails sent

## Integration Points

### External Systems
- **Ansible**: Playbook execution via REST API
- **Vault**: Certificate storage and retrieval
- **Bitbucket/GitHub**: Repository operations
- **MongoDB**: Data persistence
- **SMTP**: Email notifications (via Ansible)

### Security Considerations
- **Authentication**: PID username/password for repos
- **Encryption**: Sensitive data encrypted at rest
- **Audit Trail**: Complete operation logging
- **Access Control**: Role-based permissions

## Configuration

### Mandatory User Inputs
- `csiId`: Unique CSI identifier
- `requestorSoeid`: Requestor's SOEID
- `emailDl`: Primary email distribution list
- `email`: Contact email address
- `preferredRenewal`: Renewal provider preference
- `notificationEmailList`: Notification recipients
- `escalationMatrix`: Escalation contacts

### Optional Configuration
- Repository URLs (Bitbucket/GitHub)
- Auto-deployment settings
- Discovery schedules
- Custom thresholds

## Error Handling

- **Retry Logic**: Configurable retry for failed operations
- **Dead Letter Queue**: Failed jobs tracked
- **Graceful Degradation**: Partial failures handled
- **Comprehensive Logging**: Full error details captured

## Monitoring

- **Job Metrics**: Success/failure rates
- **Performance Tracking**: Execution times
- **Resource Usage**: Memory/CPU monitoring
- **Alert Thresholds**: Configurable alerts

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



Test

package com.citi.cert_management.container.e2e;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

/**
 * End-to-end test scenarios
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("e2e")
@DisplayName("CLM System E2E Tests")
class ClmSystemE2ETest {
    
    @LocalServerPort
    private int port;
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    @DisplayName("Complete onboarding flow with all modules")
    void testCompleteOnboardingFlow() {
        // Step 1: Onboard CSI
        EnhancedCsiOnboardingRequest onboardRequest = 
            new CsiOnboardingRequestBuilder()
                .withCsiId("E2E-TEST-001")
                .withModules(ModuleType.CLM, ModuleType.BCM, ModuleType.EOVS)
                .withBcmAutoRemediation()
                .withRepositories(
                    "https://bitbucket.test.com/e2e/repo",
                    "https://github.com/test/e2e-repo"
                )
                .build();
        
        ResponseEntity<OnboardingResponse> onboardResponse = restTemplate.postForEntity(
            "/api/v1/clm/csi/onboard",
            onboardRequest,
            OnboardingResponse.class
        );
        
        assertThat(onboardResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        String jobId = onboardResponse.getBody().getJobId();
        
        // Step 2: Wait for sync completion
        await().atMost(30, TimeUnit.SECONDS).until(() -> {
            ResponseEntity<SyncStatusResponse> syncStatus = restTemplate.getForEntity(
                "/api/v1/clm/csi/{csiId}/sync-status",
                SyncStatusResponse.class,
                "E2E-TEST-001"
            );
            
            return syncStatus.getBody().isCanProceedWithDiscovery();
        });
        
        // Step 3: Wait for job completion
        await().atMost(60, TimeUnit.SECONDS).until(() -> {
            ResponseEntity<JobStatusResponse> jobStatus = restTemplate.getForEntity(
                "/api/v1/clm/jobs/{jobId}/status",
                JobStatusResponse.class,
                jobId
            );
            
            return jobStatus.getBody().getStatus() == JobStatus.COMPLETED;
        });
        
        // Step 4: Verify consolidated report
        ResponseEntity<ConsolidatedReport> reportResponse = restTemplate.getForEntity(
            "/api/v1/clm/csi/{csiId}/consolidated-report?jobId={jobId}",
            ConsolidatedReport.class,
            "E2E-TEST-001",
            jobId
        );
        
        ConsolidatedReport report = reportResponse.getBody();
        assertThat(report.getClmSection().isIncluded()).isTrue();
        assertThat(report.getBcmSection().isIncluded()).isTrue();
        assertThat(report.getEovsSection().isIncluded()).isTrue();
        assertThat(report.getConsolidatedActionItems()).isNotEmpty();
    }
    
    @Test
    @DisplayName("BCM remediation workflow")
    void testBcmRemediationWorkflow() {
        // Setup: Create CSI with BCM exceptions
        String csiId = setupCsiWithBcmExceptions();
        
        // Step 1: Run BCM scan
        ResponseEntity<JobResponse> scanResponse = restTemplate.postForEntity(
            "/api/v1/clm/csi/{csiId}/module/{moduleType}/execute",
            new ModuleOperationRequest("SCAN"),
            JobResponse.class,
            csiId,
            ModuleType.BCM
        );
        
        String scanJobId = scanResponse.getBody().getJobId();
        waitForJobCompletion(scanJobId);
        
        // Step 2: Get BCM report
        ResponseEntity<ConsolidatedReport> reportResponse = restTemplate.getForEntity(
            "/api/v1/clm/csi/{csiId}/consolidated-report?jobId={jobId}",
            ConsolidatedReport.class,
            csiId,
            scanJobId
        );
        
        BcmReportSection bcmReport = reportResponse.getBody().getBcmSection();
        assertThat(bcmReport.getExceptionsFound()).isGreaterThan(0);
        
        // Step 3: Trigger remediation
        BcmRemediationRequest remediationRequest = new BcmRemediationRequest();
        remediationRequest.setExceptionIds(
            bcmReport.getTopExceptions().stream()
                .map(BcmExceptionSummary::getExceptionId)
                .collect(Collectors.toList())
        );
        remediationRequest.setApprovedBy("e2e-tester");
        remediationRequest.setReason("E2E test remediation");
        
        ResponseEntity<RemediationResponse> remediationResponse = 
            restTemplate.postForEntity(
                "/api/v1/clm/csi/{csiId}/bcm/remediate",
                remediationRequest,
                RemediationResponse.class,
                csiId
            );
        
        assertThat(remediationResponse.getBody().isSuccess()).isTrue();
    }
    
    @Test
    @DisplayName("Certificate expiry and renewal workflow")
    void testCertificateRenewalWorkflow() {
        // Setup: Create CSI with expiring certificates
        String csiId = setupCsiWithExpiringCertificates();
        
        // Step 1: Run CLM discovery
        ResponseEntity<JobResponse> discoveryResponse = restTemplate.postForEntity(
            "/api/v1/clm/csi/{csiId}/module/{moduleType}/execute",
            new ModuleOperationRequest("DISCOVER"),
            JobResponse.class,
            csiId,
            ModuleType.CLM
        );
        
        waitForJobCompletion(discoveryResponse.getBody().getJobId());
        
        // Step 2: Get inventory report
        ResponseEntity<InventoryReport> inventoryResponse = restTemplate.getForEntity(
            "/api/v1/clm/csi/{csiId}/inventory-report",
            InventoryReport.class,
            csiId
        );
        
        InventoryReport inventory = inventoryResponse.getBody();
        assertThat(inventory.getCertsByCategory().get("30_DAYS")).isGreaterThan(0);
        
        // Step 3: Initiate renewal for expiring certificate
        String expiringCertId = inventory.getExpiringCertificates().get(0).getCertificateId();
        
        RenewalRequest renewalRequest = new RenewalRequest();
        renewalRequest.setCertificateId(expiringCertId);
        renewalRequest.setAutoRenewal(true);
        renewalRequest.setRenewalProvider("TEST-CA");
        
        ResponseEntity<RenewalResponse> renewalResponse = restTemplate.postForEntity(
            "/api/v1/clm/csi/{csiId}/renewal",
            renewalRequest,
            RenewalResponse.class,
            csiId
        );
        
        assertThat(renewalResponse.getBody().getStatus()).isEqualTo("INITIATED");
    }
    
    @Test
    @DisplayName("Escalation workflow for unresolved items")
    void testEscalationWorkflow() {
        // Setup: Create CSI with old unresolved action items
        String csiId = setupCsiWithOldActionItems();
        
        // Step 1: Get action items
        ResponseEntity<ActionItem[]> actionItemsResponse = restTemplate.getForEntity(
            "/api/v1/clm/csi/{csiId}/action-items?status=OPEN",
            ActionItem[].class,
            csiId
        );
        
        ActionItem[] openItems = actionItemsResponse.getBody();
        assertThat(openItems).hasSizeGreaterThan(0);
        
        // Step 2: Trigger escalation job
        restTemplate.postForEntity(
            "/api/v1/clm/escalations/process",
            null,
            Void.class
        );
        
        // Step 3: Verify items marked as escalated
        await().atMost(10, TimeUnit.SECONDS).until(() -> {
            ResponseEntity<ActionItem[]> escalatedResponse = restTemplate.getForEntity(
                "/api/v1/clm/csi/{csiId}/action-items?status=ESCALATED",
                ActionItem[].class,
                csiId
            );
            
            return escalatedResponse.getBody().length > 0;
        });
    }
}

package com.citi.cert_management.container.test.utils;

/**
 * Test data factory
 */
public class TestDataFactory {
    
    public static ClmReport createCompleteReport() {
        ClmReport report = new ClmReport();
        report.setReportId(UUID.randomUUID().toString());
        report.setGeneratedAt(LocalDateTime.now());
        report.setReportType(ReportType.ONBOARDING);
        
        // CLM Results
        ClmResults clmResults = new ClmResults();
        clmResults.setVmDiscoveryResults(createDiscoveryResults(50, 25));
        clmResults.setContainerDiscoveryResults(createDiscoveryResults(20, 15));
        clmResults.setInventory(createCertificateInventory(40));
        report.setClmResults(clmResults);
        
        // BCM Results
        BcmResults bcmResults = new BcmResults();
        bcmResults.setServersScanned(50);
        bcmResults.setExceptionsFound(10);
        bcmResults.setExceptionsByType(Map.of(
            "FILE_PERMISSION", 5,
            "SERVICE_CONFIG", 3,
            "USER_ACCOUNT", 2
        ));
        report.setBcmResults(bcmResults);
        
        // EOVS Results
        EovsResults eovsResults = new EovsResults();
        eovsResults.setVulnerabilitiesFound(15);
        eovsResults.setVulnerabilitiesBySeverity(Map.of(
            "CRITICAL", 2,
            "HIGH", 5,
            "MEDIUM", 8
        ));
        report.setEovsResults(eovsResults);
        
        // Action Items
        report.setActionItems(createActionItems(5));
        
        return report;
    }
    
    public static DiscoveryResults createDiscoveryResults(int serversScanned, int certsFound) {
        DiscoveryResults results = new DiscoveryResults();
        results.setTotalServersScanned(serversScanned);
        results.setTotalCertsFound(certsFound);
        results.setCertsByType(Map.of(
            "SSL/TLS", certsFound * 60 / 100,
            "Code Signing", certsFound * 30 / 100,
            "Other", certsFound * 10 / 100
        ));
        return results;
    }
    
    public static CertificateInventory createCertificateInventory(int totalCerts) {
        CertificateInventory inventory = new CertificateInventory();
        inventory.setTotalCertificates(totalCerts);
        inventory.setCertsByExpiryCategory(Map.of(
            "EXPIRED", 2,
            "30_DAYS", 5,
            "60_DAYS", 8,
            "90_DAYS", 10,
            "VALID", totalCerts - 25
        ));
        
        List<CertificateDetail> certificates = new ArrayList<>();
        for (int i = 0; i < totalCerts; i++) {
            certificates.add(createCertificateDetail(i));
        }
        inventory.setCertificates(certificates);
        
        return inventory;
    }
    
    public static List<ActionItem> createActionItems(int count) {
        List<ActionItem> items = new ArrayList<>();
        Priority[] priorities = Priority.values();
        ActionType[] types = ActionType.values();
        
        for (int i = 0; i < count; i++) {
            ActionItem item = new ActionItem();
            item.setActionId("ACTION-" + i);
            item.setType(types[i % types.length]);
            item.setPriority(priorities[i % priorities.length]);
            item.setDescription("Action required for " + item.getType());
            item.setDueDate(LocalDateTime.now().plusDays(7 * (i + 1)));
            item.setStatus(ActionStatus.OPEN);
            item.setCreatedAt(LocalDateTime.now().minusDays(i));
            items.add(item);
        }
        return items;
    }
}

/**
 * Custom assertions for CLM domain objects
 */
public class ClmAssertions {
    
    public static void assertValidCsiDetails(CsiDetails csi) {
        assertThat(csi).isNotNull();
        assertThat(csi.getCsiId()).isNotBlank();
        assertThat(csi.getRequestorSoeid()).isNotBlank();
        assertThat(csi.getEmailDl()).contains("@");
        assertThat(csi.getModuleSubscriptions()).isNotEmpty();
        assertThat(csi.getModuleSubscriptions()).hasSizeGreaterThanOrEqualTo(1);
    }
    
    public static void assertSyncCompleted(CsiDetails csi) {
        assertThat(csi.getServerSyncStatus()).isNotNull();
        assertThat(csi.getServerSyncStatus().getState()).isEqualTo(SyncState.COMPLETED);
        assertThat(csi.getCertificateSyncStatus().getState()).isEqualTo(SyncState.COMPLETED);
        assertThat(csi.getRepositorySyncStatus().getState()).isEqualTo(SyncState.COMPLETED);
        assertThat(csi.getLastSuccessfulSync()).isNotNull();
    }
    
    public static void assertJobSuccessful(ClmJob job) {
        assertThat(job.getStatus()).isEqualTo(JobStatus.COMPLETED);
        assertThat(job.getErrorMessage()).isNull();
        assertThat(job.getEndTime()).isNotNull();
        assertThat(job.getEndTime()).isAfter(job.getStartTime());
        assertThat(job.getReportId()).isNotNull();
    }
}






