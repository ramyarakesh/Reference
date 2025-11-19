# CLM Enhancement - IO API Integration & Workflow Redesign

## Project Structure

```
com.citi.clm
├── config/
│   ├── IOApiConfig.java
│   ├── WorkflowConfig.java
│   └── SchedulerConfig.java
├── entity/
│   ├── CsiDetails.java (existing - enhanced)
│   ├── CertificateDetails.java (existing - enhanced)
│   ├── WorkflowDefinition.java
│   ├── WorkflowInstance.java
│   ├── WorkflowStep.java
│   ├── ExecutionLog.java
│   ├── TransactionLogs.java (existing)
│   └── AnsibleResultRequest.java (existing - enhanced)
├── enums/
│   ├── ModuleType.java (existing)
│   ├── ExecutionEnvironment.java
│   ├── WorkflowStatus.java
│   ├── StepStatus.java
│   └── ProductType.java
├── repository/
│   ├── CsiDetailsRepository.java
│   ├── CertificateDetailsRepository.java
│   ├── WorkflowDefinitionRepository.java
│   ├── WorkflowInstanceRepository.java
│   ├── ExecutionLogRepository.java
│   └── TransactionLogsRepository.java
├── service/
│   ├── io/
│   │   ├── IOApiService.java
│   │   ├── IOAuthService.java
│   │   └── IOExecutionService.java
│   ├── workflow/
│   │   ├── WorkflowOrchestrator.java
│   │   ├── WorkflowStateManager.java
│   │   ├── StepExecutor.java
│   │   └── WorkflowValidator.java
│   ├── CsiValidationService.java
│   ├── RenewalSchedulerService.java
│   ├── ResultCallbackService.java
│   ├── NotificationService.java
│   └── TransactionLogService.java
├── controller/
│   ├── ResultCallbackController.java
│   ├── WorkflowAdminController.java
│   └── ExecutionLogController.java
├── dto/
│   ├── io/
│   │   ├── IOAuthRequest.java
│   │   ├── IOAuthResponse.java
│   │   ├── IOExecuteRequest.java
│   │   ├── IOExecuteResponse.java
│   │   ├── IOOrderStatusResponse.java
│   │   ├── IOPodListResponse.java
│   │   └── IOPodLogResponse.java
│   ├── ResultCallbackRequest.java
│   └── WorkflowTriggerRequest.java
└── exception/
    ├── WorkflowException.java
    ├── IOApiException.java
    └── CsiValidationException.java
```

## Key Features

1. **IO API Integration** - Complete 5-stage execution (auth, execute, status, pods, logs)
2. **Dual Workflow Support** - New workflow (WAS/IHS) and Old workflow (others)
3. **CSI Validation** - Auto-renewal/deployment flags, execution environment preference
4. **Fire-and-Forget with Callback** - Playbooks call /result endpoint
5. **Transaction Logging** - Comprehensive audit trail
6. **Manual Retry** - Support for retrying failed steps

## Configuration

All playbook names and IO parameters are configurable via `application.yml` and database.

package com.citi.clm.enums;

public enum ExecutionEnvironment {
    IO,    // Inventory Orchestrator (default)
    AAP    // Ansible Automation Platform (Ansible Tower)
}
package com.citi.clm.enums;

public enum WorkflowStatus {
    PENDING,           // Workflow created but not started
    IN_PROGRESS,       // Workflow is executing
    COMPLETED,         // All steps completed successfully
    FAILED,            // One or more steps failed
    PAUSED,            // Waiting for manual intervention
    CANCELLED,         // Workflow cancelled by user
    ROLLBACK_IN_PROGRESS,  // Rollback is executing
    ROLLED_BACK        // Rollback completed
}
package com.citi.clm.enums;

public enum StepStatus {
    PENDING,           // Step not yet started
    EXECUTING,         // Step is currently executing
    COMPLETED,         // Step completed successfully
    FAILED,            // Step failed
    SKIPPED,           // Step was skipped
    RETRY_PENDING      // Step failed and awaiting manual retry
}
package com.citi.clm.enums;

public enum WorkflowType {
    NEW_WORKFLOW,    // For WAS, IHS - Procurement, Deployment, Restart
    OLD_WORKFLOW     // For other product types - CMP Creation, Import, Push, CR Validation, Restart
}
package com.citi.clm.enums;

public enum PlaybookExecutionEnvironment {
    IO,    // Inventory Orchestrator (default)
    AAP    // Ansible Automation Platform
}
package com.citi.clm.entity;

import com.citi.clm.enums.PlaybookExecutionEnvironment;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ModuleConfiguration {
    
    // CLM Configuration
    @Builder.Default
    private boolean autoRenewalEnabled = true;
    
    @Builder.Default
    private boolean autoDeploymentEnabled = true;
    
    private String preferredRenewalProvider; // PROVISIONING_PORTAL, CMP
    
    @Builder.Default
    private PlaybookExecutionEnvironment vmPreferredExecutionEnv = PlaybookExecutionEnvironment.IO;
    
    private Integer customExpiryThresholdDays; // Default 60 if null
    
    // BCM Configuration (not relevant for this enhancement)
    private String bcmMode;
    private java.util.List<String> bcmExceptionTypes;
    private boolean bcmAutoRemediationEnabled;
    
    // EOVS Configuration (not relevant for this enhancement)
    private String eovsMode;
    private java.util.List<String> includePaths;
    private java.util.List<String> excludePaths;
    private boolean eovsUserAcknowledgmentRequired;
    private java.util.List<String> eovsChecks;
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
@Document(collection = "certificate_details")
public class CertificateDetails {
    
    @Id
    private String id;
    
    private String uniqueNumber;
    
    @Field("CSI")
    private int csi;
    
    private String envType;
    private String productType;  // WAS, IHS, Apache, etc.
    
    @Field("URL")
    private String url;
    
    private String serialNumber;
    
    @Field("SAN")
    private String san;
    
    private String fileName;
    private String keyDBType;
    private String keyDBPath;
    private String targetHostname;
    private String cmpStatus;
    private String group;
    
    @Field("CSR")
    private String csr;
    
    private String newCSR;
    private String overallStatus;
    private Date createdDate;
    private String updatedDate;
    private String cmpId;
    private String snowReqNo;
    private String expiry;
    private String checkSum;
    private String auditCheck;
    private String randomNumber;
    private String certRole;
    private String partnerCert;
    private String region;
    private String scheduledJob;
    private String envCategory;
    private String isAutoscanned;
    private Date transactionDate;
    private String dmgrPath;
    private String instance;
    private String rPath;
    
    // Workflow tracking
    private String currentWorkflowId;
    private String renewalStatus;
    private Date lastRenewalAttempt;
    private Date renewalCompletedDate;
}

package com.citi.clm.entity;

import com.citi.clm.enums.WorkflowType;
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
@Document(collection = "workflow_definitions")
public class WorkflowDefinition {
    
    @Id
    private String id;
    
    private String name;
    private String description;
    private WorkflowType workflowType;
    
    // Product types this workflow applies to (e.g., "WAS", "IHS" for new workflow)
    private List<String> applicableProductTypes;
    
    // Ordered list of step definitions
    private List<WorkflowStepDefinition> steps;
    
    // Rollback step (optional)
    private WorkflowStepDefinition rollbackStep;
    
    private boolean active;
    private int version;
    
    private Date createdDate;
    private Date updatedDate;
    private String createdBy;
    private String updatedBy;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class WorkflowStepDefinition {
        private int stepOrder;
        private String stepName;
        private String stepDescription;
        private String playbookName;
        private String ioAction;  // AUTOSCAN, PROCUREMENT, DEPLOYMENT, etc.
        private boolean requiresDeploymentCheck; // If true, check autoDeploymentEnabled
        private boolean isOptional;
        private int timeoutMinutes;
    }
}

package com.citi.clm.entity;

import com.citi.clm.enums.ExecutionEnvironment;
import com.citi.clm.enums.StepStatus;
import com.citi.clm.enums.WorkflowStatus;
import com.citi.clm.enums.WorkflowType;
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
@Document(collection = "workflow_instances")
public class WorkflowInstance {
    
    @Id
    private String id;
    
    private String workflowDefinitionId;
    private WorkflowType workflowType;
    private String certificateId;
    private String certificateUniqueNumber;
    private Integer csi;
    private String productType;
    
    private WorkflowStatus status;
    private int currentStepIndex;
    
    // Execution environment resolved for this workflow
    private ExecutionEnvironment executionEnvironment;
    
    // Steps with their execution status
    private List<WorkflowStepInstance> steps;
    
    // Rollback tracking
    private boolean rollbackTriggered;
    private WorkflowStepInstance rollbackStep;
    
    // Timing
    private Date createdDate;
    private Date startedDate;
    private Date completedDate;
    private Date lastUpdatedDate;
    
    // Metadata
    private String triggeredBy; // SCHEDULER or userId
    private String lastUpdatedBy;
    private int retryCount;
    private String failureReason;
    
    // Transaction tracking
    private String transactionId;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class WorkflowStepInstance {
        private int stepOrder;
        private String stepName;
        private String playbookName;
        private String ioAction;
        private StepStatus status;
        private boolean requiresDeploymentCheck;
        
        // IO API tracking
        private String orderId;
        private String podName;
        
        // Execution details
        private Date startedDate;
        private Date completedDate;
        private String executionLog;
        private String errorMessage;
        private int retryCount;
        
        // Result from playbook callback
        private Object result;
        private String connectionStatus;
        private String executionStatus;
    }
}

package com.citi.clm.entity;

import com.citi.clm.enums.ExecutionEnvironment;
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
@Document(collection = "AnsibleResultRequest")
public class AnsibleResultRequest {
    
    @Id
    private String id;
    
    private String servername;
    private String transactionId;
    private Object result;
    private String module;
    private String connectionStatus;
    private String executionStatus;
    private String executionLog;
    private List<AnsibleChronology> chronology;
    private String ansibleRefId;
    private Date createdDate;
    private Date updatedDate;
    private String userId;
    
    // New fields for IO API support
    private ExecutionEnvironment executionEnvironment; // IO or AAP
    private String orderId;       // IO order ID
    private String podName;       // IO pod name
    private String workflowInstanceId;
    private Integer stepOrder;
    private String playbookName;
    private Integer csi;
    private String certificateId;
    
    // IO specific details
    private String ioOrderStatus;
    private Date ioCreatedDate;
    private Date ioLastModifiedDate;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class AnsibleChronology {
        private String action;
        private String status;
        private Date timestamp;
        private String details;
    }
}

package com.citi.clm.entity;

import com.citi.clm.enums.ExecutionEnvironment;
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
@Document(collection = "AnsibleResultRequest")
public class AnsibleResultRequest {
    
    @Id
    private String id;
    
    private String servername;
    private String transactionId;
    private Object result;
    private String module;
    private String connectionStatus;
    private String executionStatus;
    private String executionLog;
    private List<AnsibleChronology> chronology;
    private String ansibleRefId;
    private Date createdDate;
    private Date updatedDate;
    private String userId;
    
    // New fields for IO API support
    private ExecutionEnvironment executionEnvironment; // IO or AAP
    private String orderId;       // IO order ID
    private String podName;       // IO pod name
    private String workflowInstanceId;
    private Integer stepOrder;
    private String playbookName;
    private Integer csi;
    private String certificateId;
    
    // IO specific details
    private String ioOrderStatus;
    private Date ioCreatedDate;
    private Date ioLastModifiedDate;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class AnsibleChronology {
        private String action;
        private String status;
        private Date timestamp;
        private String details;
    }
}

package com.citi.clm.entity;

import com.citi.clm.enums.ExecutionEnvironment;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;
import io.swagger.v3.oas.annotations.media.Schema;

import java.util.Date;
import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Document(collection = "Transaction_Logs_Inventory")
public class TransactionLogs {
    
    @Id
    private Long transactionId;
    
    @Schema(name = "date", example = "05/04/2023", required = true)
    private Date date;
    
    @Schema(name = "userId", example = "RP67333", required = true)
    private String userId;
    
    @Field("api_accessed")
    @Schema(name = "apiaccessed", example = "Yes/No", required = true)
    private String apiaccessed;
    
    @Schema(name = "arguments", example = "the arguments.", required = true)
    private String arguments;
    
    @Schema(name = "response", example = "the result.", required = true)
    private Object response;
    
    @Schema(name = "responseText", example = "the result.", required = true)
    private Object responseText;
    
    private List<ProviderInfo> providerinfo;
    
    // New fields for workflow tracking
    private String workflowInstanceId;
    private String certificateId;
    private Integer csi;
    private String stepName;
    private String action;
    private ExecutionEnvironment executionEnvironment;
    private String orderId;
    private String playbookName;
    private String status;
    private String errorMessage;
    
    // IO API specific logging
    private IOApiCallDetails ioApiCallDetails;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class ProviderInfo {
        private String providerName;
        private String status;
        private String message;
    }
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class IOApiCallDetails {
        private String endpoint;
        private String method;
        private Object requestBody;
        private Object responseBody;
        private int httpStatus;
        private long durationMs;
        private Date timestamp;
    }
}

DTO--------
