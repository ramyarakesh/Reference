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

package com.citi.clm.dto.io;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class IOAuthRequest {
    private String grantType;
    private String scope;
}

package com.citi.clm.dto.io;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class IOAuthResponse {
    
    @JsonProperty("token_type")
    private String tokenType;
    
    @JsonProperty("access_token")
    private String accessToken;
    
    @JsonProperty("expires_in")
    private String expiresIn;
    
    private String scope;
}

package com.citi.clm.dto.io;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

import java.util.List;
import java.util.Map;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class IOExecuteRequest {
    
    private String csiAppId;
    private String environment;  // DEV, UAT, PROD
    private String principal;    // TODO: Provide principal ID format
    private String target;       // Target server hostname
    private String ticket;       // SNOW ticket number
    
    private List<TaskDefinition> tasks;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class TaskDefinition {
        private String identifier;  // IOACTION
        private String programmaticName;  // e.g., platformforyou_clm
        private String action;  // AUTOSCAN, PROCUREMENT, DEPLOYMENT, etc.
        private Map<String, String> parameters;
    }
    
    // URLs for callback and inventory
    @JsonProperty("cmtHostURL")
    private String cmtHostUrl;  // TODO: Provide CMT host URL
    
    @JsonProperty("inventory_HostURL")
    private String inventoryHostUrl;  // TODO: Provide inventory host URL
    
    private String transactionId;
    
    @JsonProperty("token_value")
    private String tokenValue;  // PFY Bearer token for callback
}

package com.citi.clm.dto.io;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

import java.util.List;
import java.util.Map;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class IOExecuteResponse {
    
    private String orderId;
    private String status;  // PENDING, PROCESSING, etc.
    private String message;
    
    private List<TaskResponse> tasks;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class TaskResponse {
        private String action;
        private String name;
        private String version;
        private String componentIdentityId;
        private List<String> inventory;
        
        @JsonProperty("inventory_HostURL")
        private String inventoryHostUrl;
        
        @JsonProperty("cmtHostURL")
        private String cmtHostUrl;
        
        private String hostUrl;
        
        @JsonProperty("token_value")
        private String tokenValue;
        
        private Map<String, Object> properties;
        
        @JsonProperty("server-names")
        private String serverNames;
        
        private String scriptPath;
    }
    
    @JsonProperty("Module")
    private String module;
    
    private String transactionId;
}

package com.citi.clm.dto.io;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class IOOrderStatusResponse {
    
    private String id;
    private String status;  // SUCCESS, ERROR, PENDING, PROCESSING
    private String message;
    
    private OrderData data;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class OrderData {
        private String csiAppId;
        private String orderStatus;  // ERROR, SUCCESS, PENDING
        private String orderId;
        private String message;
        private String createdDate;
        private String lastModifiedDate;
    }
}

package com.citi.clm.dto.io;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class IOPodListResponse {
    
    private List<PodInfo> pods;
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
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

package com.citi.clm.dto.io;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class IOPodLogResponse {
    
    private String podName;
    private String log;
    private String status;
    private String timestamp;
}

package com.citi.clm.dto;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ResultCallbackRequest {
    
    private String id;
    private String servername;
    private String transactionId;
    private Object result;
    private String module;
    private String connectionStatus;
    private String executionStatus;
    private String executionLog;
    private List<ChronologyEntry> chronology;
    private String ansibleRefId;
    private String userId;
    
    // New fields for workflow integration
    private String workflowInstanceId;
    private Integer stepOrder;
    private String orderId;  // IO order ID
    private String podName;  // IO pod name
    
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class ChronologyEntry {
        private String action;
        private String status;
        private String timestamp;
        private String details;
    }
}

package com.citi.clm.dto;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class WorkflowTriggerRequest {
    
    private String certificateId;
    private String triggeredBy;
    private boolean forceRenewal;  // Bypass auto-renewal check
    private String reason;
}

package com.citi.clm.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Data
@Configuration
@ConfigurationProperties(prefix = "clm.io-api")
public class IOApiConfig {
    
    // Base URLs
    private String baseUrl = "https://b2b.cti.otservices.citigroup.net";
    private String authUrl = "/2loauth-auth-ioservices/v1/oauth/token";
    private String executeUrl = "/io-service-gateway/v1/order/v3/execute-automations";
    private String orderStatusUrl = "/io-service-gateway/v1/order-info";
    private String listPodsUrl = "/io-service-gateway/v1/execution/list-execution-pods";
    private String downloadLogUrl = "/io-service-gateway/v1/execution/download-execution-log";
    
    // Authentication
    private String basicAuthCredentials; // Base64 encoded credentials
    private String grantType = "client_credentials";
    private String scope = "/retrieve-2loauth-auth/v1";
    
    // Token caching
    private int tokenCacheMinutes = 50; // Token expires in ~60 min, refresh at 50
    
    // Timeouts
    private int connectTimeoutSeconds = 30;
    private int readTimeoutSeconds = 60;
    
    // Retry configuration
    private int maxRetries = 3;
    private int retryDelaySeconds = 5;
    
    // Callback URLs
    private String cmtHostUrl; // TODO: Provide CLM callback URL
    private String inventoryHostUrl; // TODO: Provide inventory host URL
    
    // Programmatic name for CLM
    private String programmaticName = "platformforyou_clm";
    
    // Headers
    private boolean bypassOverride = true;
    private String executionMode = "GRAPH";
    private String validationMode = "DEBUG";
}

package com.citi.clm.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Data
@Configuration
@ConfigurationProperties(prefix = "clm.workflow")
public class WorkflowConfig {
    
    // Default expiry threshold in days
    private int defaultExpiryThresholdDays = 60;
    
    // Product types that use new workflow
    private List<String> newWorkflowProductTypes = List.of("WAS", "IHS");
    
    // New workflow playbook configuration (Procurement, Deployment, Restart)
    private PlaybookConfig newWorkflow = new PlaybookConfig();
    
    // Old workflow playbook configuration (CMP Creation, Import, Push, CR Validation, Restart)
    private PlaybookConfig oldWorkflow = new PlaybookConfig();
    
    // Notification playbook
    private String notificationPlaybook = "clm_notification"; // TODO: Provide actual playbook name
    
    // Rollback playbook
    private String rollbackPlaybook = "clm_rollback"; // TODO: Provide actual playbook name
    
    // Step timeout in minutes
    private int stepTimeoutMinutes = 30;
    
    @Data
    public static class PlaybookConfig {
        private Map<String, StepConfig> steps = new HashMap<>();
        
        @Data
        public static class StepConfig {
            private String playbookName;
            private String ioAction;
            private int order;
            private boolean requiresDeploymentCheck;
            private boolean isOptional;
            private int timeoutMinutes = 30;
        }
    }
    
    // Initialize default playbook names
    public void initDefaults() {
        // New workflow defaults
        newWorkflow.getSteps().put("procurement", createStep("clm_procurement", "PROCUREMENT", 1, false, false));
        newWorkflow.getSteps().put("deployment", createStep("clm_deployment", "DEPLOYMENT", 2, true, false));
        newWorkflow.getSteps().put("restart", createStep("clm_restart", "RESTART", 3, true, false));
        
        // Old workflow defaults
        oldWorkflow.getSteps().put("cmp_creation", createStep("clm_cmp_creation", "CMP_CREATION", 1, false, false));
        oldWorkflow.getSteps().put("import", createStep("clm_import", "IMPORT", 2, false, false));
        oldWorkflow.getSteps().put("push", createStep("clm_push", "PUSH", 3, true, false));
        oldWorkflow.getSteps().put("cr_validation", createStep("clm_cr_validation", "CR_VALIDATION", 4, false, false));
        oldWorkflow.getSteps().put("restart", createStep("clm_restart", "RESTART", 5, true, false));
    }
    
    private PlaybookConfig.StepConfig createStep(String playbookName, String ioAction, int order, 
                                                   boolean requiresDeploymentCheck, boolean isOptional) {
        PlaybookConfig.StepConfig step = new PlaybookConfig.StepConfig();
        step.setPlaybookName(playbookName);
        step.setIoAction(ioAction);
        step.setOrder(order);
        step.setRequiresDeploymentCheck(requiresDeploymentCheck);
        step.setIsOptional(isOptional);
        return step;
    }
}

package com.citi.clm.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableScheduling;

@Data
@Configuration
@EnableScheduling
@ConfigurationProperties(prefix = "clm.scheduler")
public class SchedulerConfig {
    
    // Renewal check scheduler
    private String renewalCheckCron = "0 0 2 * * ?"; // Daily at 2 AM
    
    // Batch size for processing certificates
    private int batchSize = 100;
    
    // Maximum concurrent workflow executions
    private int maxConcurrentWorkflows = 10;
    
    // Stuck workflow cleanup (workflows in progress for too long)
    private int stuckWorkflowTimeoutHours = 24;
    
    // Retry configuration
    private int maxRetryAttempts = 3;
    private int retryDelayMinutes = 15;
    
    // Enable/disable scheduler
    private boolean enabled = true;
}

package com.citi.clm.repository;

import com.citi.clm.entity.CsiDetails;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface CsiDetailsRepository extends MongoRepository<CsiDetails, String> {
    
    Optional<CsiDetails> findByCSI(Integer csi);
    
    @Query("{'CSI': ?0, 'moduleSubscriptions.module': 'CLM_VM', 'moduleSubscriptions.isSelected': true}")
    Optional<CsiDetails> findByCSIWithCLMEnabled(Integer csi);
    
    boolean existsByCSI(Integer csi);
}

package com.citi.clm.repository;

import com.citi.clm.entity.CertificateDetails;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.Date;
import java.util.List;
import java.util.Optional;

@Repository
public interface CertificateDetailsRepository extends MongoRepository<CertificateDetails, String> {
    
    Optional<CertificateDetails> findByUniqueNumber(String uniqueNumber);
    
    List<CertificateDetails> findByCsi(int csi);
    
    List<CertificateDetails> findByProductType(String productType);
    
    // Find certificates expiring within given days that are not already in renewal
    @Query("{'expiry': {$lte: ?0}, 'renewalStatus': {$nin: ['IN_PROGRESS', 'COMPLETED', 'PENDING']}}")
    List<CertificateDetails> findCertificatesForRenewal(String expiryDate);
    
    // Find by CSI and expiry range
    @Query("{'csi': ?0, 'expiry': {$lte: ?1}, 'renewalStatus': {$nin: ['IN_PROGRESS', 'COMPLETED', 'PENDING']}}")
    List<CertificateDetails> findCertificatesForRenewalByCsi(int csi, String expiryDate);
    
    // Find certificates with active workflow
    List<CertificateDetails> findByCurrentWorkflowIdIsNotNull();
    
    // Find by renewal status
    List<CertificateDetails> findByRenewalStatus(String renewalStatus);
}

package com.citi.clm.repository;

import com.citi.clm.entity.WorkflowDefinition;
import com.citi.clm.enums.WorkflowType;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface WorkflowDefinitionRepository extends MongoRepository<WorkflowDefinition, String> {
    
    Optional<WorkflowDefinition> findByWorkflowTypeAndActiveTrue(WorkflowType workflowType);
    
    @Query("{'applicableProductTypes': ?0, 'active': true}")
    Optional<WorkflowDefinition> findByProductTypeAndActiveTrue(String productType);
    
    List<WorkflowDefinition> findByActiveTrue();
    
    Optional<WorkflowDefinition> findByNameAndActiveTrue(String name);
}

package com.citi.clm.repository;

import com.citi.clm.entity.WorkflowInstance;
import com.citi.clm.enums.WorkflowStatus;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.Date;
import java.util.List;
import java.util.Optional;

@Repository
public interface WorkflowInstanceRepository extends MongoRepository<WorkflowInstance, String> {
    
    Optional<WorkflowInstance> findByCertificateId(String certificateId);
    
    List<WorkflowInstance> findByCsi(Integer csi);
    
    List<WorkflowInstance> findByStatus(WorkflowStatus status);
    
    // Find active workflows (not completed, failed, or cancelled)
    @Query("{'status': {$in: ['PENDING', 'IN_PROGRESS', 'PAUSED']}}")
    List<WorkflowInstance> findActiveWorkflows();
    
    // Find stuck workflows (in progress for too long)
    @Query("{'status': 'IN_PROGRESS', 'startedDate': {$lt: ?0}}")
    List<WorkflowInstance> findStuckWorkflows(Date cutoffDate);
    
    // Find by transaction ID
    Optional<WorkflowInstance> findByTransactionId(String transactionId);
    
    // Find workflows pending retry
    @Query("{'status': 'PAUSED', 'steps.status': 'RETRY_PENDING'}")
    List<WorkflowInstance> findWorkflowsPendingRetry();
    
    // Count active workflows
    @Query(value = "{'status': {$in: ['PENDING', 'IN_PROGRESS']}}", count = true)
    long countActiveWorkflows();
    
    // Find by certificate unique number
    Optional<WorkflowInstance> findByCertificateUniqueNumber(String uniqueNumber);
}

package com.citi.clm.repository;

import com.citi.clm.entity.TransactionLogs;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.Date;
import java.util.List;

@Repository
public interface TransactionLogsRepository extends MongoRepository<TransactionLogs, Long> {
    
    List<TransactionLogs> findByWorkflowInstanceId(String workflowInstanceId);
    
    List<TransactionLogs> findByCertificateId(String certificateId);
    
    List<TransactionLogs> findByCsi(Integer csi);
    
    List<TransactionLogs> findByOrderId(String orderId);
    
    @Query("{'date': {$gte: ?0, $lte: ?1}}")
    List<TransactionLogs> findByDateRange(Date startDate, Date endDate);
    
    List<TransactionLogs> findByUserId(String userId);
    
    // Find latest transaction ID for auto-increment
    @Query(value = "{}", sort = "{'transactionId': -1}")
    List<TransactionLogs> findTopByOrderByTransactionIdDesc();
}

package com.citi.clm.repository;

import com.citi.clm.entity.AnsibleResultRequest;
import com.citi.clm.enums.ExecutionEnvironment;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface AnsibleResultRequestRepository extends MongoRepository<AnsibleResultRequest, String> {
    
    List<AnsibleResultRequest> findByTransactionId(String transactionId);
    
    List<AnsibleResultRequest> findByWorkflowInstanceId(String workflowInstanceId);
    
    Optional<AnsibleResultRequest> findByOrderId(String orderId);
    
    List<AnsibleResultRequest> findByCsi(Integer csi);
    
    List<AnsibleResultRequest> findByCertificateId(String certificateId);
    
    List<AnsibleResultRequest> findByExecutionEnvironment(ExecutionEnvironment executionEnvironment);
    
    Optional<AnsibleResultRequest> findByWorkflowInstanceIdAndStepOrder(String workflowInstanceId, Integer stepOrder);
}

package com.citi.clm.exception;

public class WorkflowException extends RuntimeException {
    
    private String workflowInstanceId;
    private String stepName;
    
    public WorkflowException(String message) {
        super(message);
    }
    
    public WorkflowException(String message, Throwable cause) {
        super(message, cause);
    }
    
    public WorkflowException(String message, String workflowInstanceId) {
        super(message);
        this.workflowInstanceId = workflowInstanceId;
    }
    
    public WorkflowException(String message, String workflowInstanceId, String stepName) {
        super(message);
        this.workflowInstanceId = workflowInstanceId;
        this.stepName = stepName;
    }
    
    public String getWorkflowInstanceId() {
        return workflowInstanceId;
    }
    
    public String getStepName() {
        return stepName;
    }
}

package com.citi.clm.exception;

public class IOApiException extends RuntimeException {
    
    private int httpStatus;
    private String endpoint;
    private String responseBody;
    
    public IOApiException(String message) {
        super(message);
    }
    
    public IOApiException(String message, Throwable cause) {
        super(message, cause);
    }
    
    public IOApiException(String message, int httpStatus, String endpoint) {
        super(message);
        this.httpStatus = httpStatus;
        this.endpoint = endpoint;
    }
    
    public IOApiException(String message, int httpStatus, String endpoint, String responseBody) {
        super(message);
        this.httpStatus = httpStatus;
        this.endpoint = endpoint;
        this.responseBody = responseBody;
    }
    
    public int getHttpStatus() {
        return httpStatus;
    }
    
    public String getEndpoint() {
        return endpoint;
    }
    
    public String getResponseBody() {
        return responseBody;
    }
}

package com.citi.clm.exception;

public class CsiValidationException extends RuntimeException {
    
    private Integer csi;
    private String validationType;
    
    public CsiValidationException(String message) {
        super(message);
    }
    
    public CsiValidationException(String message, Integer csi) {
        super(message);
        this.csi = csi;
    }
    
    public CsiValidationException(String message, Integer csi, String validationType) {
        super(message);
        this.csi = csi;
        this.validationType = validationType;
    }
    
    public Integer getCsi() {
        return csi;
    }
    
    public String getValidationType() {
        return validationType;
    }
}

package com.citi.clm.service.io;

import com.citi.clm.config.IOApiConfig;
import com.citi.clm.dto.io.IOAuthResponse;
import com.citi.clm.exception.IOApiException;
import com.citi.clm.service.TransactionLogService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.RestTemplate;

import java.time.LocalDateTime;
import java.util.concurrent.locks.ReentrantLock;

@Slf4j
@Service
@RequiredArgsConstructor
public class IOAuthService {
    
    private final IOApiConfig ioApiConfig;
    private final RestTemplate restTemplate;
    private final TransactionLogService transactionLogService;
    
    private String cachedToken;
    private LocalDateTime tokenExpiryTime;
    private final ReentrantLock tokenLock = new ReentrantLock();
    
    /**
     * Get a valid IO API authentication token.
     * Returns cached token if still valid, otherwise fetches a new one.
     */
    public String getAuthToken() {
        tokenLock.lock();
        try {
            if (isTokenValid()) {
                log.debug("Using cached IO API token");
                return cachedToken;
            }
            
            log.info("Fetching new IO API authentication token");
            IOAuthResponse authResponse = fetchNewToken();
            
            cachedToken = authResponse.getAccessToken();
            int expiresInSeconds = Integer.parseInt(authResponse.getExpiresIn());
            tokenExpiryTime = LocalDateTime.now().plusSeconds(expiresInSeconds - 60); // Buffer of 60 seconds
            
            log.info("Successfully obtained IO API token, expires at: {}", tokenExpiryTime);
            return cachedToken;
            
        } finally {
            tokenLock.unlock();
        }
    }
    
    /**
     * Force refresh the token regardless of current validity.
     */
    public String refreshToken() {
        tokenLock.lock();
        try {
            cachedToken = null;
            tokenExpiryTime = null;
            return getAuthToken();
        } finally {
            tokenLock.unlock();
        }
    }
    
    private boolean isTokenValid() {
        return cachedToken != null && 
               tokenExpiryTime != null && 
               LocalDateTime.now().isBefore(tokenExpiryTime);
    }
    
    private IOAuthResponse fetchNewToken() {
        String url = ioApiConfig.getBaseUrl() + ioApiConfig.getAuthUrl();
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        headers.set("Authorization", "Basic " + ioApiConfig.getBasicAuthCredentials());
        
        MultiValueMap<String, String> body = new LinkedMultiValueMap<>();
        body.add("grant_type", ioApiConfig.getGrantType());
        body.add("scope", ioApiConfig.getScope());
        
        HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(body, headers);
        
        long startTime = System.currentTimeMillis();
        try {
            ResponseEntity<IOAuthResponse> response = restTemplate.exchange(
                url,
                HttpMethod.POST,
                request,
                IOAuthResponse.class
            );
            
            long duration = System.currentTimeMillis() - startTime;
            
            if (response.getStatusCode().is2xxSuccessful() && response.getBody() != null) {
                // Log successful auth
                transactionLogService.logIOApiCall(
                    url, "POST", body, response.getBody(), 
                    response.getStatusCode().value(), duration,
                    null, null, null, null
                );
                return response.getBody();
            }
            
            throw new IOApiException(
                "Failed to obtain IO API token: " + response.getStatusCode(),
                response.getStatusCode().value(),
                url
            );
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            transactionLogService.logIOApiCall(
                url, "POST", body, null, 
                500, duration,
                null, null, null, null
            );
            
            if (e instanceof IOApiException) {
                throw e;
            }
            throw new IOApiException("Error fetching IO API token: " + e.getMessage(), e);
        }
    }
    
    /**
     * Clear cached token (for testing or forced refresh scenarios).
     */
    public void clearCache() {
        tokenLock.lock();
        try {
            cachedToken = null;
            tokenExpiryTime = null;
        } finally {
            tokenLock.unlock();
        }
    }
}

package com.citi.clm.service.io;

import com.citi.clm.config.IOApiConfig;
import com.citi.clm.dto.io.*;
import com.citi.clm.entity.CertificateDetails;
import com.citi.clm.entity.WorkflowInstance;
import com.citi.clm.exception.IOApiException;
import com.citi.clm.service.TransactionLogService;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class IOExecutionService {
    
    private final IOApiConfig ioApiConfig;
    private final IOAuthService ioAuthService;
    private final RestTemplate restTemplate;
    private final TransactionLogService transactionLogService;
    private final ObjectMapper objectMapper;
    
    /**
     * Execute a playbook via IO API.
     * This is a fire-and-forget operation - playbook will callback to /result endpoint.
     */
    public IOExecuteResponse executePlaybook(
            WorkflowInstance workflowInstance,
            WorkflowInstance.WorkflowStepInstance step,
            CertificateDetails certificate,
            String pfyToken) {
        
        String url = ioApiConfig.getBaseUrl() + ioApiConfig.getExecuteUrl();
        String token = ioAuthService.getAuthToken();
        
        IOExecuteRequest request = buildExecuteRequest(workflowInstance, step, certificate, pfyToken);
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(token);
        headers.set("X-BYPASS-OVERRIDE", String.valueOf(ioApiConfig.isBypassOverride()));
        headers.set("X-EXECUTION-MODE", ioApiConfig.getExecutionMode());
        headers.set("X-VALIDATION-MODE", ioApiConfig.getValidationMode());
        
        HttpEntity<IOExecuteRequest> httpRequest = new HttpEntity<>(request, headers);
        
        long startTime = System.currentTimeMillis();
        try {
            ResponseEntity<IOExecuteResponse> response = restTemplate.exchange(
                url,
                HttpMethod.POST,
                httpRequest,
                IOExecuteResponse.class
            );
            
            long duration = System.currentTimeMillis() - startTime;
            
            if (response.getStatusCode().is2xxSuccessful() && response.getBody() != null) {
                IOExecuteResponse responseBody = response.getBody();
                
                log.info("Successfully triggered playbook execution. OrderId: {}, Status: {}", 
                    responseBody.getOrderId(), responseBody.getStatus());
                
                // Log the API call
                transactionLogService.logIOApiCall(
                    url, "POST", request, responseBody,
                    response.getStatusCode().value(), duration,
                    workflowInstance.getId(), certificate.getId(), 
                    workflowInstance.getCsi(), step.getStepName()
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
            transactionLogService.logIOApiCall(
                url, "POST", request, null,
                500, duration,
                workflowInstance.getId(), certificate.getId(),
                workflowInstance.getCsi(), step.getStepName()
            );
            
            if (e instanceof IOApiException) {
                throw e;
            }
            throw new IOApiException("Error executing playbook: " + e.getMessage(), e);
        }
    }
    
    /**
     * Check the status of an IO order.
     */
    public IOOrderStatusResponse getOrderStatus(String orderId) {
        String url = ioApiConfig.getBaseUrl() + ioApiConfig.getOrderStatusUrl() + "/" + orderId;
        String token = ioAuthService.getAuthToken();
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(token);
        
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
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(token);
        
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
     * This is called from UI when user wants to view detailed logs.
     */
    public IOPodLogResponse downloadPodLog(String podName) {
        String url = ioApiConfig.getBaseUrl() + ioApiConfig.getDownloadLogUrl() + 
                     "?podName=" + podName;
        String token = ioAuthService.getAuthToken();
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setBearerAuth(token);
        
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
    
    private IOExecuteRequest buildExecuteRequest(
            WorkflowInstance workflowInstance,
            WorkflowInstance.WorkflowStepInstance step,
            CertificateDetails certificate,
            String pfyToken) {
        
        // Build task parameters
        Map<String, String> parameters = new HashMap<>();
        parameters.put("Module", step.getIoAction());
        // TODO: Add certificate-specific parameters
        parameters.put("certificateId", certificate.getId());
        parameters.put("uniqueNumber", certificate.getUniqueNumber());
        parameters.put("targetHostname", certificate.getTargetHostname());
        parameters.put("productType", certificate.getProductType());
        
        IOExecuteRequest.TaskDefinition task = IOExecuteRequest.TaskDefinition.builder()
            .identifier("IOACTION")
            .programmaticName(ioApiConfig.getProgrammaticName())
            .action(step.getIoAction())
            .parameters(parameters)
            .build();
        
        return IOExecuteRequest.builder()
            .csiAppId(String.valueOf(workflowInstance.getCsi()))
            .environment(certificate.getEnvType() != null ? certificate.getEnvType() : "PROD") // TODO: Map environment properly
            .principal("") // TODO: Provide principal ID
            .target(certificate.getTargetHostname())
            .ticket("") // TODO: Generate or retrieve SNOW ticket
            .tasks(List.of(task))
            .cmtHostUrl(ioApiConfig.getCmtHostUrl())
            .inventoryHostUrl(ioApiConfig.getInventoryHostUrl())
            .transactionId(workflowInstance.getTransactionId())
            .tokenValue(pfyToken)
            .build();
    }
}

package com.citi.clm.service;

import com.citi.clm.entity.TransactionLogs;
import com.citi.clm.entity.TransactionLogs.IOApiCallDetails;
import com.citi.clm.enums.ExecutionEnvironment;
import com.citi.clm.repository.TransactionLogsRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class TransactionLogService {
    
    private final TransactionLogsRepository transactionLogsRepository;
    
    /**
     * Log an IO API call with all details.
     */
    public TransactionLogs logIOApiCall(
            String endpoint,
            String method,
            Object requestBody,
            Object responseBody,
            int httpStatus,
            long durationMs,
            String workflowInstanceId,
            String certificateId,
            Integer csi,
            String stepName) {
        
        IOApiCallDetails apiDetails = IOApiCallDetails.builder()
            .endpoint(endpoint)
            .method(method)
            .requestBody(requestBody)
            .responseBody(responseBody)
            .httpStatus(httpStatus)
            .durationMs(durationMs)
            .timestamp(new Date())
            .build();
        
        TransactionLogs log = TransactionLogs.builder()
            .transactionId(generateTransactionId())
            .date(new Date())
            .userId("SYSTEM")
            .apiaccessed("Yes")
            .arguments(method + " " + endpoint)
            .response(responseBody)
            .workflowInstanceId(workflowInstanceId)
            .certificateId(certificateId)
            .csi(csi)
            .stepName(stepName)
            .action("IO_API_CALL")
            .executionEnvironment(ExecutionEnvironment.IO)
            .status(httpStatus >= 200 && httpStatus < 300 ? "SUCCESS" : "ERROR")
            .ioApiCallDetails(apiDetails)
            .build();
        
        return transactionLogsRepository.save(log);
    }
    
    /**
     * Log a workflow action.
     */
    public TransactionLogs logWorkflowAction(
            String workflowInstanceId,
            String certificateId,
            Integer csi,
            String action,
            String stepName,
            ExecutionEnvironment environment,
            String orderId,
            String playbookName,
            String status,
            String errorMessage,
            String userId) {
        
        TransactionLogs log = TransactionLogs.builder()
            .transactionId(generateTransactionId())
            .date(new Date())
            .userId(userId != null ? userId : "SYSTEM")
            .apiaccessed("No")
            .arguments(action + " - " + (stepName != null ? stepName : "N/A"))
            .workflowInstanceId(workflowInstanceId)
            .certificateId(certificateId)
            .csi(csi)
            .stepName(stepName)
            .action(action)
            .executionEnvironment(environment)
            .orderId(orderId)
            .playbookName(playbookName)
            .status(status)
            .errorMessage(errorMessage)
            .build();
        
        return transactionLogsRepository.save(log);
    }
    
    /**
     * Log workflow initiation.
     */
    public void logWorkflowInitiation(
            String workflowInstanceId,
            String certificateId,
            Integer csi,
            String triggeredBy) {
        
        logWorkflowAction(
            workflowInstanceId,
            certificateId,
            csi,
            "WORKFLOW_INITIATED",
            null,
            null,
            null,
            null,
            "INITIATED",
            null,
            triggeredBy
        );
    }
    
    /**
     * Log workflow completion.
     */
    public void logWorkflowCompletion(
            String workflowInstanceId,
            String certificateId,
            Integer csi,
            boolean success,
            String errorMessage) {
        
        logWorkflowAction(
            workflowInstanceId,
            certificateId,
            csi,
            success ? "WORKFLOW_COMPLETED" : "WORKFLOW_FAILED",
            null,
            null,
            null,
            null,
            success ? "COMPLETED" : "FAILED",
            errorMessage,
            "SYSTEM"
        );
    }
    
    /**
     * Log step execution.
     */
    public void logStepExecution(
            String workflowInstanceId,
            String certificateId,
            Integer csi,
            String stepName,
            ExecutionEnvironment environment,
            String orderId,
            String playbookName,
            String status) {
        
        logWorkflowAction(
            workflowInstanceId,
            certificateId,
            csi,
            "STEP_EXECUTED",
            stepName,
            environment,
            orderId,
            playbookName,
            status,
            null,
            "SYSTEM"
        );
    }
    
    /**
     * Log step callback received.
     */
    public void logStepCallback(
            String workflowInstanceId,
            String certificateId,
            Integer csi,
            String stepName,
            String orderId,
            String status,
            String errorMessage) {
        
        logWorkflowAction(
            workflowInstanceId,
            certificateId,
            csi,
            "STEP_CALLBACK_RECEIVED",
            stepName,
            null,
            orderId,
            null,
            status,
            errorMessage,
            "SYSTEM"
        );
    }
    
    /**
     * Log CSI validation failure.
     */
    public void logCsiValidationFailure(
            String certificateId,
            Integer csi,
            String validationType,
            String reason) {
        
        logWorkflowAction(
            null,
            certificateId,
            csi,
            "CSI_VALIDATION_FAILED",
            null,
            null,
            null,
            null,
            "BLOCKED",
            validationType + ": " + reason,
            "SYSTEM"
        );
    }
    
    /**
     * Get logs for a workflow instance.
     */
    public List<TransactionLogs> getLogsForWorkflow(String workflowInstanceId) {
        return transactionLogsRepository.findByWorkflowInstanceId(workflowInstanceId);
    }
    
    /**
     * Get logs for a certificate.
     */
    public List<TransactionLogs> getLogsForCertificate(String certificateId) {
        return transactionLogsRepository.findByCertificateId(certificateId);
    }
    
    /**
     * Get logs for a CSI.
     */
    public List<TransactionLogs> getLogsForCsi(Integer csi) {
        return transactionLogsRepository.findByCsi(csi);
    }
    
    private Long generateTransactionId() {
        List<TransactionLogs> lastLogs = transactionLogsRepository.findTopByOrderByTransactionIdDesc();
        if (lastLogs.isEmpty()) {
            return 1L;
        }
        return lastLogs.get(0).getTransactionId() + 1;
    }
}

package com.citi.clm.service;

import com.citi.clm.config.WorkflowConfig;
import com.citi.clm.entity.CsiDetails;
import com.citi.clm.entity.ModuleConfiguration;
import com.citi.clm.entity.ModuleSubscription;
import com.citi.clm.enums.ExecutionEnvironment;
import com.citi.clm.enums.ModuleType;
import com.citi.clm.enums.PlaybookExecutionEnvironment;
import com.citi.clm.exception.CsiValidationException;
import com.citi.clm.repository.CsiDetailsRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.Optional;

@Slf4j
@Service
@RequiredArgsConstructor
public class CsiValidationService {
    
    private final CsiDetailsRepository csiDetailsRepository;
    private final WorkflowConfig workflowConfig;
    private final TransactionLogService transactionLogService;
    
    /**
     * Validate if auto-renewal is enabled for the given CSI.
     * Returns the CSI details if valid, throws exception if not.
     */
    public CsiDetails validateAutoRenewalEnabled(Integer csi, String certificateId) {
        CsiDetails csiDetails = getCsiDetails(csi);
        ModuleConfiguration config = getClmConfiguration(csiDetails);
        
        // Check if auto-renewal is enabled (default is true if config is null)
        boolean autoRenewalEnabled = config == null || config.isAutoRenewalEnabled();
        
        if (!autoRenewalEnabled) {
            String reason = "Auto-renewal is disabled for CSI " + csi;
            log.warn(reason);
            transactionLogService.logCsiValidationFailure(
                certificateId, csi, "AUTO_RENEWAL_CHECK", reason
            );
            throw new CsiValidationException(reason, csi, "AUTO_RENEWAL_CHECK");
        }
        
        log.debug("Auto-renewal validation passed for CSI {}", csi);
        return csiDetails;
    }
    
    /**
     * Validate if auto-deployment is enabled for the given CSI.
     * This is checked before deployment/push steps.
     */
    public boolean isAutoDeploymentEnabled(Integer csi, String certificateId) {
        CsiDetails csiDetails = getCsiDetails(csi);
        ModuleConfiguration config = getClmConfiguration(csiDetails);
        
        // Default is true if config is null
        boolean autoDeploymentEnabled = config == null || config.isAutoDeploymentEnabled();
        
        if (!autoDeploymentEnabled) {
            String reason = "Auto-deployment is disabled for CSI " + csi;
            log.info(reason);
            transactionLogService.logCsiValidationFailure(
                certificateId, csi, "AUTO_DEPLOYMENT_CHECK", reason
            );
        }
        
        return autoDeploymentEnabled;
    }
    
    /**
     * Get the preferred execution environment for the given CSI.
     * Returns IO as default if no preference is set.
     */
    public ExecutionEnvironment getPreferredExecutionEnvironment(Integer csi) {
        try {
            CsiDetails csiDetails = getCsiDetails(csi);
            ModuleConfiguration config = getClmConfiguration(csiDetails);
            
            if (config != null && config.getVmPreferredExecutionEnv() != null) {
                PlaybookExecutionEnvironment preference = config.getVmPreferredExecutionEnv();
                ExecutionEnvironment env = preference == PlaybookExecutionEnvironment.AAP 
                    ? ExecutionEnvironment.AAP 
                    : ExecutionEnvironment.IO;
                log.debug("CSI {} has preferred execution environment: {}", csi, env);
                return env;
            }
        } catch (Exception e) {
            log.warn("Could not get execution environment preference for CSI {}: {}", csi, e.getMessage());
        }
        
        // Default to IO
        log.debug("Using default execution environment IO for CSI {}", csi);
        return ExecutionEnvironment.IO;
    }
    
    /**
     * Get the custom expiry threshold for the given CSI.
     * Returns default (60 days) if no custom threshold is set.
     */
    public int getExpiryThresholdDays(Integer csi) {
        try {
            CsiDetails csiDetails = getCsiDetails(csi);
            ModuleConfiguration config = getClmConfiguration(csiDetails);
            
            if (config != null && config.getCustomExpiryThresholdDays() != null) {
                int threshold = config.getCustomExpiryThresholdDays();
                log.debug("CSI {} has custom expiry threshold: {} days", csi, threshold);
                return threshold;
            }
        } catch (Exception e) {
            log.warn("Could not get expiry threshold for CSI {}: {}", csi, e.getMessage());
        }
        
        // Return default
        return workflowConfig.getDefaultExpiryThresholdDays();
    }
    
    /**
     * Check if CLM_VM module is enabled for the given CSI.
     */
    public boolean isClmVmEnabled(Integer csi) {
        try {
            CsiDetails csiDetails = getCsiDetails(csi);
            return csiDetails.getModuleSubscriptions() != null &&
                   csiDetails.getModuleSubscriptions().stream()
                       .anyMatch(sub -> sub.getModule() == ModuleType.CLM_VM && sub.isSelected());
        } catch (Exception e) {
            log.warn("Could not check CLM_VM status for CSI {}: {}", csi, e.getMessage());
            return false;
        }
    }
    
    /**
     * Get CSI details or throw exception if not found.
     */
    public CsiDetails getCsiDetails(Integer csi) {
        return csiDetailsRepository.findByCSI(csi)
            .orElseThrow(() -> new CsiValidationException(
                "CSI not found: " + csi, csi, "CSI_NOT_FOUND"
            ));
    }
    
    /**
     * Get CLM module configuration from CSI details.
     * Returns null if no CLM_VM subscription or no configuration.
     */
    public ModuleConfiguration getClmConfiguration(CsiDetails csiDetails) {
        if (csiDetails.getModuleSubscriptions() == null) {
            return null;
        }
        
        Optional<ModuleSubscription> clmSubscription = csiDetails.getModuleSubscriptions().stream()
            .filter(sub -> sub.getModule() == ModuleType.CLM_VM && sub.isSelected())
            .findFirst();
        
        if (clmSubscription.isEmpty()) {
            return null;
        }
        
        return clmSubscription.get().getConfiguration();
    }
    
    /**
     * Get notification emails for a CSI.
     */
    public java.util.List<String> getNotificationEmails(Integer csi) {
        CsiDetails csiDetails = getCsiDetails(csi);
        
        java.util.List<String> emails = new java.util.ArrayList<>();
        
        // Add primary email
        if (csiDetails.getEmail() != null) {
            emails.add(csiDetails.getEmail());
        }
        
        // Add owner team DL
        if (csiDetails.getOwnerTeamEmailDlName() != null) {
            emails.add(csiDetails.getOwnerTeamEmailDlName());
        }
        
        // Add notification emails if configured
        if (csiDetails.getNotificationEmails() != null) {
            emails.addAll(csiDetails.getNotificationEmails());
        }
        
        return emails;
    }
}

package com.citi.clm.service.workflow;

import com.citi.clm.entity.WorkflowInstance;
import com.citi.clm.entity.WorkflowInstance.WorkflowStepInstance;
import com.citi.clm.enums.StepStatus;
import com.citi.clm.enums.WorkflowStatus;
import com.citi.clm.repository.WorkflowInstanceRepository;
import com.citi.clm.service.TransactionLogService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.Date;

@Slf4j
@Service
@RequiredArgsConstructor
public class WorkflowStateManager {
    
    private final WorkflowInstanceRepository workflowInstanceRepository;
    private final TransactionLogService transactionLogService;
    
    /**
     * Update workflow status and persist.
     */
    public WorkflowInstance updateWorkflowStatus(
            WorkflowInstance workflow, 
            WorkflowStatus newStatus,
            String reason) {
        
        WorkflowStatus oldStatus = workflow.getStatus();
        workflow.setStatus(newStatus);
        workflow.setLastUpdatedDate(new Date());
        
        if (newStatus == WorkflowStatus.IN_PROGRESS && workflow.getStartedDate() == null) {
            workflow.setStartedDate(new Date());
        }
        
        if (isTerminalStatus(newStatus)) {
            workflow.setCompletedDate(new Date());
        }
        
        if (newStatus == WorkflowStatus.FAILED) {
            workflow.setFailureReason(reason);
        }
        
        WorkflowInstance saved = workflowInstanceRepository.save(workflow);
        
        log.info("Workflow {} status changed: {} -> {} (reason: {})", 
            workflow.getId(), oldStatus, newStatus, reason);
        
        return saved;
    }
    
    /**
     * Update a specific step's status.
     */
    public WorkflowInstance updateStepStatus(
            WorkflowInstance workflow,
            int stepOrder,
            StepStatus newStatus,
            String orderId,
            String podName,
            String errorMessage) {
        
        WorkflowStepInstance step = workflow.getSteps().stream()
            .filter(s -> s.getStepOrder() == stepOrder)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "Step not found: " + stepOrder + " in workflow " + workflow.getId()
            ));
        
        StepStatus oldStatus = step.getStatus();
        step.setStatus(newStatus);
        
        if (orderId != null) {
            step.setOrderId(orderId);
        }
        
        if (podName != null) {
            step.setPodName(podName);
        }
        
        if (newStatus == StepStatus.EXECUTING && step.getStartedDate() == null) {
            step.setStartedDate(new Date());
        }
        
        if (isTerminalStepStatus(newStatus)) {
            step.setCompletedDate(new Date());
        }
        
        if (newStatus == StepStatus.FAILED || newStatus == StepStatus.RETRY_PENDING) {
            step.setErrorMessage(errorMessage);
            step.setRetryCount(step.getRetryCount() + 1);
        }
        
        workflow.setLastUpdatedDate(new Date());
        WorkflowInstance saved = workflowInstanceRepository.save(workflow);
        
        log.info("Workflow {} step {} status changed: {} -> {}", 
            workflow.getId(), step.getStepName(), oldStatus, newStatus);
        
        return saved;
    }
    
    /**
     * Move to next step in workflow.
     */
    public WorkflowInstance moveToNextStep(WorkflowInstance workflow) {
        int nextStepIndex = workflow.getCurrentStepIndex() + 1;
        
        if (nextStepIndex >= workflow.getSteps().size()) {
            // All steps completed
            return updateWorkflowStatus(workflow, WorkflowStatus.COMPLETED, "All steps completed successfully");
        }
        
        workflow.setCurrentStepIndex(nextStepIndex);
        workflow.setLastUpdatedDate(new Date());
        
        WorkflowInstance saved = workflowInstanceRepository.save(workflow);
        log.info("Workflow {} moved to step index {}", workflow.getId(), nextStepIndex);
        
        return saved;
    }
    
    /**
     * Pause workflow for manual intervention.
     */
    public WorkflowInstance pauseWorkflow(WorkflowInstance workflow, String reason) {
        return updateWorkflowStatus(workflow, WorkflowStatus.PAUSED, reason);
    }
    
    /**
     * Resume a paused workflow.
     */
    public WorkflowInstance resumeWorkflow(WorkflowInstance workflow) {
        if (workflow.getStatus() != WorkflowStatus.PAUSED) {
            throw new IllegalStateException(
                "Cannot resume workflow in status: " + workflow.getStatus()
            );
        }
        return updateWorkflowStatus(workflow, WorkflowStatus.IN_PROGRESS, "Workflow resumed");
    }
    
    /**
     * Mark workflow as failed.
     */
    public WorkflowInstance failWorkflow(WorkflowInstance workflow, String reason) {
        transactionLogService.logWorkflowCompletion(
            workflow.getId(),
            workflow.getCertificateId(),
            workflow.getCsi(),
            false,
            reason
        );
        return updateWorkflowStatus(workflow, WorkflowStatus.FAILED, reason);
    }
    
    /**
     * Mark workflow as completed.
     */
    public WorkflowInstance completeWorkflow(WorkflowInstance workflow) {
        transactionLogService.logWorkflowCompletion(
            workflow.getId(),
            workflow.getCertificateId(),
            workflow.getCsi(),
            true,
            null
        );
        return updateWorkflowStatus(workflow, WorkflowStatus.COMPLETED, "All steps completed");
    }
    
    /**
     * Trigger rollback for workflow.
     */
    public WorkflowInstance triggerRollback(WorkflowInstance workflow, String reason) {
        workflow.setRollbackTriggered(true);
        return updateWorkflowStatus(workflow, WorkflowStatus.ROLLBACK_IN_PROGRESS, reason);
    }
    
    /**
     * Get current step for execution.
     */
    public WorkflowStepInstance getCurrentStep(WorkflowInstance workflow) {
        if (workflow.getCurrentStepIndex() >= workflow.getSteps().size()) {
            return null;
        }
        return workflow.getSteps().get(workflow.getCurrentStepIndex());
    }
    
    /**
     * Check if all steps are completed.
     */
    public boolean areAllStepsCompleted(WorkflowInstance workflow) {
        return workflow.getSteps().stream()
            .allMatch(step -> step.getStatus() == StepStatus.COMPLETED || 
                             step.getStatus() == StepStatus.SKIPPED);
    }
    
    /**
     * Check if workflow has failed steps pending retry.
     */
    public boolean hasStepsPendingRetry(WorkflowInstance workflow) {
        return workflow.getSteps().stream()
            .anyMatch(step -> step.getStatus() == StepStatus.RETRY_PENDING);
    }
    
    private boolean isTerminalStatus(WorkflowStatus status) {
        return status == WorkflowStatus.COMPLETED ||
               status == WorkflowStatus.FAILED ||
               status == WorkflowStatus.CANCELLED ||
               status == WorkflowStatus.ROLLED_BACK;
    }
    
    private boolean isTerminalStepStatus(StepStatus status) {
        return status == StepStatus.COMPLETED ||
               status == StepStatus.FAILED ||
               status == StepStatus.SKIPPED ||
               status == StepStatus.RETRY_PENDING;
    }
}

package com.citi.clm.service.workflow;

import com.citi.clm.dto.io.IOExecuteResponse;
import com.citi.clm.entity.AnsibleResultRequest;
import com.citi.clm.entity.CertificateDetails;
import com.citi.clm.entity.WorkflowInstance;
import com.citi.clm.entity.WorkflowInstance.WorkflowStepInstance;
import com.citi.clm.enums.ExecutionEnvironment;
import com.citi.clm.enums.StepStatus;
import com.citi.clm.exception.IOApiException;
import com.citi.clm.exception.WorkflowException;
import com.citi.clm.repository.AnsibleResultRequestRepository;
import com.citi.clm.repository.CertificateDetailsRepository;
import com.citi.clm.service.CsiValidationService;
import com.citi.clm.service.TransactionLogService;
import com.citi.clm.service.io.IOExecutionService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.Date;

@Slf4j
@Service
@RequiredArgsConstructor
public class StepExecutor {
    
    private final IOExecutionService ioExecutionService;
    private final CsiValidationService csiValidationService;
    private final WorkflowStateManager workflowStateManager;
    private final CertificateDetailsRepository certificateDetailsRepository;
    private final AnsibleResultRequestRepository ansibleResultRequestRepository;
    private final TransactionLogService transactionLogService;
    
    /**
     * Execute the current step of the workflow.
     * This is a fire-and-forget operation - playbook will callback.
     */
    public void executeStep(WorkflowInstance workflow) {
        WorkflowStepInstance step = workflowStateManager.getCurrentStep(workflow);
        
        if (step == null) {
            log.warn("No current step to execute for workflow {}", workflow.getId());
            return;
        }
        
        log.info("Executing step {} ({}) for workflow {}", 
            step.getStepOrder(), step.getStepName(), workflow.getId());
        
        // Check if step requires deployment check
        if (step.isRequiresDeploymentCheck()) {
            boolean deploymentEnabled = csiValidationService.isAutoDeploymentEnabled(
                workflow.getCsi(), workflow.getCertificateId()
            );
            
            if (!deploymentEnabled) {
                log.info("Skipping step {} - auto-deployment disabled for CSI {}", 
                    step.getStepName(), workflow.getCsi());
                
                // Skip this step and move to next
                workflowStateManager.updateStepStatus(
                    workflow, step.getStepOrder(), StepStatus.SKIPPED,
                    null, null, "Auto-deployment disabled"
                );
                
                // Move to next step
                workflow = workflowStateManager.moveToNextStep(workflow);
                
                // Execute next step if workflow is still in progress
                if (workflow.getStatus() == com.citi.clm.enums.WorkflowStatus.IN_PROGRESS) {
                    executeStep(workflow);
                }
                return;
            }
        }
        
        // Get certificate details
        CertificateDetails certificate = certificateDetailsRepository
            .findById(workflow.getCertificateId())
            .orElseThrow(() -> new WorkflowException(
                "Certificate not found: " + workflow.getCertificateId(),
                workflow.getId()
            ));
        
        try {
            // Update step status to executing
            workflow = workflowStateManager.updateStepStatus(
                workflow, step.getStepOrder(), StepStatus.EXECUTING,
                null, null, null
            );
            
            // Execute based on execution environment
            if (workflow.getExecutionEnvironment() == ExecutionEnvironment.IO) {
                executeViaIO(workflow, step, certificate);
            } else {
                executeViaAAP(workflow, step, certificate);
            }
            
            // Log step execution
            transactionLogService.logStepExecution(
                workflow.getId(),
                certificate.getId(),
                workflow.getCsi(),
                step.getStepName(),
                workflow.getExecutionEnvironment(),
                step.getOrderId(),
                step.getPlaybookName(),
                "EXECUTING"
            );
            
        } catch (Exception e) {
            log.error("Error executing step {} for workflow {}: {}", 
                step.getStepName(), workflow.getId(), e.getMessage(), e);
            
            // Mark step as failed and pause workflow for manual retry
            workflowStateManager.updateStepStatus(
                workflow, step.getStepOrder(), StepStatus.RETRY_PENDING,
                null, null, e.getMessage()
            );
            
            workflowStateManager.pauseWorkflow(workflow, 
                "Step " + step.getStepName() + " failed: " + e.getMessage());
            
            throw new WorkflowException(
                "Step execution failed: " + e.getMessage(),
                workflow.getId(),
                step.getStepName()
            );
        }
    }
    
    /**
     * Retry a failed step.
     */
    public void retryStep(WorkflowInstance workflow, int stepOrder) {
        WorkflowStepInstance step = workflow.getSteps().stream()
            .filter(s -> s.getStepOrder() == stepOrder)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Step not found: " + stepOrder));
        
        if (step.getStatus() != StepStatus.RETRY_PENDING && step.getStatus() != StepStatus.FAILED) {
            throw new IllegalStateException(
                "Cannot retry step in status: " + step.getStatus()
            );
        }
        
        log.info("Retrying step {} for workflow {}", step.getStepName(), workflow.getId());
        
        // Reset step status
        workflowStateManager.updateStepStatus(
            workflow, stepOrder, StepStatus.PENDING,
            null, null, null
        );
        
        // Resume workflow if paused
        if (workflow.getStatus() == com.citi.clm.enums.WorkflowStatus.PAUSED) {
            workflow = workflowStateManager.resumeWorkflow(workflow);
        }
        
        // Set current step index to this step
        workflow.setCurrentStepIndex(stepOrder - 1); // stepOrder is 1-based
        
        // Execute the step
        executeStep(workflow);
    }
    
    private void executeViaIO(
            WorkflowInstance workflow,
            WorkflowStepInstance step,
            CertificateDetails certificate) {
        
        // TODO: Generate PFY token for callback authentication
        String pfyToken = generatePfyToken();
        
        try {
            IOExecuteResponse response = ioExecutionService.executePlaybook(
                workflow, step, certificate, pfyToken
            );
            
            // Update step with order ID
            workflowStateManager.updateStepStatus(
                workflow, step.getStepOrder(), StepStatus.EXECUTING,
                response.getOrderId(), null, null
            );
            
            // Create execution record
            createExecutionRecord(workflow, step, certificate, response.getOrderId(), null);
            
            log.info("IO playbook execution triggered. OrderId: {}", response.getOrderId());
            
        } catch (IOApiException e) {
            log.error("IO API error: {}", e.getMessage());
            throw e;
        }
    }
    
    private void executeViaAAP(
            WorkflowInstance workflow,
            WorkflowStepInstance step,
            CertificateDetails certificate) {
        
        // TODO: Implement Ansible Tower execution
        // This is the legacy path for CSIs with AAP preference
        
        log.info("Executing step {} via AAP (Ansible Tower)", step.getStepName());
        
        // TODO: Call existing Ansible Tower integration
        // For now, throwing exception as this needs to be implemented
        throw new WorkflowException(
            "AAP execution not yet implemented",
            workflow.getId(),
            step.getStepName()
        );
    }
    
    private void createExecutionRecord(
            WorkflowInstance workflow,
            WorkflowStepInstance step,
            CertificateDetails certificate,
            String orderId,
            String podName) {
        
        AnsibleResultRequest record = AnsibleResultRequest.builder()
            .servername(certificate.getTargetHostname())
            .transactionId(workflow.getTransactionId())
            .module(step.getIoAction())
            .executionStatus("EXECUTING")
            .createdDate(new Date())
            .executionEnvironment(workflow.getExecutionEnvironment())
            .orderId(orderId)
            .podName(podName)
            .workflowInstanceId(workflow.getId())
            .stepOrder(step.getStepOrder())
            .playbookName(step.getPlaybookName())
            .csi(workflow.getCsi())
            .certificateId(workflow.getCertificateId())
            .build();
        
        ansibleResultRequestRepository.save(record);
    }
    
    private String generatePfyToken() {
        // TODO: Implement PFY token generation for callback authentication
        return "TODO_GENERATE_PFY_TOKEN";
    }
}

package com.citi.clm.service.workflow;

import com.citi.clm.config.WorkflowConfig;
import com.citi.clm.entity.*;
import com.citi.clm.entity.WorkflowInstance.WorkflowStepInstance;
import com.citi.clm.enums.*;
import com.citi.clm.exception.CsiValidationException;
import com.citi.clm.exception.WorkflowException;
import com.citi.clm.repository.*;
import com.citi.clm.service.CsiValidationService;
import com.citi.clm.service.NotificationService;
import com.citi.clm.service.TransactionLogService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class WorkflowOrchestrator {
    
    private final WorkflowConfig workflowConfig;
    private final WorkflowDefinitionRepository workflowDefinitionRepository;
    private final WorkflowInstanceRepository workflowInstanceRepository;
    private final CertificateDetailsRepository certificateDetailsRepository;
    private final CsiValidationService csiValidationService;
    private final WorkflowStateManager workflowStateManager;
    private final StepExecutor stepExecutor;
    private final TransactionLogService transactionLogService;
    private final NotificationService notificationService;
    
    /**
     * Initiate renewal workflow for a certificate.
     * This is called by the scheduler or manual trigger.
     */
    public WorkflowInstance initiateRenewalWorkflow(String certificateId, String triggeredBy) {
        log.info("Initiating renewal workflow for certificate {} by {}", certificateId, triggeredBy);
        
        // Get certificate details
        CertificateDetails certificate = certificateDetailsRepository.findById(certificateId)
            .orElseThrow(() -> new WorkflowException("Certificate not found: " + certificateId));
        
        // Check if certificate already has an active workflow
        Optional<WorkflowInstance> existingWorkflow = workflowInstanceRepository
            .findByCertificateId(certificateId);
        if (existingWorkflow.isPresent() && 
            !isTerminalStatus(existingWorkflow.get().getStatus())) {
            throw new WorkflowException(
                "Certificate already has active workflow: " + existingWorkflow.get().getId(),
                existingWorkflow.get().getId()
            );
        }
        
        // Validate CSI for auto-renewal
        try {
            csiValidationService.validateAutoRenewalEnabled(certificate.getCsi(), certificateId);
        } catch (CsiValidationException e) {
            // Update certificate status
            certificate.setRenewalStatus("BLOCKED_AUTO_RENEWAL_DISABLED");
            certificateDetailsRepository.save(certificate);
            throw e;
        }
        
        // Determine workflow type based on product type
        WorkflowType workflowType = determineWorkflowType(certificate.getProductType());
        
        // Get execution environment preference
        ExecutionEnvironment execEnv = csiValidationService.getPreferredExecutionEnvironment(
            certificate.getCsi()
        );
        
        // Create workflow instance
        WorkflowInstance workflow = createWorkflowInstance(
            certificate, workflowType, execEnv, triggeredBy
        );
        
        // Update certificate with workflow ID
        certificate.setCurrentWorkflowId(workflow.getId());
        certificate.setRenewalStatus("IN_PROGRESS");
        certificate.setLastRenewalAttempt(new Date());
        certificateDetailsRepository.save(certificate);
        
        // Log workflow initiation
        transactionLogService.logWorkflowInitiation(
            workflow.getId(), certificateId, certificate.getCsi(), triggeredBy
        );
        
        // Send initiation notification
        notificationService.sendWorkflowInitiatedNotification(workflow, certificate);
        
        // Start workflow execution
        workflow = workflowStateManager.updateWorkflowStatus(
            workflow, WorkflowStatus.IN_PROGRESS, "Workflow started"
        );
        
        // Execute first step
        try {
            stepExecutor.executeStep(workflow);
        } catch (Exception e) {
            log.error("Error executing first step: {}", e.getMessage());
            // Workflow is already paused by StepExecutor on error
        }
        
        return workflow;
    }
    
    /**
     * Process callback from playbook result.
     * This continues the workflow to the next step.
     */
    public void processPlaybookCallback(
            String workflowInstanceId,
            int stepOrder,
            String executionStatus,
            Object result,
            String errorMessage) {
        
        log.info("Processing callback for workflow {} step {}: {}", 
            workflowInstanceId, stepOrder, executionStatus);
        
        WorkflowInstance workflow = workflowInstanceRepository.findById(workflowInstanceId)
            .orElseThrow(() -> new WorkflowException(
                "Workflow not found: " + workflowInstanceId
            ));
        
        WorkflowStepInstance step = workflow.getSteps().stream()
            .filter(s -> s.getStepOrder() == stepOrder)
            .findFirst()
            .orElseThrow(() -> new WorkflowException(
                "Step not found: " + stepOrder,
                workflowInstanceId
            ));
        
        // Update step with result
        step.setResult(result);
        step.setExecutionStatus(executionStatus);
        
        // Log callback
        transactionLogService.logStepCallback(
            workflowInstanceId,
            workflow.getCertificateId(),
            workflow.getCsi(),
            step.getStepName(),
            step.getOrderId(),
            executionStatus,
            errorMessage
        );
        
        if ("SUCCESS".equalsIgnoreCase(executionStatus) || 
            "COMPLETED".equalsIgnoreCase(executionStatus)) {
            
            // Mark step as completed
            workflow = workflowStateManager.updateStepStatus(
                workflow, stepOrder, StepStatus.COMPLETED,
                null, null, null
            );
            
            // Move to next step
            workflow = workflowStateManager.moveToNextStep(workflow);
            
            // If workflow is still in progress, execute next step
            if (workflow.getStatus() == WorkflowStatus.IN_PROGRESS) {
                try {
                    stepExecutor.executeStep(workflow);
                } catch (Exception e) {
                    log.error("Error executing next step: {}", e.getMessage());
                }
            } else if (workflow.getStatus() == WorkflowStatus.COMPLETED) {
                // Workflow completed successfully
                handleWorkflowCompletion(workflow);
            }
            
        } else if ("ERROR".equalsIgnoreCase(executionStatus) || 
                   "FAILED".equalsIgnoreCase(executionStatus)) {
            
            // Mark step as failed and pause for retry
            workflow = workflowStateManager.updateStepStatus(
                workflow, stepOrder, StepStatus.RETRY_PENDING,
                null, null, errorMessage
            );
            
            workflowStateManager.pauseWorkflow(workflow, 
                "Step " + step.getStepName() + " failed: " + errorMessage);
            
            // Send failure notification
            notificationService.sendStepFailureNotification(workflow, step, errorMessage);
        }
    }
    
    /**
     * Manually retry a failed workflow step.
     */
    public void retryWorkflowStep(String workflowInstanceId, int stepOrder, String userId) {
        log.info("Retrying workflow {} step {} by user {}", workflowInstanceId, stepOrder, userId);
        
        WorkflowInstance workflow = workflowInstanceRepository.findById(workflowInstanceId)
            .orElseThrow(() -> new WorkflowException("Workflow not found: " + workflowInstanceId));
        
        workflow.setLastUpdatedBy(userId);
        stepExecutor.retryStep(workflow, stepOrder);
    }
    
    /**
     * Trigger rollback for a workflow.
     */
    public void triggerRollback(String workflowInstanceId, String userId, String reason) {
        log.info("Triggering rollback for workflow {} by user {}: {}", 
            workflowInstanceId, userId, reason);
        
        WorkflowInstance workflow = workflowInstanceRepository.findById(workflowInstanceId)
            .orElseThrow(() -> new WorkflowException("Workflow not found: " + workflowInstanceId));
        
        if (workflow.getRollbackStep() == null) {
            throw new WorkflowException(
                "No rollback step defined for workflow",
                workflowInstanceId
            );
        }
        
        workflow = workflowStateManager.triggerRollback(workflow, reason);
        
        // Execute rollback step
        // TODO: Implement rollback execution
        log.info("Rollback triggered for workflow {}", workflowInstanceId);
    }
    
    /**
     * Cancel a workflow.
     */
    public void cancelWorkflow(String workflowInstanceId, String userId, String reason) {
        log.info("Cancelling workflow {} by user {}: {}", workflowInstanceId, userId, reason);
        
        WorkflowInstance workflow = workflowInstanceRepository.findById(workflowInstanceId)
            .orElseThrow(() -> new WorkflowException("Workflow not found: " + workflowInstanceId));
        
        workflowStateManager.updateWorkflowStatus(workflow, WorkflowStatus.CANCELLED, reason);
        
        // Update certificate status
        CertificateDetails certificate = certificateDetailsRepository
            .findById(workflow.getCertificateId())
            .orElse(null);
        if (certificate != null) {
            certificate.setRenewalStatus("CANCELLED");
            certificate.setCurrentWorkflowId(null);
            certificateDetailsRepository.save(certificate);
        }
    }
    
    /**
     * Get workflow status and details.
     */
    public WorkflowInstance getWorkflowStatus(String workflowInstanceId) {
        return workflowInstanceRepository.findById(workflowInstanceId)
            .orElseThrow(() -> new WorkflowException("Workflow not found: " + workflowInstanceId));
    }
    
    private WorkflowType determineWorkflowType(String productType) {
        if (productType == null) {
            return WorkflowType.OLD_WORKFLOW;
        }
        
        List<String> newWorkflowTypes = workflowConfig.getNewWorkflowProductTypes();
        if (newWorkflowTypes.contains(productType.toUpperCase())) {
            return WorkflowType.NEW_WORKFLOW;
        }
        
        return WorkflowType.OLD_WORKFLOW;
    }
    
    private WorkflowInstance createWorkflowInstance(
            CertificateDetails certificate,
            WorkflowType workflowType,
            ExecutionEnvironment execEnv,
            String triggeredBy) {
        
        // Get step definitions from config
        List<WorkflowStepInstance> steps = createStepInstances(workflowType);
        
        // Create rollback step
        WorkflowStepInstance rollbackStep = createRollbackStep();
        
        String transactionId = UUID.randomUUID().toString();
        
        WorkflowInstance workflow = WorkflowInstance.builder()
            .workflowType(workflowType)
            .certificateId(certificate.getId())
            .certificateUniqueNumber(certificate.getUniqueNumber())
            .csi(certificate.getCsi())
            .productType(certificate.getProductType())
            .status(WorkflowStatus.PENDING)
            .currentStepIndex(0)
            .executionEnvironment(execEnv)
            .steps(steps)
            .rollbackTriggered(false)
            .rollbackStep(rollbackStep)
            .createdDate(new Date())
            .lastUpdatedDate(new Date())
            .triggeredBy(triggeredBy)
            .retryCount(0)
            .transactionId(transactionId)
            .build();
        
        return workflowInstanceRepository.save(workflow);
    }
    
    private List<WorkflowStepInstance> createStepInstances(WorkflowType workflowType) {
        WorkflowConfig.PlaybookConfig playbookConfig = workflowType == WorkflowType.NEW_WORKFLOW
            ? workflowConfig.getNewWorkflow()
            : workflowConfig.getOldWorkflow();
        
        return playbookConfig.getSteps().entrySet().stream()
            .map(entry -> {
                WorkflowConfig.PlaybookConfig.StepConfig config = entry.getValue();
                return WorkflowStepInstance.builder()
                    .stepOrder(config.getOrder())
                    .stepName(entry.getKey())
                    .playbookName(config.getPlaybookName())
                    .ioAction(config.getIoAction())
                    .status(StepStatus.PENDING)
                    .requiresDeploymentCheck(config.isRequiresDeploymentCheck())
                    .retryCount(0)
                    .build();
            })
            .sorted(Comparator.comparingInt(WorkflowStepInstance::getStepOrder))
            .collect(Collectors.toList());
    }
    
    private WorkflowStepInstance createRollbackStep() {
        return WorkflowStepInstance.builder()
            .stepOrder(99)
            .stepName("rollback")
            .playbookName(workflowConfig.getRollbackPlaybook())
            .ioAction("ROLLBACK")
            .status(StepStatus.PENDING)
            .requiresDeploymentCheck(false)
            .retryCount(0)
            .build();
    }
    
    private void handleWorkflowCompletion(WorkflowInstance workflow) {
        log.info("Workflow {} completed successfully", workflow.getId());
        
        // Update certificate status
        CertificateDetails certificate = certificateDetailsRepository
            .findById(workflow.getCertificateId())
            .orElse(null);
        if (certificate != null) {
            certificate.setRenewalStatus("COMPLETED");
            certificate.setRenewalCompletedDate(new Date());
            certificate.setCurrentWorkflowId(null);
            certificateDetailsRepository.save(certificate);
        }
        
        // Send completion notification
        notificationService.sendWorkflowCompletedNotification(workflow, certificate);
    }
    
    private boolean isTerminalStatus(WorkflowStatus status) {
        return status == WorkflowStatus.COMPLETED ||
               status == WorkflowStatus.FAILED ||
               status == WorkflowStatus.CANCELLED ||
               status == WorkflowStatus.ROLLED_BACK;
    }
}

package com.citi.clm.service;

import com.citi.clm.config.WorkflowConfig;
import com.citi.clm.entity.CertificateDetails;
import com.citi.clm.entity.WorkflowInstance;
import com.citi.clm.entity.WorkflowInstance.WorkflowStepInstance;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class NotificationService {
    
    private final CsiValidationService csiValidationService;
    private final WorkflowConfig workflowConfig;
    // TODO: Inject IOExecutionService or NotificationPlaybookService
    
    /**
     * Send notification when workflow is initiated.
     */
    public void sendWorkflowInitiatedNotification(
            WorkflowInstance workflow,
            CertificateDetails certificate) {
        
        List<String> recipients = csiValidationService.getNotificationEmails(workflow.getCsi());
        
        String subject = String.format(
            "Certificate Renewal Initiated - %s",
            certificate.getUniqueNumber()
        );
        
        String htmlContent = buildInitiationEmailContent(workflow, certificate);
        
        sendNotification(recipients, subject, htmlContent, workflow, "INITIATION");
    }
    
    /**
     * Send notification when workflow completes successfully.
     */
    public void sendWorkflowCompletedNotification(
            WorkflowInstance workflow,
            CertificateDetails certificate) {
        
        List<String> recipients = csiValidationService.getNotificationEmails(workflow.getCsi());
        
        String subject = String.format(
            "Certificate Renewal Completed - %s",
            certificate.getUniqueNumber()
        );
        
        String htmlContent = buildCompletionEmailContent(workflow, certificate);
        
        sendNotification(recipients, subject, htmlContent, workflow, "COMPLETION");
    }
    
    /**
     * Send notification when a step fails.
     */
    public void sendStepFailureNotification(
            WorkflowInstance workflow,
            WorkflowStepInstance step,
            String errorMessage) {
        
        List<String> recipients = csiValidationService.getNotificationEmails(workflow.getCsi());
        
        String subject = String.format(
            "Certificate Renewal Step Failed - CSI %d - %s",
            workflow.getCsi(),
            step.getStepName()
        );
        
        String htmlContent = buildStepFailureEmailContent(workflow, step, errorMessage);
        
        sendNotification(recipients, subject, htmlContent, workflow, "STEP_FAILURE");
    }
    
    /**
     * Send notification when workflow fails.
     */
    public void sendWorkflowFailureNotification(
            WorkflowInstance workflow,
            CertificateDetails certificate,
            String reason) {
        
        List<String> recipients = csiValidationService.getNotificationEmails(workflow.getCsi());
        
        String subject = String.format(
            "Certificate Renewal Failed - %s",
            certificate.getUniqueNumber()
        );
        
        String htmlContent = buildFailureEmailContent(workflow, certificate, reason);
        
        sendNotification(recipients, subject, htmlContent, workflow, "FAILURE");
    }
    
    private void sendNotification(
            List<String> recipients,
            String subject,
            String htmlContent,
            WorkflowInstance workflow,
            String notificationType) {
        
        if (recipients.isEmpty()) {
            log.warn("No recipients for notification type {} for workflow {}", 
                notificationType, workflow.getId());
            return;
        }
        
        log.info("Sending {} notification for workflow {} to {} recipients",
            notificationType, workflow.getId(), recipients.size());
        
        // TODO: Execute notification playbook via IO API
        // The notification playbook accepts:
        // - recipients (list of emails)
        // - subject
        // - htmlContent
        
        // For now, just log
        log.info("Notification email: Subject={}, Recipients={}", subject, recipients);
        
        // TODO: Implement playbook call
        // IOExecuteRequest request = buildNotificationRequest(recipients, subject, htmlContent);
        // ioExecutionService.executePlaybook(workflow, notificationStep, certificate, pfyToken);
    }
    
    private String buildInitiationEmailContent(
            WorkflowInstance workflow,
            CertificateDetails certificate) {
        
        StringBuilder html = new StringBuilder();
        html.append("<html><body>");
        html.append("<h2>Certificate Renewal Initiated</h2>");
        html.append("<p>A certificate renewal workflow has been initiated.</p>");
        html.append("<table border='1' cellpadding='5'>");
        html.append("<tr><td><b>Certificate ID</b></td><td>").append(certificate.getUniqueNumber()).append("</td></tr>");
        html.append("<tr><td><b>CSI</b></td><td>").append(workflow.getCsi()).append("</td></tr>");
        html.append("<tr><td><b>Product Type</b></td><td>").append(certificate.getProductType()).append("</td></tr>");
        html.append("<tr><td><b>Target Hostname</b></td><td>").append(certificate.getTargetHostname()).append("</td></tr>");
        html.append("<tr><td><b>Workflow Type</b></td><td>").append(workflow.getWorkflowType()).append("</td></tr>");
        html.append("<tr><td><b>Triggered By</b></td><td>").append(workflow.getTriggeredBy()).append("</td></tr>");
        html.append("</table>");
        html.append("<p>You will receive another notification when the renewal completes.</p>");
        html.append("</body></html>");
        
        return html.toString();
    }
    
    private String buildCompletionEmailContent(
            WorkflowInstance workflow,
            CertificateDetails certificate) {
        
        StringBuilder html = new StringBuilder();
        html.append("<html><body>");
        html.append("<h2>Certificate Renewal Completed Successfully</h2>");
        html.append("<p>The certificate renewal workflow has completed successfully.</p>");
        html.append("<table border='1' cellpadding='5'>");
        html.append("<tr><td><b>Certificate ID</b></td><td>").append(certificate.getUniqueNumber()).append("</td></tr>");
        html.append("<tr><td><b>CSI</b></td><td>").append(workflow.getCsi()).append("</td></tr>");
        html.append("<tr><td><b>Product Type</b></td><td>").append(certificate.getProductType()).append("</td></tr>");
        html.append("<tr><td><b>Completed Steps</b></td><td>").append(workflow.getSteps().size()).append("</td></tr>");
        html.append("</table>");
        html.append("</body></html>");
        
        return html.toString();
    }
    
    private String buildStepFailureEmailContent(
            WorkflowInstance workflow,
            WorkflowStepInstance step,
            String errorMessage) {
        
        StringBuilder html = new StringBuilder();
        html.append("<html><body>");
        html.append("<h2>Certificate Renewal Step Failed</h2>");
        html.append("<p style='color:red;'>A step in the certificate renewal workflow has failed and requires attention.</p>");
        html.append("<table border='1' cellpadding='5'>");
        html.append("<tr><td><b>Workflow ID</b></td><td>").append(workflow.getId()).append("</td></tr>");
        html.append("<tr><td><b>CSI</b></td><td>").append(workflow.getCsi()).append("</td></tr>");
        html.append("<tr><td><b>Failed Step</b></td><td>").append(step.getStepName()).append("</td></tr>");
        html.append("<tr><td><b>Error Message</b></td><td>").append(errorMessage).append("</td></tr>");
        html.append("<tr><td><b>Retry Count</b></td><td>").append(step.getRetryCount()).append("</td></tr>");
        html.append("</table>");
        html.append("<p>Please review and retry the step from the CLM portal.</p>");
        html.append("</body></html>");
        
        return html.toString();
    }
    
    private String buildFailureEmailContent(
            WorkflowInstance workflow,
            CertificateDetails certificate,
            String reason) {
        
        StringBuilder html = new StringBuilder();
        html.append("<html><body>");
        html.append("<h2>Certificate Renewal Failed</h2>");
        html.append("<p style='color:red;'>The certificate renewal workflow has failed.</p>");
        html.append("<table border='1' cellpadding='5'>");
        html.append("<tr><td><b>Certificate ID</b></td><td>").append(certificate.getUniqueNumber()).append("</td></tr>");
        html.append("<tr><td><b>CSI</b></td><td>").append(workflow.getCsi()).append("</td></tr>");
        html.append("<tr><td><b>Failure Reason</b></td><td>").append(reason).append("</td></tr>");
        html.append("</table>");
        html.append("<p>Please contact the CLM support team for assistance.</p>");
        html.append("</body></html>");
        
        return html.toString();
    }
}

package com.citi.clm.service;

import com.citi.clm.config.SchedulerConfig;
import com.citi.clm.config.WorkflowConfig;
import com.citi.clm.entity.CertificateDetails;
import com.citi.clm.entity.CsiDetails;
import com.citi.clm.entity.WorkflowInstance;
import com.citi.clm.exception.CsiValidationException;
import com.citi.clm.exception.WorkflowException;
import com.citi.clm.repository.CertificateDetailsRepository;
import com.citi.clm.repository.WorkflowInstanceRepository;
import com.citi.clm.service.workflow.WorkflowOrchestrator;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class RenewalSchedulerService {
    
    private final SchedulerConfig schedulerConfig;
    private final WorkflowConfig workflowConfig;
    private final CertificateDetailsRepository certificateDetailsRepository;
    private final WorkflowInstanceRepository workflowInstanceRepository;
    private final CsiValidationService csiValidationService;
    private final WorkflowOrchestrator workflowOrchestrator;
    private final TransactionLogService transactionLogService;
    
    private final ExecutorService executorService = Executors.newFixedThreadPool(5);
    private final ConcurrentLinkedQueue<String> renewalQueue = new ConcurrentLinkedQueue<>();
    
    /**
     * Daily scheduled job to check for certificates expiring soon and queue them for renewal.
     */
    @Scheduled(cron = "${clm.scheduler.renewal-check-cron:0 0 2 * * ?}")
    public void checkAndQueueRenewals() {
        if (!schedulerConfig.isEnabled()) {
            log.info("Renewal scheduler is disabled");
            return;
        }
        
        log.info("Starting daily renewal check");
        
        try {
            // Get all certificates that need renewal
            List<CertificateDetails> certificatesForRenewal = findCertificatesForRenewal();
            
            log.info("Found {} certificates eligible for renewal", certificatesForRenewal.size());
            
            // Group by CSI to apply CSI-specific thresholds
            Map<Integer, List<CertificateDetails>> byCsi = certificatesForRenewal.stream()
                .collect(Collectors.groupingBy(CertificateDetails::getCsi));
            
            // Queue certificates for renewal
            int queued = 0;
            for (Map.Entry<Integer, List<CertificateDetails>> entry : byCsi.entrySet()) {
                Integer csi = entry.getKey();
                List<CertificateDetails> certs = entry.getValue();
                
                // Get CSI-specific threshold
                int threshold = csiValidationService.getExpiryThresholdDays(csi);
                
                // Filter certificates within threshold
                List<CertificateDetails> eligibleCerts = certs.stream()
                    .filter(cert -> isWithinThreshold(cert.getExpiry(), threshold))
                    .collect(Collectors.toList());
                
                for (CertificateDetails cert : eligibleCerts) {
                    renewalQueue.offer(cert.getId());
                    queued++;
                }
            }
            
            log.info("Queued {} certificates for renewal", queued);
            
            // Start processing queue
            processRenewalQueue();
            
        } catch (Exception e) {
            log.error("Error in renewal check: {}", e.getMessage(), e);
        }
    }
    
    /**
     * Process the renewal queue, respecting concurrency limits.
     */
    private void processRenewalQueue() {
        log.info("Processing renewal queue. Size: {}", renewalQueue.size());
        
        // Check current active workflows
        long activeWorkflows = workflowInstanceRepository.countActiveWorkflows();
        int maxConcurrent = schedulerConfig.getMaxConcurrentWorkflows();
        
        if (activeWorkflows >= maxConcurrent) {
            log.info("Max concurrent workflows reached ({}/{}). Waiting.", 
                activeWorkflows, maxConcurrent);
            return;
        }
        
        int canProcess = maxConcurrent - (int) activeWorkflows;
        
        List<Future<WorkflowInstance>> futures = new ArrayList<>();
        
        for (int i = 0; i < canProcess && !renewalQueue.isEmpty(); i++) {
            String certificateId = renewalQueue.poll();
            if (certificateId == null) break;
            
            Future<WorkflowInstance> future = executorService.submit(() -> 
                processRenewal(certificateId)
            );
            futures.add(future);
        }
        
        // Wait for submitted tasks
        for (Future<WorkflowInstance> future : futures) {
            try {
                future.get(5, TimeUnit.MINUTES);
            } catch (Exception e) {
                log.error("Error processing renewal: {}", e.getMessage());
            }
        }
        
        log.info("Processed batch. Remaining in queue: {}", renewalQueue.size());
    }
    
    /**
     * Process a single certificate renewal.
     */
    private WorkflowInstance processRenewal(String certificateId) {
        log.info("Processing renewal for certificate: {}", certificateId);
        
        try {
            return workflowOrchestrator.initiateRenewalWorkflow(certificateId, "SCHEDULER");
        } catch (CsiValidationException e) {
            log.warn("CSI validation failed for certificate {}: {}", 
                certificateId, e.getMessage());
            return null;
        } catch (WorkflowException e) {
            log.error("Workflow error for certificate {}: {}", 
                certificateId, e.getMessage());
            return null;
        } catch (Exception e) {
            log.error("Unexpected error processing certificate {}: {}", 
                certificateId, e.getMessage(), e);
            return null;
        }
    }
    
    /**
     * Find all certificates that are eligible for renewal.
     */
    private List<CertificateDetails> findCertificatesForRenewal() {
        // Get the maximum possible expiry date (default threshold + buffer)
        int maxThreshold = workflowConfig.getDefaultExpiryThresholdDays() + 30;
        LocalDate maxExpiryDate = LocalDate.now().plusDays(maxThreshold);
        String expiryDateStr = maxExpiryDate.format(DateTimeFormatter.ISO_LOCAL_DATE);
        
        return certificateDetailsRepository.findCertificatesForRenewal(expiryDateStr);
    }
    
    /**
     * Check if certificate expiry is within the threshold days.
     */
    private boolean isWithinThreshold(String expiryDateStr, int thresholdDays) {
        if (expiryDateStr == null || expiryDateStr.isEmpty()) {
            return false;
        }
        
        try {
            LocalDate expiryDate = LocalDate.parse(expiryDateStr);
            LocalDate thresholdDate = LocalDate.now().plusDays(thresholdDays);
            
            return expiryDate.isBefore(thresholdDate) || expiryDate.isEqual(thresholdDate);
        } catch (Exception e) {
            log.warn("Could not parse expiry date: {}", expiryDateStr);
            return false;
        }
    }
    
    /**
     * Manually trigger renewal for a specific certificate.
     */
    public WorkflowInstance triggerManualRenewal(String certificateId, String userId) {
        log.info("Manual renewal triggered for certificate {} by {}", certificateId, userId);
        return workflowOrchestrator.initiateRenewalWorkflow(certificateId, userId);
    }
    
    /**
     * Get current queue size.
     */
    public int getQueueSize() {
        return renewalQueue.size();
    }
    
    /**
     * Clear the renewal queue.
     */
    public void clearQueue() {
        renewalQueue.clear();
        log.info("Renewal queue cleared");
    }
    
    /**
     * Add certificate to renewal queue manually.
     */
    public void addToQueue(String certificateId) {
        renewalQueue.offer(certificateId);
        log.info("Certificate {} added to renewal queue", certificateId);
    }
}

package com.citi.clm.service;

import com.citi.clm.dto.ResultCallbackRequest;
import com.citi.clm.entity.AnsibleResultRequest;
import com.citi.clm.entity.WorkflowInstance;
import com.citi.clm.repository.AnsibleResultRequestRepository;
import com.citi.clm.repository.WorkflowInstanceRepository;
import com.citi.clm.service.workflow.WorkflowOrchestrator;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class ResultCallbackService {
    
    private final WorkflowInstanceRepository workflowInstanceRepository;
    private final AnsibleResultRequestRepository ansibleResultRequestRepository;
    private final WorkflowOrchestrator workflowOrchestrator;
    private final TransactionLogService transactionLogService;
    
    /**
     * Process the callback result from a playbook execution.
     * This is called when playbook completes (success or failure).
     */
    public void processCallback(ResultCallbackRequest request) {
        log.info("Processing callback for transaction: {}, status: {}", 
            request.getTransactionId(), request.getExecutionStatus());
        
        // Validate request
        if (request.getWorkflowInstanceId() == null && request.getTransactionId() == null) {
            log.error("Invalid callback - no workflow or transaction ID");
            throw new IllegalArgumentException("Missing workflow or transaction ID");
        }
        
        // Find workflow instance
        WorkflowInstance workflow = findWorkflow(request);
        if (workflow == null) {
            log.error("Workflow not found for callback: {}", request);
            throw new IllegalArgumentException("Workflow not found");
        }
        
        // Determine step order
        int stepOrder = request.getStepOrder() != null 
            ? request.getStepOrder() 
            : workflow.getCurrentStepIndex() + 1; // Convert to 1-based
        
        // Save result to database
        saveResultRecord(request, workflow, stepOrder);
        
        // Process the callback in workflow orchestrator
        workflowOrchestrator.processPlaybookCallback(
            workflow.getId(),
            stepOrder,
            request.getExecutionStatus(),
            request.getResult(),
            extractErrorMessage(request)
        );
        
        log.info("Callback processed successfully for workflow {} step {}", 
            workflow.getId(), stepOrder);
    }
    
    /**
     * Process callback with idempotency check.
     * This prevents duplicate processing of the same callback.
     */
    public boolean processCallbackIdempotent(ResultCallbackRequest request) {
        // Check if this callback was already processed
        String idempotencyKey = generateIdempotencyKey(request);
        
        // Check for existing record with same order ID and step
        if (request.getOrderId() != null && request.getStepOrder() != null) {
            var existing = ansibleResultRequestRepository
                .findByWorkflowInstanceIdAndStepOrder(
                    request.getWorkflowInstanceId(), 
                    request.getStepOrder()
                );
            
            if (existing.isPresent() && 
                "COMPLETED".equals(existing.get().getExecutionStatus())) {
                log.info("Callback already processed for workflow {} step {}", 
                    request.getWorkflowInstanceId(), request.getStepOrder());
                return false; // Already processed
            }
        }
        
        processCallback(request);
        return true;
    }
    
    private WorkflowInstance findWorkflow(ResultCallbackRequest request) {
        // First try by workflow instance ID
        if (request.getWorkflowInstanceId() != null) {
            return workflowInstanceRepository.findById(request.getWorkflowInstanceId())
                .orElse(null);
        }
        
        // Then try by transaction ID
        if (request.getTransactionId() != null) {
            return workflowInstanceRepository.findByTransactionId(request.getTransactionId())
                .orElse(null);
        }
        
        return null;
    }
    
    private void saveResultRecord(
            ResultCallbackRequest request,
            WorkflowInstance workflow,
            int stepOrder) {
        
        // Find existing record or create new
        AnsibleResultRequest record = ansibleResultRequestRepository
            .findByWorkflowInstanceIdAndStepOrder(workflow.getId(), stepOrder)
            .orElse(AnsibleResultRequest.builder()
                .workflowInstanceId(workflow.getId())
                .stepOrder(stepOrder)
                .csi(workflow.getCsi())
                .certificateId(workflow.getCertificateId())
                .createdDate(new Date())
                .build());
        
        // Update with callback data
        record.setServername(request.getServername());
        record.setTransactionId(request.getTransactionId());
        record.setResult(request.getResult());
        record.setModule(request.getModule());
        record.setConnectionStatus(request.getConnectionStatus());
        record.setExecutionStatus(request.getExecutionStatus());
        record.setExecutionLog(request.getExecutionLog());
        record.setAnsibleRefId(request.getAnsibleRefId());
        record.setUserId(request.getUserId());
        record.setUpdatedDate(new Date());
        
        if (request.getOrderId() != null) {
            record.setOrderId(request.getOrderId());
        }
        if (request.getPodName() != null) {
            record.setPodName(request.getPodName());
        }
        
        // Convert chronology
        if (request.getChronology() != null) {
            record.setChronology(request.getChronology().stream()
                .map(c -> AnsibleResultRequest.AnsibleChronology.builder()
                    .action(c.getAction())
                    .status(c.getStatus())
                    .details(c.getDetails())
                    .build())
                .collect(Collectors.toList()));
        }
        
        ansibleResultRequestRepository.save(record);
    }
    
    private String extractErrorMessage(ResultCallbackRequest request) {
        if ("ERROR".equalsIgnoreCase(request.getExecutionStatus()) ||
            "FAILED".equalsIgnoreCase(request.getExecutionStatus())) {
            
            // Try to extract error from result
            if (request.getResult() != null) {
                return request.getResult().toString();
            }
            
            // Use execution log
            if (request.getExecutionLog() != null) {
                return request.getExecutionLog();
            }
            
            return "Execution failed - no details provided";
        }
        
        return null;
    }
    
    private String generateIdempotencyKey(ResultCallbackRequest request) {
        return String.format("%s-%s-%s",
            request.getWorkflowInstanceId(),
            request.getStepOrder(),
            request.getOrderId()
        );
    }
}

package com.citi.clm.controller;

import com.citi.clm.dto.ResultCallbackRequest;
import com.citi.clm.service.ResultCallbackService;
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
    
    /**
     * Endpoint called by playbooks to report execution results.
     * This is the /result API that playbooks call on completion.
     */
    @PostMapping("/result")
    @Operation(summary = "Process playbook result callback", 
               description = "Called by playbooks to report execution status and results")
    public ResponseEntity<Map<String, Object>> processResult(
            @RequestBody ResultCallbackRequest request) {
        
        log.info("Received result callback: transactionId={}, status={}", 
            request.getTransactionId(), request.getExecutionStatus());
        
        try {
            boolean processed = resultCallbackService.processCallbackIdempotent(request);
            
            if (processed) {
                return ResponseEntity.ok(Map.of(
                    "status", "SUCCESS",
                    "message", "Callback processed successfully"
                ));
            } else {
                return ResponseEntity.ok(Map.of(
                    "status", "SKIPPED",
                    "message", "Callback already processed"
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
}

package com.citi.clm.controller;

import com.citi.clm.dto.WorkflowTriggerRequest;
import com.citi.clm.entity.WorkflowInstance;
import com.citi.clm.enums.WorkflowStatus;
import com.citi.clm.exception.CsiValidationException;
import com.citi.clm.exception.WorkflowException;
import com.citi.clm.repository.WorkflowInstanceRepository;
import com.citi.clm.service.RenewalSchedulerService;
import com.citi.clm.service.TransactionLogService;
import com.citi.clm.service.workflow.WorkflowOrchestrator;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/v1/workflow")
@RequiredArgsConstructor
@Tag(name = "Workflow Admin", description = "Endpoints for workflow management")
public class WorkflowAdminController {
    
    private final WorkflowOrchestrator workflowOrchestrator;
    private final RenewalSchedulerService renewalSchedulerService;
    private final WorkflowInstanceRepository workflowInstanceRepository;
    private final TransactionLogService transactionLogService;
    
    /**
     * Manually trigger renewal workflow for a certificate.
     */
    @PostMapping("/trigger")
    @Operation(summary = "Trigger renewal workflow", 
               description = "Manually initiate renewal workflow for a certificate")
    public ResponseEntity<?> triggerWorkflow(
            @RequestBody WorkflowTriggerRequest request,
            @RequestHeader(value = "X-User-Id", required = false) String userId) {
        
        log.info("Manual workflow trigger: certificate={}, user={}", 
            request.getCertificateId(), userId);
        
        try {
            String triggeredBy = userId != null ? userId : request.getTriggeredBy();
            WorkflowInstance workflow = renewalSchedulerService.triggerManualRenewal(
                request.getCertificateId(), triggeredBy
            );
            
            return ResponseEntity.ok(Map.of(
                "status", "SUCCESS",
                "workflowId", workflow.getId(),
                "message", "Workflow initiated successfully"
            ));
            
        } catch (CsiValidationException e) {
            return ResponseEntity.badRequest().body(Map.of(
                "status", "BLOCKED",
                "reason", e.getMessage(),
                "validationType", e.getValidationType()
            ));
        } catch (WorkflowException e) {
            return ResponseEntity.badRequest().body(Map.of(
                "status", "ERROR",
                "message", e.getMessage()
            ));
        }
    }
    
    /**
     * Get workflow status and details.
     */
    @GetMapping("/{workflowId}")
    @Operation(summary = "Get workflow status", 
               description = "Retrieve workflow details and current status")
    public ResponseEntity<?> getWorkflowStatus(@PathVariable String workflowId) {
        try {
            WorkflowInstance workflow = workflowOrchestrator.getWorkflowStatus(workflowId);
            return ResponseEntity.ok(workflow);
        } catch (WorkflowException e) {
            return ResponseEntity.notFound().build();
        }
    }
    
    /**
     * Get all active workflows.
     */
    @GetMapping("/active")
    @Operation(summary = "Get active workflows", 
               description = "List all currently active workflows")
    public ResponseEntity<List<WorkflowInstance>> getActiveWorkflows() {
        List<WorkflowInstance> workflows = workflowInstanceRepository.findActiveWorkflows();
        return ResponseEntity.ok(workflows);
    }
    
    /**
     * Get workflows by status.
     */
    @GetMapping("/status/{status}")
    @Operation(summary = "Get workflows by status", 
               description = "List workflows with specific status")
    public ResponseEntity<List<WorkflowInstance>> getWorkflowsByStatus(
            @PathVariable WorkflowStatus status) {
        List<WorkflowInstance> workflows = workflowInstanceRepository.findByStatus(status);
        return ResponseEntity.ok(workflows);
    }
    
    /**
     * Get workflows for a CSI.
     */
    @GetMapping("/csi/{csi}")
    @Operation(summary = "Get workflows for CSI", 
               description = "List all workflows for a specific CSI")
    public ResponseEntity<List<WorkflowInstance>> getWorkflowsByCsi(@PathVariable Integer csi) {
        List<WorkflowInstance> workflows = workflowInstanceRepository.findByCsi(csi);
        return ResponseEntity.ok(workflows);
    }
    
    /**
     * Retry a failed workflow step.
     */
    @PostMapping("/{workflowId}/step/{stepOrder}/retry")
    @Operation(summary = "Retry failed step", 
               description = "Retry a failed workflow step")
    public ResponseEntity<?> retryStep(
            @PathVariable String workflowId,
            @PathVariable int stepOrder,
            @RequestHeader(value = "X-User-Id", required = false) String userId) {
        
        log.info("Retry step {} for workflow {} by user {}", stepOrder, workflowId, userId);
        
        try {
            workflowOrchestrator.retryWorkflowStep(workflowId, stepOrder, userId);
            
            return ResponseEntity.ok(Map.of(
                "status", "SUCCESS",
                "message", "Step retry initiated"
            ));
            
        } catch (Exception e) {
            return ResponseEntity.badRequest().body(Map.of(
                "status", "ERROR",
                "message", e.getMessage()
            ));
        }
    }
    
    /**
     * Trigger rollback for a workflow.
     */
    @PostMapping("/{workflowId}/rollback")
    @Operation(summary = "Trigger rollback", 
               description = "Trigger rollback for a workflow")
    public ResponseEntity<?> triggerRollback(
            @PathVariable String workflowId,
            @RequestBody Map<String, String> request,
            @RequestHeader(value = "X-User-Id", required = false) String userId) {
        
        String reason = request.getOrDefault("reason", "Manual rollback triggered");
        
        log.info("Rollback triggered for workflow {} by user {}: {}", workflowId, userId, reason);
        
        try {
            workflowOrchestrator.triggerRollback(workflowId, userId, reason);
            
            return ResponseEntity.ok(Map.of(
                "status", "SUCCESS",
                "message", "Rollback initiated"
            ));
            
        } catch (Exception e) {
            return ResponseEntity.badRequest().body(Map.of(
                "status", "ERROR",
                "message", e.getMessage()
            ));
        }
    }
    
    /**
     * Cancel a workflow.
     */
    @PostMapping("/{workflowId}/cancel")
    @Operation(summary = "Cancel workflow", 
               description = "Cancel an active workflow")
    public ResponseEntity<?> cancelWorkflow(
            @PathVariable String workflowId,
            @RequestBody Map<String, String> request,
            @RequestHeader(value = "X-User-Id", required = false) String userId) {
        
        String reason = request.getOrDefault("reason", "Cancelled by user");
        
        log.info("Cancel workflow {} by user {}: {}", workflowId, userId, reason);
        
        try {
            workflowOrchestrator.cancelWorkflow(workflowId, userId, reason);
            
            return ResponseEntity.ok(Map.of(
                "status", "SUCCESS",
                "message", "Workflow cancelled"
            ));
            
        } catch (Exception e) {
            return ResponseEntity.badRequest().body(Map.of(
                "status", "ERROR",
                "message", e.getMessage()
            ));
        }
    }
    
    /**
     * Get transaction logs for a workflow.
     */
    @GetMapping("/{workflowId}/logs")
    @Operation(summary = "Get workflow logs", 
               description = "Get all transaction logs for a workflow")
    public ResponseEntity<?> getWorkflowLogs(@PathVariable String workflowId) {
        return ResponseEntity.ok(transactionLogService.getLogsForWorkflow(workflowId));
    }
    
    /**
     * Get renewal queue status.
     */
    @GetMapping("/queue/status")
    @Operation(summary = "Get queue status", 
               description = "Get current renewal queue size and status")
    public ResponseEntity<?> getQueueStatus() {
        return ResponseEntity.ok(Map.of(
            "queueSize", renewalSchedulerService.getQueueSize(),
            "activeWorkflows", workflowInstanceRepository.countActiveWorkflows()
        ));
    }
}

package com.citi.clm.controller;

import com.citi.clm.dto.io.IOOrderStatusResponse;
import com.citi.clm.dto.io.IOPodListResponse;
import com.citi.clm.dto.io.IOPodLogResponse;
import com.citi.clm.entity.AnsibleResultRequest;
import com.citi.clm.entity.TransactionLogs;
import com.citi.clm.repository.AnsibleResultRequestRepository;
import com.citi.clm.service.TransactionLogService;
import com.citi.clm.service.io.IOExecutionService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Slf4j
@RestController
@RequestMapping("/api/v1/logs")
@RequiredArgsConstructor
@Tag(name = "Execution Logs", description = "Endpoints for viewing execution logs")
public class ExecutionLogController {
    
    private final IOExecutionService ioExecutionService;
    private final AnsibleResultRequestRepository ansibleResultRequestRepository;
    private final TransactionLogService transactionLogService;
    
    /**
     * Get order status from IO API.
     * Called from UI to check current status of an execution.
     */
    @GetMapping("/io/order/{orderId}/status")
    @Operation(summary = "Get IO order status", 
               description = "Get current status of an IO API order")
    public ResponseEntity<IOOrderStatusResponse> getOrderStatus(@PathVariable String orderId) {
        log.info("Getting order status for: {}", orderId);
        
        IOOrderStatusResponse status = ioExecutionService.getOrderStatus(orderId);
        return ResponseEntity.ok(status);
    }
    
    /**
     * List pods for a completed IO order.
     * Called from UI to see all execution pods.
     */
    @GetMapping("/io/order/{orderId}/pods")
    @Operation(summary = "List IO order pods", 
               description = "List all execution pods for an IO order")
    public ResponseEntity<IOPodListResponse> listPods(@PathVariable String orderId) {
        log.info("Listing pods for order: {}", orderId);
        
        IOPodListResponse pods = ioExecutionService.listPods(orderId);
        return ResponseEntity.ok(pods);
    }
    
    /**
     * Download detailed pod log from IO API.
     * Called from UI to view detailed execution logs.
     */
    @GetMapping("/io/pod/{podName}/log")
    @Operation(summary = "Download pod log", 
               description = "Download detailed execution log for a pod")
    public ResponseEntity<IOPodLogResponse> downloadPodLog(@PathVariable String podName) {
        log.info("Downloading pod log: {}", podName);
        
        IOPodLogResponse logResponse = ioExecutionService.downloadPodLog(podName);
        return ResponseEntity.ok(logResponse);
    }
    
    /**
     * Get execution records for a workflow.
     */
    @GetMapping("/workflow/{workflowId}")
    @Operation(summary = "Get workflow execution records", 
               description = "Get all execution records for a workflow")
    public ResponseEntity<List<AnsibleResultRequest>> getWorkflowExecutionRecords(
            @PathVariable String workflowId) {
        
        List<AnsibleResultRequest> records = ansibleResultRequestRepository
            .findByWorkflowInstanceId(workflowId);
        return ResponseEntity.ok(records);
    }
    
    /**
     * Get execution records for a certificate.
     */
    @GetMapping("/certificate/{certificateId}")
    @Operation(summary = "Get certificate execution records", 
               description = "Get all execution records for a certificate")
    public ResponseEntity<List<AnsibleResultRequest>> getCertificateExecutionRecords(
            @PathVariable String certificateId) {
        
        List<AnsibleResultRequest> records = ansibleResultRequestRepository
            .findByCertificateId(certificateId);
        return ResponseEntity.ok(records);
    }
    
    /**
     * Get execution records for a CSI.
     */
    @GetMapping("/csi/{csi}")
    @Operation(summary = "Get CSI execution records", 
               description = "Get all execution records for a CSI")
    public ResponseEntity<List<AnsibleResultRequest>> getCsiExecutionRecords(
            @PathVariable Integer csi) {
        
        List<AnsibleResultRequest> records = ansibleResultRequestRepository.findByCsi(csi);
        return ResponseEntity.ok(records);
    }
    
    /**
     * Get transaction logs for a certificate.
     */
    @GetMapping("/transactions/certificate/{certificateId}")
    @Operation(summary = "Get certificate transaction logs", 
               description = "Get all transaction logs for a certificate")
    public ResponseEntity<List<TransactionLogs>> getCertificateTransactionLogs(
            @PathVariable String certificateId) {
        
        List<TransactionLogs> logs = transactionLogService.getLogsForCertificate(certificateId);
        return ResponseEntity.ok(logs);
    }
    
    /**
     * Get transaction logs for a CSI.
     */
    @GetMapping("/transactions/csi/{csi}")
    @Operation(summary = "Get CSI transaction logs", 
               description = "Get all transaction logs for a CSI")
    public ResponseEntity<List<TransactionLogs>> getCsiTransactionLogs(
            @PathVariable Integer csi) {
        
        List<TransactionLogs> logs = transactionLogService.getLogsForCsi(csi);
        return ResponseEntity.ok(logs);
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

  # Scheduler Configuration
  scheduler:
    enabled: ${SCHEDULER_ENABLED:true}
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

      package com.citi.clm.config;

import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

import java.time.Duration;

@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder, IOApiConfig ioApiConfig) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(ioApiConfig.getConnectTimeoutSeconds()))
            .setReadTimeout(Duration.ofSeconds(ioApiConfig.getReadTimeoutSeconds()))
            .build();
    }
}

package com.citi.clm.config;

import com.citi.clm.entity.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.data.domain.Sort;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.index.Index;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class MongoIndexConfig {
    
    private final MongoTemplate mongoTemplate;
    
    @EventListener(ApplicationReadyEvent.class)
    public void initIndexes() {
        log.info("Creating MongoDB indexes...");
        
        // CsiDetails indexes
        mongoTemplate.indexOps(CsiDetails.class)
            .ensureIndex(new Index().on("CSI", Sort.Direction.ASC).unique());
        
        // CertificateDetails indexes
        mongoTemplate.indexOps(CertificateDetails.class)
            .ensureIndex(new Index().on("uniqueNumber", Sort.Direction.ASC).unique());
        mongoTemplate.indexOps(CertificateDetails.class)
            .ensureIndex(new Index().on("csi", Sort.Direction.ASC));
        mongoTemplate.indexOps(CertificateDetails.class)
            .ensureIndex(new Index().on("expiry", Sort.Direction.ASC));
        mongoTemplate.indexOps(CertificateDetails.class)
            .ensureIndex(new Index().on("renewalStatus", Sort.Direction.ASC));
        mongoTemplate.indexOps(CertificateDetails.class)
            .ensureIndex(new Index().on("productType", Sort.Direction.ASC));
        
        // WorkflowInstance indexes
        mongoTemplate.indexOps(WorkflowInstance.class)
            .ensureIndex(new Index().on("certificateId", Sort.Direction.ASC));
        mongoTemplate.indexOps(WorkflowInstance.class)
            .ensureIndex(new Index().on("csi", Sort.Direction.ASC));
        mongoTemplate.indexOps(WorkflowInstance.class)
            .ensureIndex(new Index().on("status", Sort.Direction.ASC));
        mongoTemplate.indexOps(WorkflowInstance.class)
            .ensureIndex(new Index().on("transactionId", Sort.Direction.ASC).unique());
        mongoTemplate.indexOps(WorkflowInstance.class)
            .ensureIndex(new Index().on("createdDate", Sort.Direction.DESC));
        
        // AnsibleResultRequest indexes
        mongoTemplate.indexOps(AnsibleResultRequest.class)
            .ensureIndex(new Index().on("workflowInstanceId", Sort.Direction.ASC));
        mongoTemplate.indexOps(AnsibleResultRequest.class)
            .ensureIndex(new Index().on("transactionId", Sort.Direction.ASC));
        mongoTemplate.indexOps(AnsibleResultRequest.class)
            .ensureIndex(new Index().on("orderId", Sort.Direction.ASC));
        mongoTemplate.indexOps(AnsibleResultRequest.class)
            .ensureIndex(new Index().on("csi", Sort.Direction.ASC));
        mongoTemplate.indexOps(AnsibleResultRequest.class)
            .ensureIndex(new Index()
                .on("workflowInstanceId", Sort.Direction.ASC)
                .on("stepOrder", Sort.Direction.ASC));
        
        // TransactionLogs indexes
        mongoTemplate.indexOps(TransactionLogs.class)
            .ensureIndex(new Index().on("workflowInstanceId", Sort.Direction.ASC));
        mongoTemplate.indexOps(TransactionLogs.class)
            .ensureIndex(new Index().on("certificateId", Sort.Direction.ASC));
        mongoTemplate.indexOps(TransactionLogs.class)
            .ensureIndex(new Index().on("csi", Sort.Direction.ASC));
        mongoTemplate.indexOps(TransactionLogs.class)
            .ensureIndex(new Index().on("date", Sort.Direction.DESC));
        mongoTemplate.indexOps(TransactionLogs.class)
            .ensureIndex(new Index().on("transactionId", Sort.Direction.DESC));
        
        log.info("MongoDB indexes created successfully");
    }
}

package com.citi.clm.config;

import com.citi.clm.entity.WorkflowDefinition;
import com.citi.clm.entity.WorkflowDefinition.WorkflowStepDefinition;
import com.citi.clm.enums.WorkflowType;
import com.citi.clm.repository.WorkflowDefinitionRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

import java.util.Date;
import java.util.List;

@Slf4j
@Component
@RequiredArgsConstructor
public class DataInitializer {
    
    private final WorkflowDefinitionRepository workflowDefinitionRepository;
    private final WorkflowConfig workflowConfig;
    
    @EventListener(ApplicationReadyEvent.class)
    public void initializeData() {
        log.info("Initializing workflow definitions...");
        
        // Initialize workflow config defaults
        workflowConfig.initDefaults();
        
        // Create new workflow definition if not exists
        if (workflowDefinitionRepository.findByWorkflowTypeAndActiveTrue(WorkflowType.NEW_WORKFLOW).isEmpty()) {
            createNewWorkflowDefinition();
        }
        
        // Create old workflow definition if not exists
        if (workflowDefinitionRepository.findByWorkflowTypeAndActiveTrue(WorkflowType.OLD_WORKFLOW).isEmpty()) {
            createOldWorkflowDefinition();
        }
        
        log.info("Workflow definitions initialized");
    }
    
    private void createNewWorkflowDefinition() {
        WorkflowDefinition definition = WorkflowDefinition.builder()
            .name("New Renewal Workflow")
            .description("Certificate renewal workflow for WAS and IHS tech stacks")
            .workflowType(WorkflowType.NEW_WORKFLOW)
            .applicableProductTypes(List.of("WAS", "IHS"))
            .steps(List.of(
                WorkflowStepDefinition.builder()
                    .stepOrder(1)
                    .stepName("Procurement")
                    .stepDescription("Procure new certificate from CA")
                    .playbookName("clm_procurement") // TODO: Update with actual playbook name
                    .ioAction("PROCUREMENT")
                    .requiresDeploymentCheck(false)
                    .isOptional(false)
                    .timeoutMinutes(30)
                    .build(),
                WorkflowStepDefinition.builder()
                    .stepOrder(2)
                    .stepName("Deployment")
                    .stepDescription("Deploy certificate to target server")
                    .playbookName("clm_deployment") // TODO: Update with actual playbook name
                    .ioAction("DEPLOYMENT")
                    .requiresDeploymentCheck(true)
                    .isOptional(false)
                    .timeoutMinutes(30)
                    .build(),
                WorkflowStepDefinition.builder()
                    .stepOrder(3)
                    .stepName("Restart")
                    .stepDescription("Restart services to apply certificate")
                    .playbookName("clm_restart") // TODO: Update with actual playbook name
                    .ioAction("RESTART")
                    .requiresDeploymentCheck(true)
                    .isOptional(false)
                    .timeoutMinutes(30)
                    .build()
            ))
            .rollbackStep(WorkflowStepDefinition.builder()
                .stepOrder(99)
                .stepName("Rollback")
                .stepDescription("Rollback to previous certificate")
                .playbookName("clm_rollback") // TODO: Update with actual playbook name
                .ioAction("ROLLBACK")
                .requiresDeploymentCheck(false)
                .isOptional(true)
                .timeoutMinutes(30)
                .build())
            .active(true)
            .version(1)
            .createdDate(new Date())
            .updatedDate(new Date())
            .createdBy("SYSTEM")
            .updatedBy("SYSTEM")
            .build();
        
        workflowDefinitionRepository.save(definition);
        log.info("Created New Workflow Definition");
    }
    
    private void createOldWorkflowDefinition() {
        WorkflowDefinition definition = WorkflowDefinition.builder()
            .name("Old Renewal Workflow")
            .description("Certificate renewal workflow for other tech stacks (Apache, Nginx, etc.)")
            .workflowType(WorkflowType.OLD_WORKFLOW)
            .applicableProductTypes(List.of("APACHE", "NGINX", "TOMCAT", "OTHER"))
            .steps(List.of(
                WorkflowStepDefinition.builder()
                    .stepOrder(1)
                    .stepName("CMP Creation")
                    .stepDescription("Create certificate management package")
                    .playbookName("clm_cmp_creation") // TODO: Update with actual playbook name
                    .ioAction("CMP_CREATION")
                    .requiresDeploymentCheck(false)
                    .isOptional(false)
                    .timeoutMinutes(30)
                    .build(),
                WorkflowStepDefinition.builder()
                    .stepOrder(2)
                    .stepName("Import")
                    .stepDescription("Import certificate to keystore")
                    .playbookName("clm_import") // TODO: Update with actual playbook name
                    .ioAction("IMPORT")
                    .requiresDeploymentCheck(false)
                    .isOptional(false)
                    .timeoutMinutes(30)
                    .build(),
                WorkflowStepDefinition.builder()
                    .stepOrder(3)
                    .stepName("Push")
                    .stepDescription("Push certificate to target server")
                    .playbookName("clm_push") // TODO: Update with actual playbook name
                    .ioAction("PUSH")
                    .requiresDeploymentCheck(true)
                    .isOptional(false)
                    .timeoutMinutes(30)
                    .build(),
                WorkflowStepDefinition.builder()
                    .stepOrder(4)
                    .stepName("CR Validation")
                    .stepDescription("Validate certificate request")
                    .playbookName("clm_cr_validation") // TODO: Update with actual playbook name
                    .ioAction("CR_VALIDATION")
                    .requiresDeploymentCheck(false)
                    .isOptional(false)
                    .timeoutMinutes(30)
                    .build(),
                WorkflowStepDefinition.builder()
                    .stepOrder(5)
                    .stepName("Restart")
                    .stepDescription("Restart services to apply certificate")
                    .playbookName("clm_restart") // TODO: Update with actual playbook name
                    .ioAction("RESTART")
                    .requiresDeploymentCheck(true)
                    .isOptional(false)
                    .timeoutMinutes(30)
                    .build()
            ))
            .rollbackStep(WorkflowStepDefinition.builder()
                .stepOrder(99)
                .stepName("Rollback")
                .stepDescription("Rollback to previous certificate")
                .playbookName("clm_rollback") // TODO: Update with actual playbook name
                .ioAction("ROLLBACK")
                .requiresDeploymentCheck(false)
                .isOptional(true)
                .timeoutMinutes(30)
                .build())
            .active(true)
            .version(1)
            .createdDate(new Date())
            .updatedDate(new Date())
            .createdBy("SYSTEM")
            .updatedBy("SYSTEM")
            .build();
        
        workflowDefinitionRepository.save(definition);
        log.info("Created Old Workflow Definition");
    }
}

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.18</version>
        <relativePath/>
    </parent>
    
    <groupId>com.citi</groupId>
    <artifactId>clm-service</artifactId>
    <version>2.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>CLM Service</name>
    <description>Certificate Lifecycle Management Service with IO API Integration</description>
    
    <properties>
        <java.version>11</java.version>
        <lombok.version>1.18.30</lombok.version>
        <springdoc.version>1.7.0</springdoc.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
        
        <!-- OpenAPI / Swagger -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-ui</artifactId>
            <version>${springdoc.version}</version>
        </dependency>
        
        <!-- Jackson for JSON -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        
        <dependency>
            <groupId>com.fasterxml.jackson.datatype</groupId>
            <artifactId>jackson-datatype-jsr310</artifactId>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <dependency>
            <groupId>de.flapdoodle.embed</groupId>
            <artifactId>de.flapdoodle.embed.mongo</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>


---------------------------------------------------------------------------------------

# CLM Enhancement - Technical Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Code Walkthrough](#code-walkthrough)
4. [Configuration Guide](#configuration-guide)
5. [API Reference](#api-reference)
6. [Database Schema](#database-schema)
7. [Workflow Execution Flow](#workflow-execution-flow)
8. [TODOs and Integration Points](#todos-and-integration-points)

---

## Overview

This enhancement redesigns the CLM (Certificate Lifecycle Management) system to:
- Replace Ansible Tower with IO API (Inventory Orchestrator) as the default execution environment
- Support dual workflows: New (WAS/IHS) and Old (other tech stacks)
- Implement CSI-level controls for auto-renewal, auto-deployment, and execution environment preferences
- Provide comprehensive audit logging and transaction tracking

### Key Features
- **Fire-and-forget execution**: Playbooks call back to `/result` endpoint on completion
- **CSI validation**: Checks auto-renewal/deployment flags before each action
- **Configurable workflows**: Step definitions stored in DB and application.yml
- **Manual retry support**: Failed steps can be retried from the admin portal

---

## Architecture

### Package Structure

```
com.citi.clm
├── config/                    # Configuration classes
│   ├── IOApiConfig           # IO API connection settings
│   ├── WorkflowConfig        # Workflow and playbook configuration
│   ├── SchedulerConfig       # Daily scheduler settings
│   ├── DataInitializer       # Seeds workflow definitions
│   └── MongoIndexConfig      # Database indexes
│
├── controller/               # REST API endpoints
│   ├── ResultCallbackController    # /result endpoint for playbook callbacks
│   ├── WorkflowAdminController     # Workflow management APIs
│   └── ExecutionLogController      # Log viewing and IO API log fetch
│
├── service/                  # Business logic
│   ├── io/
│   │   ├── IOAuthService          # Token management with caching
│   │   └── IOExecutionService     # Execute playbooks, get status/logs
│   ├── workflow/
│   │   ├── WorkflowOrchestrator   # Main workflow coordination
│   │   ├── WorkflowStateManager   # State transitions
│   │   └── StepExecutor           # Individual step execution
│   ├── CsiValidationService       # CSI restriction checks
│   ├── RenewalSchedulerService    # Daily renewal job
│   ├── ResultCallbackService      # Process playbook results
│   ├── NotificationService        # Email notifications
│   └── TransactionLogService      # Audit logging
│
├── entity/                   # MongoDB documents
├── repository/               # MongoDB repositories
├── dto/                      # Data transfer objects
├── enums/                    # Enumerations
└── exception/                # Custom exceptions
```

### Component Interaction

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Scheduler  │────>│  Orchestrator │────>│   IO API    │
└─────────────┘     └──────────────┘     └─────────────┘
                           │                     │
                           │                     │
                    ┌──────▼──────┐        ┌─────▼─────┐
                    │   State     │        │  Playbook │
                    │   Manager   │        │ Execution │
                    └─────────────┘        └─────┬─────┘
                                                 │
                           ┌─────────────────────┘
                           │
                    ┌──────▼──────┐
                    │  /result    │
                    │  Callback   │
                    └─────────────┘
```

---

## Code Walkthrough

### 1. Workflow Initiation

**Entry Point**: `RenewalSchedulerService.checkAndQueueRenewals()`

```java
// Daily at 2 AM - finds certificates expiring within threshold
@Scheduled(cron = "${clm.scheduler.renewal-check-cron}")
public void checkAndQueueRenewals() {
    // 1. Find certificates expiring soon
    List<CertificateDetails> certs = findCertificatesForRenewal();
    
    // 2. Group by CSI to apply custom thresholds
    Map<Integer, List<CertificateDetails>> byCsi = ...;
    
    // 3. Queue for processing
    for (CertificateDetails cert : eligibleCerts) {
        renewalQueue.offer(cert.getId());
    }
    
    // 4. Process queue
    processRenewalQueue();
}
```

### 2. CSI Validation

**Service**: `CsiValidationService`

```java
public CsiDetails validateAutoRenewalEnabled(Integer csi, String certificateId) {
    CsiDetails csiDetails = getCsiDetails(csi);
    ModuleConfiguration config = getClmConfiguration(csiDetails);
    
    // Default is true if config is null
    boolean autoRenewalEnabled = config == null || config.isAutoRenewalEnabled();
    
    if (!autoRenewalEnabled) {
        throw new CsiValidationException("Auto-renewal disabled", csi);
    }
    
    return csiDetails;
}

public ExecutionEnvironment getPreferredExecutionEnvironment(Integer csi) {
    // Check CSI config for vmPreferredExecutionEnv
    // Returns IO as default
}
```

### 3. Workflow Creation

**Service**: `WorkflowOrchestrator.initiateRenewalWorkflow()`

```java
public WorkflowInstance initiateRenewalWorkflow(String certificateId, String triggeredBy) {
    // 1. Validate CSI for auto-renewal
    csiValidationService.validateAutoRenewalEnabled(csi, certificateId);
    
    // 2. Determine workflow type (NEW for WAS/IHS, OLD for others)
    WorkflowType type = determineWorkflowType(certificate.getProductType());
    
    // 3. Get execution environment preference
    ExecutionEnvironment env = csiValidationService.getPreferredExecutionEnvironment(csi);
    
    // 4. Create workflow instance with steps
    WorkflowInstance workflow = createWorkflowInstance(certificate, type, env);
    
    // 5. Send notification
    notificationService.sendWorkflowInitiatedNotification(workflow);
    
    // 6. Execute first step
    stepExecutor.executeStep(workflow);
    
    return workflow;
}
```

### 4. Step Execution

**Service**: `StepExecutor.executeStep()`

```java
public void executeStep(WorkflowInstance workflow) {
    WorkflowStepInstance step = getCurrentStep(workflow);
    
    // Check deployment flag for push/deploy steps
    if (step.isRequiresDeploymentCheck()) {
        if (!csiValidationService.isAutoDeploymentEnabled(csi, certId)) {
            // Skip this step
            updateStepStatus(workflow, step, SKIPPED);
            moveToNextStep(workflow);
            return;
        }
    }
    
    // Execute via IO or AAP based on preference
    if (workflow.getExecutionEnvironment() == IO) {
        executeViaIO(workflow, step, certificate);
    } else {
        executeViaAAP(workflow, step, certificate);
    }
}
```

### 5. IO API Execution

**Service**: `IOExecutionService.executePlaybook()`

```java
public IOExecuteResponse executePlaybook(WorkflowInstance workflow, 
                                         WorkflowStepInstance step,
                                         CertificateDetails certificate) {
    // 1. Get auth token (cached)
    String token = ioAuthService.getAuthToken();
    
    // 2. Build request
    IOExecuteRequest request = buildExecuteRequest(workflow, step, certificate);
    
    // 3. Execute (fire-and-forget)
    ResponseEntity<IOExecuteResponse> response = restTemplate.exchange(...);
    
    // 4. Log API call
    transactionLogService.logIOApiCall(...);
    
    // 5. Return order ID for tracking
    return response.getBody();
}
```

### 6. Callback Processing

**Service**: `ResultCallbackService.processCallback()`

```java
public void processCallback(ResultCallbackRequest request) {
    // Find workflow
    WorkflowInstance workflow = findWorkflow(request);
    
    // Save result record
    saveResultRecord(request, workflow);
    
    // Process in orchestrator
    workflowOrchestrator.processPlaybookCallback(
        workflow.getId(),
        stepOrder,
        request.getExecutionStatus(),
        request.getResult(),
        errorMessage
    );
}
```

**Orchestrator callback handling**:

```java
public void processPlaybookCallback(String workflowId, int stepOrder, 
                                    String status, Object result) {
    if ("SUCCESS".equals(status)) {
        // Mark step completed
        updateStepStatus(workflow, stepOrder, COMPLETED);
        
        // Move to next step
        workflow = moveToNextStep(workflow);
        
        // Execute next step or complete workflow
        if (workflow.getStatus() == IN_PROGRESS) {
            stepExecutor.executeStep(workflow);
        } else if (workflow.getStatus() == COMPLETED) {
            handleWorkflowCompletion(workflow);
        }
    } else {
        // Mark step for retry
        updateStepStatus(workflow, stepOrder, RETRY_PENDING);
        pauseWorkflow(workflow, "Step failed");
        notificationService.sendStepFailureNotification(...);
    }
}
```

---

## Configuration Guide

### IO API Configuration

In `application.yml`:

```yaml
clm:
  io-api:
    base-url: https://b2b.cti.otservices.citigroup.net
    basic-auth-credentials: ${IO_API_BASIC_AUTH}  # Base64 encoded
    cmt-host-url: ${CMT_HOST_URL}  # CLM callback URL
    inventory-host-url: ${INVENTORY_HOST_URL}
    programmatic-name: platformforyou_clm
```

### Workflow Configuration

```yaml
clm:
  workflow:
    default-expiry-threshold-days: 60
    new-workflow-product-types:
      - WAS
      - IHS
    
    new-workflow:
      steps:
        procurement:
          playbook-name: clm_procurement
          io-action: PROCUREMENT
          order: 1
          requires-deployment-check: false
```

### Scheduler Configuration

```yaml
clm:
  scheduler:
    enabled: true
    renewal-check-cron: "0 0 2 * * ?"  # Daily at 2 AM
    max-concurrent-workflows: 10
```

---

## API Reference

### Callback Endpoint

**POST** `/api/v1/result`

Called by playbooks to report execution results.

```json
{
  "workflowInstanceId": "...",
  "stepOrder": 1,
  "transactionId": "...",
  "executionStatus": "SUCCESS",
  "result": { ... },
  "servername": "...",
  "orderId": "...",
  "podName": "..."
}
```

### Workflow Management

**POST** `/api/v1/workflow/trigger` - Manually trigger renewal
**GET** `/api/v1/workflow/{workflowId}` - Get workflow status
**GET** `/api/v1/workflow/active` - List active workflows
**POST** `/api/v1/workflow/{id}/step/{stepOrder}/retry` - Retry failed step
**POST** `/api/v1/workflow/{id}/cancel` - Cancel workflow

### Execution Logs

**GET** `/api/v1/logs/io/order/{orderId}/status` - Get IO order status
**GET** `/api/v1/logs/io/order/{orderId}/pods` - List execution pods
**GET** `/api/v1/logs/io/pod/{podName}/log` - Download pod log

---

## Database Schema

### Key Collections

1. **csi_details** - CSI configuration with module subscriptions
2. **certificate_details** - Certificate information and renewal status
3. **workflow_instances** - Runtime workflow state and step status
4. **AnsibleResultRequest** - Execution results (IO and AAP)
5. **Transaction_Logs_Inventory** - Comprehensive audit trail
6. **workflow_definitions** - Workflow templates (seeded on startup)

### Important Indexes

```javascript
// Performance indexes created on startup
db.certificate_details.createIndex({ "csi": 1, "expiry": 1, "renewalStatus": 1 })
db.workflow_instances.createIndex({ "status": 1, "certificateId": 1 })
db.AnsibleResultRequest.createIndex({ "workflowInstanceId": 1, "stepOrder": 1 })
```

---

## Workflow Execution Flow

### New Workflow (WAS/IHS)

```
1. PROCUREMENT → 2. DEPLOYMENT* → 3. RESTART*
                      ↓ if deployment disabled, skip
                      SKIPPED
```

### Old Workflow (Others)

```
1. CMP_CREATION → 2. IMPORT → 3. PUSH* → 4. CR_VALIDATION → 5. RESTART*
                                  ↓ if deployment disabled
                                SKIPPED
```

*Steps marked with * require `autoDeploymentEnabled = true`

### State Transitions

```
PENDING → IN_PROGRESS → COMPLETED
                ↓
              FAILED
                ↓
              PAUSED (manual retry needed)
                ↓
            IN_PROGRESS (after retry)
```

---

## TODOs and Integration Points

### High Priority TODOs

1. **IO API Credentials** (IOApiConfig)
   - `basic-auth-credentials` - Base64 encoded Basic auth
   
2. **Callback URLs** (IOApiConfig)
   - `cmt-host-url` - CLM callback URL for /result
   - `inventory-host-url` - Inventory host URL

3. **Playbook Names** (application.yml)
   - All playbook names are placeholders
   - Update for each step in new-workflow and old-workflow

4. **Execute Request Parameters** (IOExecutionService.buildExecuteRequest)
   - `principal` - Principal ID format
   - `ticket` - SNOW ticket generation
   - Additional certificate-specific parameters

5. **PFY Token Generation** (StepExecutor.generatePfyToken)
   - Token for callback authentication

### Medium Priority TODOs

6. **AAP Integration** (StepExecutor.executeViaAAP)
   - Implement Ansible Tower execution path
   - For CSIs with AAP preference

7. **Notification Playbook** (NotificationService)
   - Execute notification playbook via IO
   - Pass HTML content and recipients

8. **Rollback Execution** (WorkflowOrchestrator.triggerRollback)
   - Implement rollback step execution

---

## Testing

### Unit Tests
- Test CSI validation with various configurations
- Test workflow state transitions
- Test step skipping logic

### Integration Tests
- Test IO API calls with mocks
- Test callback processing
- Test concurrent workflow execution

### End-to-End Tests
- Full renewal workflow (new and old)
- CSI restriction enforcement
- Error recovery and retry

---

## Deployment Notes

1. **Environment Variables**
   - `MONGODB_URI`
   - `IO_API_BASIC_AUTH`
   - `CMT_HOST_URL`
   - `INVENTORY_HOST_URL`
   - `SCHEDULER_ENABLED`

2. **MongoDB Indexes**
   - Automatically created on startup
   - Verify after first deployment

3. **Workflow Definitions**
   - Seeded on first startup
   - Update via admin portal after deployment

4. **Feature Flags**
   - `clm.scheduler.enabled` - Enable/disable daily scheduler
   - Use for gradual rollout
  
-------------------------------------------------------------------------------_------------------------------------------------------------------------------_
Generated Code Summary
Total Files: 35+ Java files
Core Components Created:

Enums (6 files): ExecutionEnvironment, WorkflowStatus, StepStatus, WorkflowType, ModuleType, PlaybookExecutionEnvironment
Entities (8 files): CsiDetails, CertificateDetails, ModuleConfiguration, ModuleSubscription, WorkflowDefinition, WorkflowInstance, AnsibleResultRequest, TransactionLogs
DTOs (9 files): IO API request/response objects, ResultCallbackRequest, WorkflowTriggerRequest
Repositories (6 files): All MongoDB repository interfaces with custom queries
Services (9 files):

IOAuthService - Token caching and refresh
IOExecutionService - Execute playbooks, get status/pods/logs
CsiValidationService - Auto-renewal/deployment checks
WorkflowOrchestrator - Main workflow coordination
WorkflowStateManager - State transitions
StepExecutor - Individual step execution
RenewalSchedulerService - Daily scheduler
ResultCallbackService - Callback processing
NotificationService - Email notifications
TransactionLogService - Audit logging


Controllers (3 files): ResultCallbackController, WorkflowAdminController, ExecutionLogController
Configuration (6 files): IOApiConfig, WorkflowConfig, SchedulerConfig, RestTemplateConfig, MongoIndexConfig, DataInitializer


Key TODOs to Complete
Before deployment, you need to provide:
ItemLocationDescriptionIO API Basic Authapplication.ymlBase64 encoded credentialsCMT Host URLapplication.ymlCLM callback URL for /resultInventory Host URLapplication.ymlInventory service URLPlaybook Namesapplication.ymlActual names for all playbooksPrincipal IDIOExecutionServicePrincipal format for IO requestsSNOW TicketIOExecutionServiceTicket generation logicPFY TokenStepExecutorCallback authentication token
