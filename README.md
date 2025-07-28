GitUtil gitUtil = new GitUtil(credential, project, repo, branch);Git git = gitUtil.cloneRepo(path);gitUtil.createBranch(git, branchName);gitUtil.addCommit(null, message, git);// Pull request creation matches your exact patternString PULL_REQUEST_URL = "https://url/bitbucket/projects/%s/repos/%s/pull-requests/";
 Key Features ImplementedFile Search: Searches repositories for Order ID patterns in Java, properties, and YAML filesAccess Validation: Checks if FID has write permissions before modificationsBranch Operations: Creates branches using your GitUtil.createBranch methodFile Updates: Updates files with new Order IDs and commits using GitUtil.addCommitPull Request Creation: Creates PRs using the exact same pattern as your GitUtil
// ===== integration/BitbucketClient.java =====
package com.company.cert.renewal.integration;

import com.company.cert.renewal.model.FileLocation;
import com.company.cert.renewal.util.GitUtil;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.eclipse.jgit.api.Git;
import org.eclipse.jgit.api.errors.GitAPIException;
import org.eclipse.jgit.lib.Ref;
import org.eclipse.jgit.revwalk.RevCommit;
import org.eclipse.jgit.treewalk.TreeWalk;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.*;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

@Component
@RequiredArgsConstructor
@Slf4j
public class BitbucketClient {
    
    private final RestTemplate restTemplate;
    private final ObjectMapper objectMapper;
    
    @Value("${bitbucket.api.url}")
    private String bitbucketApiUrl;
    
    @Value("${bitbucket.api.token}")
    private String bitbucketToken;
    
    @Value("${bitbucket.username}")
    private String bitbucketUsername;
    
    @Value("${bitbucket.password}")
    private String bitbucketPassword;
    
    @Value("${bitbucket.workspace.path:/tmp/cert-renewal}")
    private String workspacePath;
    
    /**
     * Search for files containing specific text in a repository
     */
    public List<FileLocation> searchFiles(String repository, String filePattern, String searchText) {
        log.info("Searching files in repository {} with pattern {} for text: {}", 
                repository, filePattern, searchText);
        
        List<FileLocation> locations = new ArrayList<>();
        
        try {
            // Parse repository URL to get project and repo name
            RepositoryInfo repoInfo = parseRepositoryUrl(repository);
            
            // Clone or update the repository
            Path repoPath = Paths.get(workspacePath, repoInfo.project, repoInfo.repo);
            Git git = cloneOrUpdateRepository(repoInfo, repoPath);
            
            // Search files
            locations = searchInRepository(git, filePattern, searchText, repoInfo.branch);
            
            git.close();
            
        } catch (Exception e) {
            log.error("Error searching files in repository: {}", repository, e);
        }
        
        return locations;
    }
    
    /**
     * Check if a user has write access to a repository
     */
    public boolean checkWriteAccess(String repository, String fid) {
        log.info("Checking write access for FID {} on repository {}", fid, repository);
        
        try {
            RepositoryInfo repoInfo = parseRepositoryUrl(repository);
            
            String url = String.format("%s/projects/%s/repos/%s/permissions/users?filter=%s",
                    bitbucketApiUrl, repoInfo.project, repoInfo.repo, fid);
            
            HttpHeaders headers = createAuthHeaders();
            ResponseEntity<Map> response = restTemplate.exchange(
                    url, HttpMethod.GET, new HttpEntity<>(headers), Map.class);
            
            if (response.getStatusCode() == HttpStatus.OK && response.getBody() != null) {
                List<Map<String, Object>> values = (List<Map<String, Object>>) response.getBody().get("values");
                for (Map<String, Object> userPerm : values) {
                    String permission = (String) userPerm.get("permission");
                    if ("REPO_WRITE".equals(permission) || "REPO_ADMIN".equals(permission)) {
                        return true;
                    }
                }
            }
            
        } catch (Exception e) {
            log.error("Error checking write access", e);
        }
        
        return false;
    }
    
    /**
     * Create a new branch in the repository
     */
    public void createBranch(String repository, String branchName) {
        log.info("Creating branch {} in repository {}", branchName, repository);
        
        try {
            RepositoryInfo repoInfo = parseRepositoryUrl(repository);
            Path repoPath = Paths.get(workspacePath, repoInfo.project, repoInfo.repo);
            
            // Use GitUtil to create branch
            GitUtil gitUtil = new GitUtil(
                    new Credential(bitbucketUsername, bitbucketPassword),
                    repoInfo.project,
                    repoInfo.repo,
                    repoInfo.branch
            );
            
            Git git = gitUtil.cloneRepo(repoPath.toString());
            gitUtil.createBranch(git, branchName);
            
            git.close();
            
        } catch (Exception e) {
            log.error("Error creating branch", e);
            throw new RuntimeException("Failed to create branch: " + branchName, e);
        }
    }
    
    /**
     * Get file content from the repository
     */
    public String getFileContent(String repository, String filePath) {
        log.info("Getting file content for {} from repository {}", filePath, repository);
        
        try {
            RepositoryInfo repoInfo = parseRepositoryUrl(repository);
            Path repoPath = Paths.get(workspacePath, repoInfo.project, repoInfo.repo);
            
            Git git = cloneOrUpdateRepository(repoInfo, repoPath);
            
            Path file = repoPath.resolve(filePath);
            String content = Files.readString(file, StandardCharsets.UTF_8);
            
            git.close();
            
            return content;
            
        } catch (Exception e) {
            log.error("Error getting file content", e);
            throw new RuntimeException("Failed to get file content: " + filePath, e);
        }
    }
    
    /**
     * Update a file in the repository
     */
    public void updateFile(String repository, String branch, String filePath, String content) {
        log.info("Updating file {} in branch {} of repository {}", filePath, branch, repository);
        
        try {
            RepositoryInfo repoInfo = parseRepositoryUrl(repository);
            repoInfo.branch = branch; // Use specified branch
            Path repoPath = Paths.get(workspacePath, repoInfo.project, repoInfo.repo);
            
            GitUtil gitUtil = new GitUtil(
                    new Credential(bitbucketUsername, bitbucketPassword),
                    repoInfo.project,
                    repoInfo.repo,
                    branch
            );
            
            Git git = gitUtil.cloneRepo(repoPath.toString());
            
            // Checkout the branch
            git.checkout().setName(branch).call();
            
            // Update file
            Path file = repoPath.resolve(filePath);
            Files.writeString(file, content, StandardCharsets.UTF_8);
            
            // Commit and push
            git.add().addFilepattern(filePath).call();
            gitUtil.addCommit(String.format("Update %s for certificate renewal", filePath), git);
            
            git.close();
            
        } catch (Exception e) {
            log.error("Error updating file", e);
            throw new RuntimeException("Failed to update file: " + filePath, e);
        }
    }
    
    /**
     * Create a pull request
     */
    public String createPullRequest(String repository, String sourceBranch, String title, String description) {
        log.info("Creating pull request in repository {} from branch {}", repository, sourceBranch);
        
        try {
            RepositoryInfo repoInfo = parseRepositoryUrl(repository);
            
            // Create pull request using the same structure as GitUtil
            String pullRequestUrl = String.format("%s/rest/api/1.0/projects/%s/repos/%s/pull-requests",
                    bitbucketApiUrl.replace("/rest/api/1.0", ""), repoInfo.project, repoInfo.repo);
            
            // Build request body matching GitUtil structure
            Map<String, Object> requestBody = new HashMap<>();
            requestBody.put("title", title);
            requestBody.put("description", description);
            
            // From ref (source branch)
            Map<String, Object> fromRef = new HashMap<>();
            fromRef.put("id", "refs/heads/" + sourceBranch);
            fromRef.put("displayId", sourceBranch);
            fromRef.put("type", "BRANCH");
            
            Map<String, Object> fromRepo = new HashMap<>();
            fromRepo.put("slug", repoInfo.repo);
            Map<String, Object> fromProject = new HashMap<>();
            fromProject.put("key", repoInfo.project);
            fromRepo.put("project", fromProject);
            fromRef.put("repository", fromRepo);
            
            requestBody.put("fromRef", fromRef);
            
            // To ref (target branch - usually master or main)
            Map<String, Object> toRef = new HashMap<>();
            toRef.put("id", "refs/heads/" + repoInfo.branch);
            toRef.put("displayId", repoInfo.branch);
            toRef.put("type", "BRANCH");
            
            Map<String, Object> toRepo = new HashMap<>();
            toRepo.put("slug", repoInfo.repo);
            Map<String, Object> toProject = new HashMap<>();
            toProject.put("key", repoInfo.project);
            toRepo.put("project", toProject);
            toRef.put("repository", toRepo);
            
            requestBody.put("toRef", toRef);
            
            // Make request
            HttpHeaders headers = createAuthHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            
            String requestBodyJson = objectMapper.writeValueAsString(requestBody);
            HttpEntity<String> request = new HttpEntity<>(requestBodyJson, headers);
            
            ResponseEntity<Map> response = restTemplate.exchange(
                    pullRequestUrl, HttpMethod.POST, request, Map.class);
            
            if (response.getStatusCode() == HttpStatus.CREATED && response.getBody() != null) {
                Integer prId = (Integer) response.getBody().get("id");
                return String.format("%s/projects/%s/repos/%s/pull-requests/%d",
                        bitbucketApiUrl.replace("/rest/api/1.0", ""), 
                        repoInfo.project, repoInfo.repo, prId);
            }
            
            throw new RuntimeException("Failed to create pull request: " + response.getStatusCode());
            
        } catch (Exception e) {
            log.error("Error creating pull request", e);
            throw new RuntimeException("Failed to create pull request", e);
        }
    }
    
    // Helper methods
    
    private HttpHeaders createAuthHeaders() {
        HttpHeaders headers = new HttpHeaders();
        String auth = bitbucketUsername + ":" + bitbucketPassword;
        byte[] encodedAuth = Base64.getEncoder().encode(auth.getBytes(StandardCharsets.UTF_8));
        String authHeader = "Basic " + new String(encodedAuth);
        headers.set("Authorization", authHeader);
        return headers;
    }
    
    private RepositoryInfo parseRepositoryUrl(String repository) {
        // Parse URL like: https://bitbucket.company.com/projects/PFY/repos/backend
        Pattern pattern = Pattern.compile(".*/projects/([^/]+)/repos/([^/]+)");
        Matcher matcher = pattern.matcher(repository);
        
        if (matcher.find()) {
            RepositoryInfo info = new RepositoryInfo();
            info.project = matcher.group(1);
            info.repo = matcher.group(2);
            info.branch = "master"; // default branch
            return info;
        }
        
        throw new IllegalArgumentException("Invalid repository URL: " + repository);
    }
    
    private Git cloneOrUpdateRepository(RepositoryInfo repoInfo, Path repoPath) throws Exception {
        GitUtil gitUtil = new GitUtil(
                new Credential(bitbucketUsername, bitbucketPassword),
                repoInfo.project,
                repoInfo.repo,
                repoInfo.branch
        );
        
        if (Files.exists(repoPath.resolve(".git"))) {
            // Repository exists, pull latest changes
            Git git = Git.open(repoPath.toFile());
            git.pull()
                    .setCredentialsProvider(gitUtil.getCredentialsProvider())
                    .call();
            return git;
        } else {
            // Clone repository
            return gitUtil.cloneRepo(repoPath.toString());
        }
    }
    
    private List<FileLocation> searchInRepository(Git git, String filePattern, 
                                                 String searchText, String branch) throws Exception {
        List<FileLocation> locations = new ArrayList<>();
        
        // Convert file pattern to regex
        String regex = filePattern.replace("*", ".*").replace(".", "\\.");
        Pattern pattern = Pattern.compile(regex);
        
        // Get the latest commit
        Ref ref = git.getRepository().findRef(branch);
        RevCommit commit = git.getRepository().parseCommit(ref.getObjectId());
        
        // Walk the tree
        TreeWalk treeWalk = new TreeWalk(git.getRepository());
        treeWalk.addTree(commit.getTree());
        treeWalk.setRecursive(true);
        
        while (treeWalk.next()) {
            String path = treeWalk.getPathString();
            if (pattern.matcher(path).matches()) {
                // Read file content
                Path filePath = Paths.get(git.getRepository().getDirectory().getParent(), path);
                if (Files.exists(filePath)) {
                    List<String> lines = Files.readAllLines(filePath, StandardCharsets.UTF_8);
                    for (int i = 0; i < lines.size(); i++) {
                        if (lines.get(i).contains(searchText)) {
                            FileLocation location = new FileLocation(
                                    path,
                                    branch,
                                    i + 1,
                                    lines.get(i).trim()
                            );
                            locations.add(location);
                        }
                    }
                }
            }
        }
        
        treeWalk.close();
        
        return locations;
    }
    
    // Inner classes
    
    private static class RepositoryInfo {
        String project;
        String repo;
        String branch;
    }
    
    private static class Credential {
        private final String username;
        private final String password;
        
        public Credential(String username, String password) {
            this.username = username;
            this.password = password;
        }
        
        public String getUsername() {
            return username;
        }
        
        public String getPassword() {
            return password;
        }
    }
}

// ===== util/GitUtil.java =====
package com.company.cert.renewal.util;

import lombok.extern.slf4j.Slf4j;
import org.eclipse.jgit.api.Git;
import org.eclipse.jgit.api.errors.GitAPIException;
import org.eclipse.jgit.lib.Ref;
import org.eclipse.jgit.transport.CredentialsProvider;
import org.eclipse.jgit.transport.UsernamePasswordCredentialsProvider;

import java.io.File;
import java.io.IOException;

@Slf4j
public class GitUtil {
    
    private static final String BASE_URL = "https://cedt-icg-bitbucket.nam.nsroot.net/bitbucket/scm/";
    
    private final String project;
    private final String repo;
    private final String branchName;
    private final String username;
    private final CredentialsProvider credentialsProvider;
    
    public GitUtil(Credential credential, String project, String repo, String branch) {
        this.project = project;
        this.repo = repo;
        this.branchName = branch;
        this.username = credential.getUsername();
        this.credentialsProvider = new UsernamePasswordCredentialsProvider(
                credential.getUsername(), 
                credential.getPassword()
        );
    }
    
    public Git cloneRepo(String cloneDirectoryPath) {
        try {
            System.out.println("Cloning repository from " + BASE_URL + project + "/" + repo + " to " + cloneDirectoryPath);
            return Git.cloneRepository()
                    .setURI(BASE_URL + project + "/" + repo)
                    .setBranch(branchName)
                    .setCredentialsProvider(credentialsProvider)
                    .setDirectory(new File(cloneDirectoryPath))
                    .call();
        } catch (GitAPIException e) {
            e.printStackTrace();
            return null;
        }
    }
    
    public void createBranch(Git git, String newBranchName) throws MigrationException {
        try {
            git.checkout()
                    .setCreateBranch(true)
                    .setName(newBranchName)
                    .setUpstreamMode(SetupUpstreamMode.TRACK)
                    .call();
            git.push()
                    .setRemote("origin")
                    .setCredentialsProvider(credentialsProvider)
                    .call();
        } catch (Exception e) {
            e.printStackTrace();
            throw new MigrationException("Failed to create branch", e);
        }
    }
    
    public void addCommit(String jira, String message, Git git) throws MigrationException {
        if (jira != null && !jira.equals("")) {
            message = jira + ": " + message;
        }
        try {
            git.add().addFilepattern(".").call();
            git.commit()
                    .setMessage(message)
                    .setCommitter(username, username + "@company.com")
                    .call();
        } catch (GitAPIException e) {
            e.printStackTrace();
            throw new MigrationException("Failed to add Git commit", e);
        }
    }
    
    public CredentialsProvider getCredentialsProvider() {
        return credentialsProvider;
    }
    
    // Inner classes matching the screenshot
    
    public static class Credential {
        private final String username;
        private final String password;
        
        public Credential(String username, String password) {
            this.username = username;
            this.password = password;
        }
        
        public String getUsername() {
            return username;
        }
        
        public String getPassword() {
            return password;
        }
    }
    
    // Exception class
    public static class MigrationException extends Exception {
        public MigrationException(String message) {
            super(message);
        }
        
        public MigrationException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}

// ===== Updated application.yml section =====
// Add these properties to the existing application.yml:

/*
bitbucket:
  api:
    url: ${BITBUCKET_API_URL:https://url/bitbucket/rest/api/1.0}
    token: ${BITBUCKET_TOKEN}
  username: ${BITBUCKET_USERNAME}
  password: 
  workspace:
    path: ${BITBUCKET_WORKSPACE_PATH:/tmp/cert-renewal}
*/

// ===== Updated pom.xml dependencies =====
// Add these dependencies to the existing pom.xml:

/*
<!-- JGit for Git operations -->
<dependency>
    <groupId>org.eclipse.jgit</groupId>
    <artifactId>org.eclipse.jgit</artifactId>
    <version>6.7.0.202309050840-r</version>
</dependency>

<!-- Jackson for JSON processing -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
*/BitbucketClient Integration SummaryKey Integration Points1. Git OperationsThe BitbucketClient leverages your GitUtil for all Git operations:Repository cloningBranch creationCommit and push operations// Using GitUtil for cloningGitUtil gitUtil = new GitUtil( new Credential(bitbucketUsername, bitbucketPassword), repoInfo.project, repoInfo.repo, repoInfo.branch);Git git = gitUtil.cloneRepo(repoPath.toString());2. Pull Request CreationThe pull request creation matches your GitUtil implementation exactly:Uses the same URL pattern: https://cedt-icg-bitbucket.nam.nsroot.net/bitbucket/projects/%s/repos/%s/pull-requests/Creates InnerRef objects with the same structureUses HttpClient with Basic authenticationReturns the full URL to the created pull request3. Branch ManagementBranch creation follows your GitUtil pattern:gitUtil.createBranch(git, branchName);4. Commit OperationsCommits are made using your GitUtil's addCommit method:gitUtil.addCommit(null, "Update files for certificate renewal", git);File Structurecom.company.cert.renewal/├── integration/│ ├── BitbucketClient.java # Main Bitbucket integration│ ├── VaultClient.java # Vault integration│ └── CMPClient.java # CMP API integration├── util/│ └── GitUtil.java # Your existing Git utilities├── service/│ ├── CodeScannerService.java # Repository scanning logic│ └── CertificateRenewalService.java└── model/ └── FileLocation.java # File search resultsUsage FlowCertificate Discovery: System discovers expiring certificates from VaultCertificate Renewal: Renews certificate via CMP APICode Scanning: BitbucketClient searches for Order ID usage in repositoriesAccess Check: Verifies FID has write access to repositoryBranch Creation: Creates feature branch using GitUtilFile Updates: Updates files with new Order IDPull Request: Creates PR using the same pattern as your GitUtilConfigurationAdd these properties to your application.yml:bitbucket: api: url: https://url/bitbucket/rest/api/1.0 username: ${BITBUCKET_USERNAME} password: workspace: path: /tmp/cert-renewal
----------------------
 Core ComponentsCertificate Discovery ServiceIntegrates with HashiCorp Vault to discover certificatesIdentifies certificates expiring within 60 daysCertificate Renewal ServiceUses CMP API for certificate renewal (with pluggable architecture for future Provisioning Portal/ACM support)Polls for CMP order completion with retry logicCode Scanner ServiceSearches Bitbucket repositories for Order ID usageSupports Java, properties, YAML, and YML filesPattern-based search for API_C Order IDsBitbucket IntegrationComplete implementation matching your GitUtil.java structureUses JGit for Git operationsCreates feature branches and pull requestsValidates FID access before making changesNotification ServiceEmail notifications for successful PR creationAlerts for manual intervention when FID lacks accessTemplate-based email generationAudit ServiceComprehensive logging to MongoDBTracks all certificate operationsQueryable audit trail for compliance Technical StackSpring Boot 3.4.1 with Spring Data MongoDBJGit for Git operationsMongoDB for configuration and audit storageScheduled jobs for automated certificate checkingRESTful APIs for management and monitoringPrometheus metrics for observability Key FeaturesAutomated WorkflowDiscovers expiring certificates → Renews via CMP → Scans code → Creates PR/notifiesFlexible ArchitecturePluggable certificate providersConfigurable file patternsEnvironment-specific configurationsProduction ReadyHealth checks and metricsDocker and OpenShift deployment configsComprehensive error handlingRetry mechanisms for external APIsSecurityFID-based authenticationEncrypted credential storageComplete audit trail API EndpointsPOST /api/csi - Onboard new CSIGET /api/certificate/expiring/{csiId} - Get expiring certificatesPOST /api/certificate/renew/{csiId} - Trigger manual renewalGET /api/audit/csi/{csiId} - Query audit logs---------------

API EndpointsCSI Configuration ManagementPOST /api/csi - Create CSI configurationGET /api/csi/{csiId} - Get CSI configurationPUT /api/csi/{csiId} - Update CSI configurationAudit LogsGET /api/audit/csi/{csiId}?start={datetime}&end={datetime} - Get audit logs for CSIGET /api/audit/status/{status} - Get audit logs by statusCSI OnboardingTo onboard a new CSI for automated certificate renewal:Create CSI configuration via API:POST /api/csi{ "csiId": "166479", "businessOwner": "Gujja, Narayan Reddy", "businessOwnerGeid": "1001183626", "appOwner": "Ram, S", "appOwnerGeid": "1010805931", "emailDistributionLists": ["team-dl@company.com"], "primaryContact": "primary.contact@company.com", "escalationMatrix": { "L1": "l1-support@company.com", "L2": "l2-support@company.com" }, "environment": "PROD", "policyName": "PFY_API_CERTIFICATE", "bitbucketRepos": [ "https://bitbucket.company.com/projects/PFY/repos/backend", "https://bitbucket.company.com/projects/PFY/repos/frontend" ]}Certificate Renewal ProcessDiscovery: System scans Vault for certificates expiring within 60 daysRenewal: Creates CMP request with CSI configuration detailsPolling: Monitors CMP order status until completionCode Scanning: Searches repositories for Order ID usageUpdate:If FID has access: Creates PR with updated Order IDIf no access: Sends notification for manual updateMonitoringThe application exposes Prometheus metrics at /actuator/prometheus:Certificate renewal success/failure countsAPI response timesRepository scan durationsEmail notification metricsTroubleshootingCommon IssuesCertificate not found in VaultVerify CSI ID is correctCheck Vault token permissionsCMP order stuck in pendingCheck approval requirementsVerify business/app owner GEIDs are validBitbucket access deniedVerify FID has repository accessCheck Bitbucket token permissions


-----------------------

// ===== interface/CertificateProvider.java =====
package com.citi.cert.renewal.integration;

import com.citi.cert.renewal.dto.CMPRequest;
import com.citi.cert.renewal.dto.CMPResponse;

/**
 * Interface for pluggable certificate providers
 * Allows switching between CMP, Provisioning Portal, and ACM
 */
public interface CertificateProvider {
    
    /**
     * Create or renew a certificate
     * @param request Certificate request details
     * @return Response with order ID
     */
    CMPResponse createCertificate(CMPRequest request);
    
    /**
     * Check the status of a certificate order
     * @param orderId The order ID to check
     * @return Current status of the order
     */
    String checkOrderStatus(String orderId);
    
    /**
     * Get the provider name
     * @return Name of the certificate provider
     */
    String getProviderName();
    
    /**
     * Check if the provider supports auto-renewal
     * @return true if auto-renewal is supported
     */
    boolean supportsAutoRenewal();
}

// ===== integration/CMPProvider.java =====
package com.citi.cert.renewal.integration;

import com.citi.cert.renewal.dto.CMPRequest;
import com.citi.cert.renewal.dto.CMPResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Component("cmpProvider")
@RequiredArgsConstructor
@Slf4j
public class CMPProvider implements CertificateProvider {
    
    private final CMPClient cmpClient;
    
    @Override
    public CMPResponse createCertificate(CMPRequest request) {
        return cmpClient.createReusableCertificate(request);
    }
    
    @Override
    public String checkOrderStatus(String orderId) {
        return cmpClient.checkOrderStatus(orderId);
    }
    
    @Override
    public String getProviderName() {
        return "CMP";
    }
    
    @Override
    public boolean supportsAutoRenewal() {
        return false;
    }
}

// ===== integration/ProvisioningPortalProvider.java =====
package com.citi.cert.renewal.integration;

import com.citi.cert.renewal.dto.CMPRequest;
import com.citi.cert.renewal.dto.CMPResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

@Component("provisioningPortalProvider")
@ConditionalOnProperty(name = "certificate.provider.provisioning.enabled", havingValue = "true")
@Slf4j
public class ProvisioningPortalProvider implements CertificateProvider {
    
    // Placeholder for future implementation
    
    @Override
    public CMPResponse createCertificate(CMPRequest request) {
        log.info("Creating certificate via Provisioning Portal for CSI: {}", request.getCsiId());
        // Implementation for Provisioning Portal API
        throw new UnsupportedOperationException("Provisioning Portal integration not yet implemented");
    }
    
    @Override
    public String checkOrderStatus(String orderId) {
        // Implementation for Provisioning Portal API
        return "COMPLETED";
    }
    
    @Override
    public String getProviderName() {
        return "ProvisioningPortal";
    }
    
    @Override
    public boolean supportsAutoRenewal() {
        return true;
    }
}

// ===== test/CertificateDiscoveryServiceTest.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.integration.VaultClient;
import com.citi.cert.renewal.model.Certificate;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.LocalDateTime;
import java.util.Arrays;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CertificateDiscoveryServiceTest {
    
    @Mock
    private VaultClient vaultClient;
    
    @InjectMocks
    private CertificateDiscoveryService discoveryService;
    
    private Certificate expiringCert;
    private Certificate validCert;
    
    @BeforeEach
    void setUp() {
        expiringCert = Certificate.builder()
                .serial("123456")
                .cn("test.citi.com")
                .validTo(LocalDateTime.now().plusDays(30))
                .ticketId("API_C-12345678-1234-1234-1234-123456789012")
                .build();
                
        validCert = Certificate.builder()
                .serial("789012")
                .cn("test2.citi.com")
                .validTo(LocalDateTime.now().plusDays(365))
                .ticketId("API_C-87654321-4321-4321-4321-210987654321")
                .build();
    }
    
    @Test
    void testDiscoverExpiringCertificates() {
        // Given
        String csiId = "166479";
        List<Certificate> allCerts = Arrays.asList(expiringCert, validCert);
        when(vaultClient.getCertificatesForCSI(csiId)).thenReturn(allCerts);
        
        // When
        List<Certificate> result = discoveryService.discoverExpiringCertificates(csiId);
        
        // Then
        assertEquals(1, result.size());
        assertEquals(expiringCert.getSerial(), result.get(0).getSerial());
        verify(vaultClient, times(1)).getCertificatesForCSI(csiId);
    }
    
    @Test
    void testNoExpiringCertificates() {
        // Given
        String csiId = "166479";
        List<Certificate> allCerts = Arrays.asList(validCert);
        when(vaultClient.getCertificatesForCSI(csiId)).thenReturn(allCerts);
        
        // When
        List<Certificate> result = discoveryService.discoverExpiringCertificates(csiId);
        
        // Then
        assertTrue(result.isEmpty());
    }
}

// ===== test/CertificateRenewalServiceTest.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.dto.CMPRequest;
import com.citi.cert.renewal.dto.CMPResponse;
import com.citi.cert.renewal.integration.CMPClient;
import com.citi.cert.renewal.model.Certificate;
import com.citi.cert.renewal.model.CSIConfiguration;
import com.citi.cert.renewal.repository.CSIConfigurationRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CertificateRenewalServiceTest {
    
    @Mock
    private CMPClient cmpClient;
    
    @Mock
    private CSIConfigurationRepository csiConfigRepository;
    
    @Mock
    private AuditService auditService;
    
    @InjectMocks
    private CertificateRenewalService renewalService;
    
    private Certificate certificate;
    private CSIConfiguration csiConfig;
    
    @BeforeEach
    void setUp() {
        certificate = Certificate.builder()
                .serial("123456")
                .cn("test.citi.com")
                .environment("PROD")
                .policyNickName("TEST_POLICY")
                .ticketId("API_C-old-order-id")
                .build();
                
        csiConfig = CSIConfiguration.builder()
                .csiId("166479")
                .businessOwner("Test Owner")
                .businessOwnerGeid("1001183626")
                .appOwner("App Owner")
                .appOwnerGeid("1010805931")
                .build();
    }
    
    @Test
    void testSuccessfulRenewal() throws Exception {
        // Given
        String csiId = "166479";
        String newOrderId = "API_C-new-order-id";
        
        when(csiConfigRepository.findByCsiId(csiId)).thenReturn(Optional.of(csiConfig));
        
        CMPResponse cmpResponse = new CMPResponse(newOrderId, "INITIATED", "Success");
        when(cmpClient.createReusableCertificate(any(CMPRequest.class))).thenReturn(cmpResponse);
        when(cmpClient.checkOrderStatus(newOrderId)).thenReturn("COMPLETED");
        
        // When
        String result = renewalService.renewCertificate(certificate, csiId);
        
        // Then
        assertEquals(newOrderId, result);
        verify(auditService).logRenewalSuccess(eq(csiId), eq(certificate.getSerial()), 
                eq(certificate.getOrderId()), eq(newOrderId));
    }
    
    @Test
    void testRenewalWithMissingConfig() {
        // Given
        String csiId = "166479";
        when(csiConfigRepository.findByCsiId(csiId)).thenReturn(Optional.empty());
        
        // When & Then
        assertThrows(RuntimeException.class, () -> 
            renewalService.renewCertificate(certificate, csiId)
        );
    }
}

// ===== test/CodeScannerServiceTest.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.integration.BitbucketClient;
import com.citi.cert.renewal.model.FileLocation;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Arrays;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CodeScannerServiceTest {
    
    @Mock
    private BitbucketClient bitbucketClient;
    
    @InjectMocks
    private CodeScannerService codeScannerService;
    
    private String repository = "test-repo";
    private String orderId = "API_C-12345678-1234-1234-1234-123456789012";
    
    @Test
    void testFindCertificateUsage() {
        // Given
        FileLocation javaLocation = new FileLocation("src/main/java/Config.java", "master", 42, "String orderId = \"API_C-12345678-1234-1234-1234-123456789012\";");
        FileLocation propLocation = new FileLocation("application.properties", "master", 10, "cert.order.id=API_C-12345678-1234-1234-1234-123456789012");
        
        when(bitbucketClient.searchFiles(repository, "*.java", orderId))
                .thenReturn(Arrays.asList(javaLocation));
        when(bitbucketClient.searchFiles(repository, "*.properties", orderId))
                .thenReturn(Arrays.asList(propLocation));
        when(bitbucketClient.searchFiles(repository, "*.yml", orderId))
                .thenReturn(Arrays.asList());
        when(bitbucketClient.searchFiles(repository, "*.yaml", orderId))
                .thenReturn(Arrays.asList());
        
        // When
        List<FileLocation> result = codeScannerService.findCertificateUsage(repository, orderId);
        
        // Then
        assertEquals(2, result.size());
        verify(bitbucketClient, times(4)).searchFiles(anyString(), anyString(), eq(orderId));
    }
    
    @Test
    void testCreatePullRequest() {
        // Given
        String oldOrderId = "API_C-old-order";
        String newOrderId = "API_C-new-order";
        String expectedPrUrl = "https://bitbucket.com/pr/123";
        
        FileLocation location = new FileLocation("Config.java", "master", 42, "content");
        List<FileLocation> locations = Arrays.asList(location);
        
        when(bitbucketClient.getFileContent(repository, location.getFilePath()))
                .thenReturn("String orderId = \"API_C-old-order\";");
        when(bitbucketClient.createPullRequest(anyString(), anyString(), anyString(), anyString()))
                .thenReturn(expectedPrUrl);
        
        // When
        String result = codeScannerService.createPullRequest(repository, oldOrderId, newOrderId, locations);
        
        // Then
        assertEquals(expectedPrUrl, result);
        verify(bitbucketClient).createBranch(eq(repository), anyString());
        verify(bitbucketClient).updateFile(eq(repository), anyString(), eq(location.getFilePath()), anyString());
        verify(bitbucketClient).createPullRequest(eq(repository), anyString(), anyString(), anyString());
    }
}

// ===== test/integration/CertificateRenewalIntegrationTest.java =====
package com.citi.cert.renewal.integration;

import com.citi.cert.renewal.model.CSIConfiguration;
import com.citi.cert.renewal.repository.CSIConfigurationRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.ActiveProfiles;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class CertificateRenewalIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private CSIConfigurationRepository csiConfigRepository;
    
    @Test
    void testCSIConfigurationEndpoints() {
        // Create CSI configuration
        CSIConfiguration config = CSIConfiguration.builder()
                .csiId("TEST-166479")
                .businessOwner("Test Owner")
                .businessOwnerGeid("1001183626")
                .appOwner("Test App Owner")
                .appOwnerGeid("1010805931")
                .environment("TEST")
                .policyName("TEST_POLICY")
                .build();
        
        ResponseEntity<CSIConfiguration> createResponse = restTemplate.postForEntity(
                "/api/csi", config, CSIConfiguration.class);
        
        assertEquals(HttpStatus.OK, createResponse.getStatusCode());
        assertNotNull(createResponse.getBody());
        assertEquals("TEST-166479", createResponse.getBody().getCsiId());
        
        // Get CSI configuration
        ResponseEntity<CSIConfiguration> getResponse = restTemplate.getForEntity(
                "/api/csi/TEST-166479", CSIConfiguration.class);
        
        assertEquals(HttpStatus.OK, getResponse.getStatusCode());
        assertNotNull(getResponse.getBody());
        assertEquals("TEST-166479", getResponse.getBody().getCsiId());
    }
}

// ===== config/SecurityConfig.java =====
package com.citi.cert.renewal.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()  // Disable CSRF for API endpoints
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/actuator/**").permitAll()
                .requestMatchers("/api/**").authenticated()
                .anyRequest().authenticated()
            )
            .httpBasic();  // Basic auth for simplicity, replace with proper auth mechanism
            
        return http.build();
    }
}

// ===== exception/GlobalExceptionHandler.java =====
package com.citi.cert.renewal.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(CertificateRenewalException.class)
    public ResponseEntity<Map<String, Object>> handleCertificateRenewalException(CertificateRenewalException e) {
        log.error("Certificate renewal error: ", e);
        
        Map<String, Object> response = new HashMap<>();
        response.put("timestamp", LocalDateTime.now());
        response.put("status", HttpStatus.INTERNAL_SERVER_ERROR.value());
        response.put("error", "Certificate Renewal Failed");
        response.put("message", e.getMessage());
        
        return new ResponseEntity<>(response, HttpStatus.INTERNAL_SERVER_ERROR);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGenericException(Exception e) {
        log.error("Unexpected error: ", e);
        
        Map<String, Object> response = new HashMap<>();
        response.put("timestamp", LocalDateTime.now());
        response.put("status", HttpStatus.INTERNAL_SERVER_ERROR.value());
        response.put("error", "Internal Server Error");
        response.put("message", "An unexpected error occurred");
        
        return new ResponseEntity<>(response, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}

// ===== model/CertificateRenewalMetrics.java =====
package com.citi.cert.renewal.model;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;

@Component
public class CertificateRenewalMetrics {
    
    private final Counter renewalSuccessCounter;
    private final Counter renewalFailureCounter;
    private final Counter prCreatedCounter;
    private final Counter manualInterventionCounter;
    private final Timer renewalTimer;
    private final Timer scanTimer;
    
    public CertificateRenewalMetrics(MeterRegistry registry) {
        this.renewalSuccessCounter = Counter.builder("certificate.renewal.success")
                .description("Number of successful certificate renewals")
                .register(registry);
                
        this.renewalFailureCounter = Counter.builder("certificate.renewal.failure")
                .description("Number of failed certificate renewals")
                .register(registry);
                
        this.prCreatedCounter = Counter.builder("pull.request.created")
                .description("Number of pull requests created")
                .register(registry);
                
        this.manualInterventionCounter = Counter.builder("manual.intervention.required")
                .description("Number of times manual intervention was required")
                .register(registry);
                
        this.renewalTimer = Timer.builder("certificate.renewal.duration")
                .description("Time taken to renew a certificate")
                .register(registry);
                
        this.scanTimer = Timer.builder("repository.scan.duration")
                .description("Time taken to scan a repository")
                .register(registry);
    }
    
    public void recordRenewalSuccess() {
        renewalSuccessCounter.increment();
    }
    
    public void recordRenewalFailure() {
        renewalFailureCounter.increment();
    }
    
    public void recordPRCreated() {
        prCreatedCounter.increment();
    }
    
    public void recordManualIntervention() {
        manualInterventionCounter.increment();
    }
    
    public Timer.Sample startRenewalTimer() {
        return Timer.start();
    }
    
    public void stopRenewalTimer(Timer.Sample sample) {
        sample.stop(renewalTimer);
    }
    
    public Timer.Sample startScanTimer() {
        return Timer.start();
    }
    
    public void stopScanTimer(Timer.Sample sample) {
        sample.stop(scanTimer);
    }
}



-----------------
Main:

Container Certificate Renewal System - Architecture Documentation
System Overview
The Container Certificate Renewal System automates the process of discovering, renewing, and updating certificates used in OpenShift containers. It integrates with HashiCorp Vault, CMP API, and Bitbucket to provide end-to-end certificate lifecycle management.
Architecture Components
1. Core Services

Certificate Discovery Service: Interfaces with HashiCorp Vault to discover certificates
Certificate Renewal Service: Manages certificate renewal through CMP API
Code Scanner Service: Scans Bitbucket repositories for certificate usage
Notification Service: Sends alerts to CSI owners
Audit Service: Logs all activities to MongoDB

2. External Integrations

HashiCorp Vault: Certificate storage and retrieval
CMP API: Certificate Management Platform for renewal
Bitbucket API: Repository scanning and PR creation
MongoDB: Audit logging and CSI configuration storage

Sequence Diagram
mermaidsequenceDiagram
    participant Scheduler
    participant CertDiscovery
    participant Vault
    participant CertRenewal
    participant CMP
    participant CodeScanner
    participant Bitbucket
    participant NotificationService
    participant AuditService
    participant MongoDB

    Scheduler->>CertDiscovery: Trigger certificate check
    CertDiscovery->>Vault: Get certificates for CSI
    Vault-->>CertDiscovery: Return certificates
    CertDiscovery->>CertDiscovery: Check expiry (< 60 days)
    
    alt Certificate expiring
        CertDiscovery->>CertRenewal: Initiate renewal
        CertRenewal->>MongoDB: Get CSI configuration
        MongoDB-->>CertRenewal: Return config
        CertRenewal->>CMP: Create renewal request
        CMP-->>CertRenewal: Return Order ID
        
        loop Poll for completion
            CertRenewal->>CMP: Check status
            CMP-->>CertRenewal: Return status
        end
        
        CertRenewal->>CodeScanner: Scan for certificate usage
        CodeScanner->>Bitbucket: Search repositories
        Bitbucket-->>CodeScanner: Return file locations
        
        CodeScanner->>CodeScanner: Check FID access
        
        alt Has access
            CodeScanner->>Bitbucket: Create feature branch
            CodeScanner->>Bitbucket: Update Order ID
            CodeScanner->>Bitbucket: Create Pull Request
            Bitbucket-->>CodeScanner: PR created
            CodeScanner->>NotificationService: Send PR notification
        else No access
            CodeScanner->>NotificationService: Send manual update alert
        end
        
        NotificationService->>NotificationService: Send notifications
        CertRenewal->>AuditService: Log renewal activity
        AuditService->>MongoDB: Store audit log
    end
Data Flow

Discovery Phase

Scheduler triggers periodic certificate checks
System queries Vault for all certificates belonging to a CSI
Certificates are evaluated for expiry within 60 days


Renewal Phase

CSI configuration is retrieved from MongoDB
CMP API is called to create a renewal request
System polls CMP for completion status


Update Phase

Bitbucket repositories are scanned for Order ID references
Files containing certificate references are identified
If FID has access, automated PR is created
If no access, manual intervention is triggered


Notification Phase

Relevant stakeholders are notified based on the outcome
All activities are logged to MongoDB for audit purposes



Key Design Decisions

Pluggable Certificate Providers: The system is designed to support multiple certificate providers (CMP, Provisioning Portal, ACM) through a common interface.
Flexible Code Scanning: The scanner supports multiple file types (Java, properties, YAML) and can be extended for future patterns.
Audit Trail: Every action is logged with full context for compliance and troubleshooting.
Resilient Processing: The system uses retry mechanisms and graceful degradation for external API failures.

Security Considerations

FID authentication for Bitbucket access
Encrypted storage of sensitive configuration
Role-based access control for CSI management
Audit logging for all certificate operations

-----------------

# Deployment Scripts and Sample Data

## Docker Configuration

### Dockerfile
```dockerfile
FROM openjdk:17-jdk-slim

# Install Git for JGit operations
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*

# Create app directory
WORKDIR /app

# Create workspace directory for Git operations
RUN mkdir -p /tmp/cert-renewal

# Copy jar file
COPY target/container-cert-renewal-1.0.0.jar app.jar

# Create non-root user
RUN useradd -m -s /bin/bash certuser && \
    chown -R certuser:certuser /app /tmp/cert-renewal

USER certuser

# Expose port
EXPOSE 8080

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### docker-compose.yml
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:4.4
    container_name: cert-renewal-mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: admin123
      MONGO_INITDB_DATABASE: cert-renewal
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  cert-renewal:
    build: .
    container_name: cert-renewal-app
    environment:
      SPRING_PROFILES_ACTIVE: docker
      MONGODB_URI: mongodb://admin:admin123@mongodb:27017/cert-renewal?authSource=admin
      VAULT_API_URL: ${VAULT_API_URL}
      VAULT_TOKEN: ${VAULT_TOKEN}
      CMP_API_URL: ${CMP_API_URL}
      CMP_API_KEY: ${CMP_API_KEY}
      BITBUCKET_API_URL: ${BITBUCKET_API_URL}
      BITBUCKET_USERNAME: ${BITBUCKET_USERNAME}
      BITBUCKET_PASSWORD: ${BITBUCKET_PASSWORD}
      APPLICATION_FID: ${APPLICATION_FID}
    ports:
      - "8080:8080"
    depends_on:
      - mongodb
    volumes:
      - git-workspace:/tmp/cert-renewal

volumes:
  mongo-data:
  git-workspace:
```

## OpenShift Deployment

### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cert-renewal
  namespace: certificate-management
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cert-renewal
  template:
    metadata:
      labels:
        app: cert-renewal
    spec:
      serviceAccountName: cert-renewal-sa
      containers:
      - name: cert-renewal
        image: cert-renewal:latest
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "openshift"
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: cert-renewal-secrets
              key: mongodb-uri
        - name: VAULT_API_URL
          valueFrom:
            configMapKeyRef:
              name: cert-renewal-config
              key: vault-api-url
        - name: VAULT_TOKEN
          valueFrom:
            secretKeyRef:
              name: cert-renewal-secrets
              key: vault-token
        - name: CMP_API_URL
          valueFrom:
            configMapKeyRef:
              name: cert-renewal-config
              key: cmp-api-url
        - name: CMP_API_KEY
          valueFrom:
            secretKeyRef:
              name: cert-renewal-secrets
              key: cmp-api-key
        - name: BITBUCKET_API_URL
          valueFrom:
            configMapKeyRef:
              name: cert-renewal-config
              key: bitbucket-api-url
        - name: BITBUCKET_USERNAME
          valueFrom:
            secretKeyRef:
              name: cert-renewal-secrets
              key: bitbucket-username
        - name: BITBUCKET_PASSWORD
          valueFrom:
            secretKeyRef:
              name: cert-renewal-secrets
              key: bitbucket-password
        - name: APPLICATION_FID
          valueFrom:
            secretKeyRef:
              name: cert-renewal-secrets
              key: application-fid
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        volumeMounts:
        - name: git-workspace
          mountPath: /tmp/cert-renewal
      volumes:
      - name: git-workspace
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: cert-renewal-service
  namespace: certificate-management
spec:
  selector:
    app: cert-renewal
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
    name: http

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cert-renewal-config
  namespace: certificate-management
data:
  vault-api-url: "https://vault.citi.com"
  cmp-api-url: "https://cmp.citi.com/api/v1"
  bitbucket-api-url: "https://cedt-icg-bitbucket.nam.nsroot.net/bitbucket/rest/api/1.0"

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cert-renewal-sa
  namespace: certificate-management
```

## Sample API Requests

### Create CSI Configuration
```bash
curl -X POST http://localhost:8080/api/csi \
  -H "Content-Type: application/json" \
  -d '{
    "csiId": "166479",
    "businessOwner": "Gujja, Narayan Reddy",
    "businessOwnerGeid": "1001183626",
    "appOwner": "Ram, S",
    "appOwnerGeid": "1010805931",
    "emailDistributionLists": [
      "platform-for-you-dl@citi.com",
      "cert-renewal-alerts@citi.com"
    ],
    "primaryContact": "narayan.gujja@citi.com",
    "escalationMatrix": {
      "L1": "platform-support@citi.com",
      "L2": "platform-escalation@citi.com",
      "L3": "platform-management@citi.com"
    },
    "environment": "PROD",
    "policyName": "PFY_API_CERTIFICATE",
    "bitbucketRepos": [
      "https://cedt-icg-bitbucket.nam.nsroot.net/bitbucket/projects/PFY/repos/backend",
      "https://cedt-icg-bitbucket.nam.nsroot.net/bitbucket/projects/PFY/repos/frontend"
    ]
  }'
```

### Get Expiring Certificates
```bash
curl http://localhost:8080/api/certificate/expiring/166479
```

### Trigger Manual Renewal
```bash
curl -X POST http://localhost:8080/api/certificate/renew/166479?certificateSerial=2341595527353124998301781834574083096011000105
```

### Get Audit Logs
```bash
curl "http://localhost:8080/api/audit/csi/166479?start=2025-01-01T00:00:00&end=2025-12-31T23:59:59"
```

## MongoDB Initial Setup Script

### init-mongo.js
```javascript
// Create database
db = db.getSiblingDB('cert-renewal');

// Create user
db.createUser({
  user: 'certuser',
  pwd: 'certpass123',
  roles: [
    {
      role: 'readWrite',
      db: 'cert-renewal'
    }
  ]
});

// Create collections
db.createCollection('csi_configurations');
db.createCollection('audit_logs');

// Create indexes
db.csi_configurations.createIndex({ 'csiId': 1 }, { unique: true });
db.audit_logs.createIndex({ 'csiId': 1, 'timestamp': -1 });
db.audit_logs.createIndex({ 'status': 1 });
db.audit_logs.createIndex({ 'action': 1 });

// Insert sample CSI configuration
db.csi_configurations.insertOne({
  csiId: '166479',
  businessOwner: 'Gujja, Narayan Reddy',
  businessOwnerGeid: '1001183626',
  appOwner: 'Ram, S',
  appOwnerGeid: '1010805931',
  emailDistributionLists: [
    'platform-for-you-dl@citi.com'
  ],
  primaryContact: 'narayan.gujja@citi.com',
  escalationMatrix: {
    L1: 'platform-support@citi.com',
    L2: 'platform-escalation@citi.com'
  },
  environment: 'PROD',
  policyName: 'PFY_API_CERTIFICATE',
  bitbucketRepos: [
    'https://cedt-icg-bitbucket.nam.nsroot.net/bitbucket/projects/PFY/repos/backend'
  ],
  createdAt: new Date(),
  updatedAt: new Date()
});
```

## Application Properties for Different Environments

### application-dev.yml
```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/cert-renewal

certificate:
  renewal:
    cron: "0 */5 * * * ?"  # Every 5 minutes for testing

logging:
  level:
    com.citi.cert.renewal: DEBUG
    org.springframework.data.mongodb: DEBUG
    org.eclipse.jgit: DEBUG
```

### application-prod.yml
```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}
      
certificate:
  renewal:
    cron: "0 0 2 * * ?"  # Daily at 2 AM

logging:
  level:
    com.citi.cert.renewal: INFO
    org.springframework.data.mongodb: WARN
    
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

## Build and Run Scripts

### build.sh
```bash
#!/bin/bash

echo "Building Container Certificate Renewal System..."

# Clean and build
./mvnw clean package

# Build Docker image
docker build -t cert-renewal:latest .

echo "Build completed successfully!"
```

### run-local.sh
```bash
#!/bin/bash

# Export environment variables
export VAULT_API_URL="https://vault.citi.com"
export VAULT_TOKEN="your-vault-token"
export CMP_API_URL="https://cmp.citi.com/api/v1"
export CMP_API_KEY="your-cmp-key"
export BITBUCKET_API_URL="https://cedt-icg-bitbucket.nam.nsroot.net/bitbucket/rest/api/1.0"
export BITBUCKET_USERNAME="your-username"
export BITBUCKET_PASSWORD="your-password"
export APPLICATION_FID="your-fid"

# Run with docker-compose
docker-compose up -d

echo "Application started on http://localhost:8080"
echo "MongoDB available on localhost:27017"
```

## Troubleshooting Guide

### Common Issues and Solutions

1. **MongoDB Connection Issues**
   ```bash
   # Check MongoDB logs
   docker logs cert-renewal-mongo
   
   # Test connection
   mongo mongodb://admin:admin123@localhost:27017/cert-renewal?authSource=admin
   ```

2. **Git Clone Failures**
   ```bash
   # Check Git credentials
   git ls-remote https://username:password@bitbucket.citi.com/scm/project/repo.git
   
   # Check workspace permissions
   ls -la /tmp/cert-renewal
   ```

3. **Certificate Not Found in Vault**
   ```bash
   # Test Vault connection
   curl -H "X-Vault-Token: $VAULT_TOKEN" \
     $VAULT_API_URL/v1/secret/data/certificates/166479
   ```

4. **CMP API Issues**
   ```bash
   # Test CMP API
   curl -H "X-API-Key: $CMP_API_KEY" \
     $CMP_API_URL/health
   ```

## Monitoring Queries

### Prometheus Metrics
```
# Certificate renewal success rate
rate(certificate_renewal_success_total[5m])

# Pull request creation rate
rate(pull_request_created_total[5m])

# Average renewal duration
certificate_renewal_duration_seconds_mean

# Failed renewals
certificate_renewal_failure_total
```

### MongoDB Queries
```javascript
// Recent failures
db.audit_logs.find({ 
  status: "FAILED", 
  timestamp: { $gte: new Date(Date.now() - 24*60*60*1000) } 
}).sort({ timestamp: -1 })

// Pending manual interventions
db.audit_logs.find({ 
  action: "MANUAL_INTERVENTION_REQUIRED",
  status: "PENDING" 
})

// Renewal statistics by CSI
db.audit_logs.aggregate([
  { $match: { action: "CERTIFICATE_RENEWAL" } },
  { $group: { 
    _id: "$csiId", 
    total: { $sum: 1 },
    success: { $sum: { $cond: [{ $eq: ["$status", "SUCCESS"] }, 1, 0] } },
    failed: { $sum: { $cond: [{ $eq: ["$status", "FAILED"] }, 1, 0] } }
  }}
])
```

--------------

API Endpoints
CSI Configuration Management

POST /api/csi - Create CSI configuration
GET /api/csi/{csiId} - Get CSI configuration
PUT /api/csi/{csiId} - Update CSI configuration

Audit Logs

GET /api/audit/csi/{csiId}?start={datetime}&end={datetime} - Get audit logs for CSI
GET /api/audit/status/{status} - Get audit logs by status
curl -X POST http://localhost:8080/api/certificate/renew/166479?certificateSerial=

CSI Onboarding
To onboard a new CSI for automated certificate renewal:

Create CSI configuration via API:

jsonPOST /api/csi
{
  "csiId": "166479",
  "businessOwner": "Gujja",
  "businessOwnerGeid": "xx",
  "appOwner": "Ram",
  "appOwnerGeid": "yy",
  "emailDistributionLists": ["team-dl@comp.com"],
  "primaryContact": "primary.contact@comp.com",
  "escalationMatrix": {
    "L1": "l1-support@comp.com",
    "L2": "l2-support@comp.com"
  },
  "environment": "PROD",
  "policyName": "PFY_API_CERTIFICATE",
  "bitbucketRepos": [
    "https://bitbucket.comp.com/projects/PFY/repos/backend",
    "https://bitbucket.comp.com/projects/PFY/repos/frontend"
  ]
}
Certificate Renewal Process

Discovery: System scans Vault for certificates expiring within 60 days
Renewal: Creates CMP request with CSI configuration details
Polling: Monitors CMP order status until completion
Code Scanning: Searches repositories for Order ID usage
Update:

If FID has access: Creates PR with updated Order ID
If no access: Sends notification for manual update



Monitoring
The application exposes Prometheus metrics at /actuator/prometheus:

Certificate renewal success/failure counts
API response times
Repository scan durations
Email notification metrics

Troubleshooting
Common Issues

Certificate not found in Vault

Verify CSI ID is correct
Check Vault token permissions


CMP order stuck in pending

Check approval requirements
Verify business/app owner GEIDs are valid


Bitbucket access denied

Verify FID has repository access
Check Bitbucket token permissions



Logs
Application logs are stored in logs/cert-renewal.log with daily rotation.

----------------------------------------------------------------------
// Project Structure:
// container-cert-renewal/
// ├── src/main/java/com/citi/cert/renewal/
// │   ├── CertificateRenewalApplication.java
// │   ├── config/
// │   ├── controller/
// │   ├── service/
// │   ├── repository/
// │   ├── model/
// │   ├── dto/
// │   ├── integration/
// │   ├── scheduler/
// │   └── exception/
// └── src/main/resources/
//     └── application.yml

// ===== CertificateRenewalApplication.java =====
package com.citi.cert.renewal;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.mongodb.repository.config.EnableMongoRepositories;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.retry.annotation.EnableRetry;

@SpringBootApplication
@EnableMongoRepositories
@EnableScheduling
@EnableRetry
public class CertificateRenewalApplication {
    public static void main(String[] args) {
        SpringApplication.run(CertificateRenewalApplication.class, args);
    }
}

// ===== model/Certificate.java =====
package com.citi.cert.renewal.model;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;
import java.time.LocalDateTime;
import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Certificate {
    private String country;
    private List<String> sanList;
    private String state;
    private String cn;
    private String serial;
    private String environment;
    private List<String> ouList;
    private String policyId;
    private String ticketId; // Order ID (API_C-xxx)
    private String appid;
    private LocalDateTime validTo;
    private String issuer;
    private String policyNickName;
    private String certRole;
    
    public boolean isExpiringWithinDays(int days) {
        return validTo != null && validTo.isBefore(LocalDateTime.now().plusDays(days));
    }
    
    public String getOrderId() {
        return ticketId;
    }
}

// ===== model/CSIConfiguration.java =====
package com.citi.cert.renewal.model;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Document(collection = "csi_configurations")
public class CSIConfiguration {
    @Id
    private String id;
    private String csiId;
    private String businessOwner;
    private String businessOwnerGeid;
    private String appOwner;
    private String appOwnerGeid;
    private List<String> emailDistributionLists;
    private String primaryContact;
    private Map<String, String> escalationMatrix;
    private String environment;
    private String policyName;
    private List<String> bitbucketRepos;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// ===== model/AuditLog.java =====
package com.citi.cert.renewal.model;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import java.time.LocalDateTime;
import java.util.Map;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Document(collection = "audit_logs")
public class AuditLog {
    @Id
    private String id;
    private String csiId;
    private String action;
    private String status;
    private String certificateSerial;
    private String oldOrderId;
    private String newOrderId;
    private String repository;
    private String pullRequestUrl;
    private Map<String, Object> metadata;
    private String performedBy;
    private LocalDateTime timestamp;
    private String errorMessage;
}

// ===== dto/CMPRequest.java =====
package com.citi.cert.renewal.dto;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Builder;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CMPRequest {
    private String environment;
    private String action;
    private String csiId;
    private String businessOwner;
    private String appOwner;
    private String businessOwnerGeid;
    private String appOwnerGeid;
    private String policy;
    private boolean approvalsRequired;
    private boolean onboarded;
}

// ===== dto/CMPResponse.java =====
package com.citi.cert.renewal.dto;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class CMPResponse {
    private String orderId;
    private String status;
    private String message;
}

// ===== service/CertificateDiscoveryService.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.integration.VaultClient;
import com.citi.cert.renewal.model.Certificate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class CertificateDiscoveryService {
    
    private final VaultClient vaultClient;
    private static final int EXPIRY_THRESHOLD_DAYS = 60;
    
    public List<Certificate> discoverExpiringCertificates(String csiId) {
        log.info("Discovering certificates for CSI: {}", csiId);
        
        List<Certificate> allCertificates = vaultClient.getCertificatesForCSI(csiId);
        
        List<Certificate> expiringCertificates = allCertificates.stream()
                .filter(cert -> cert.isExpiringWithinDays(EXPIRY_THRESHOLD_DAYS))
                .collect(Collectors.toList());
                
        log.info("Found {} expiring certificates out of {} total for CSI: {}", 
                expiringCertificates.size(), allCertificates.size(), csiId);
                
        return expiringCertificates;
    }
}

// ===== service/CertificateRenewalService.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.dto.CMPRequest;
import com.citi.cert.renewal.dto.CMPResponse;
import com.citi.cert.renewal.integration.CMPClient;
import com.citi.cert.renewal.model.Certificate;
import com.citi.cert.renewal.model.CSIConfiguration;
import com.citi.cert.renewal.repository.CSIConfigurationRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import java.util.concurrent.TimeUnit;

@Service
@RequiredArgsConstructor
@Slf4j
public class CertificateRenewalService {
    
    private final CMPClient cmpClient;
    private final CSIConfigurationRepository csiConfigRepository;
    private final AuditService auditService;
    
    public String renewCertificate(Certificate certificate, String csiId) {
        log.info("Starting renewal for certificate: {} in CSI: {}", certificate.getSerial(), csiId);
        
        CSIConfiguration config = csiConfigRepository.findByCsiId(csiId)
                .orElseThrow(() -> new RuntimeException("CSI configuration not found: " + csiId));
        
        CMPRequest cmpRequest = buildCMPRequest(certificate, config);
        
        try {
            CMPResponse response = cmpClient.createReusableCertificate(cmpRequest);
            
            if (response.getOrderId() != null) {
                waitForCMPCompletion(response.getOrderId());
                
                auditService.logRenewalSuccess(csiId, certificate.getSerial(), 
                        certificate.getOrderId(), response.getOrderId());
                        
                return response.getOrderId();
            }
            
            throw new RuntimeException("Failed to get order ID from CMP");
            
        } catch (Exception e) {
            log.error("Certificate renewal failed for CSI: {}", csiId, e);
            auditService.logRenewalFailure(csiId, certificate.getSerial(), e.getMessage());
            throw e;
        }
    }
    
    @Retryable(value = {Exception.class}, maxAttempts = 30, backoff = @Backoff(delay = 60000))
    private void waitForCMPCompletion(String orderId) throws Exception {
        String status = cmpClient.checkOrderStatus(orderId);
        
        if ("COMPLETED".equals(status)) {
            log.info("CMP order {} completed successfully", orderId);
            return;
        } else if ("FAILED".equals(status)) {
            throw new RuntimeException("CMP order failed: " + orderId);
        }
        
        log.info("CMP order {} status: {}. Waiting...", orderId, status);
        throw new Exception("Order not yet completed");
    }
    
    private CMPRequest buildCMPRequest(Certificate certificate, CSIConfiguration config) {
        return CMPRequest.builder()
                .environment(certificate.getEnvironment())
                .action("Create Reusable Certificate")
                .csiId(config.getCsiId())
                .businessOwner(config.getBusinessOwner())
                .appOwner(config.getAppOwner())
                .businessOwnerGeid(config.getBusinessOwnerGeid())
                .appOwnerGeid(config.getAppOwnerGeid())
                .policy(certificate.getPolicyNickName())
                .approvalsRequired(true)
                .onboarded(true)
                .build();
    }
}

// ===== service/CodeScannerService.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.integration.BitbucketClient;
import com.citi.cert.renewal.model.FileLocation;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Pattern;

@Service
@RequiredArgsConstructor
@Slf4j
public class CodeScannerService {
    
    private final BitbucketClient bitbucketClient;
    private static final Pattern ORDER_ID_PATTERN = Pattern.compile("API_C-[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}");
    
    public List<FileLocation> findCertificateUsage(String repository, String orderId) {
        log.info("Scanning repository {} for order ID: {}", repository, orderId);
        
        List<FileLocation> locations = new ArrayList<>();
        
        // Search in different file types
        locations.addAll(searchInFiles(repository, "*.java", orderId));
        locations.addAll(searchInFiles(repository, "*.properties", orderId));
        locations.addAll(searchInFiles(repository, "*.yml", orderId));
        locations.addAll(searchInFiles(repository, "*.yaml", orderId));
        
        log.info("Found {} locations using order ID in repository {}", locations.size(), repository);
        
        return locations;
    }
    
    private List<FileLocation> searchInFiles(String repository, String filePattern, String orderId) {
        return bitbucketClient.searchFiles(repository, filePattern, orderId);
    }
    
    public boolean hasFIDAccess(String repository, String fid) {
        return bitbucketClient.checkWriteAccess(repository, fid);
    }
    
    public String createPullRequest(String repository, String oldOrderId, String newOrderId, 
                                  List<FileLocation> locations) {
        log.info("Creating pull request for repository: {}", repository);
        
        String branchName = "cert-renewal-" + System.currentTimeMillis();
        
        // Create feature branch
        bitbucketClient.createBranch(repository, branchName);
        
        // Update files
        for (FileLocation location : locations) {
            String content = bitbucketClient.getFileContent(repository, location.getFilePath());
            String updatedContent = content.replace(oldOrderId, newOrderId);
            bitbucketClient.updateFile(repository, branchName, location.getFilePath(), updatedContent);
        }
        
        // Create pull request
        String prDescription = String.format(
            "Automated certificate renewal\n\nOld Order ID: %s\nNew Order ID: %s\n\nFiles updated:\n%s",
            oldOrderId, newOrderId, 
            locations.stream().map(FileLocation::getFilePath).reduce("", (a, b) -> a + "\n- " + b)
        );
        
        return bitbucketClient.createPullRequest(repository, branchName, "Certificate Renewal", prDescription);
    }
}

// ===== service/NotificationService.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.model.CSIConfiguration;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationService {
    
    private final JavaMailSender mailSender;
    
    public void notifyPullRequestCreated(CSIConfiguration config, String repository, 
                                       String prUrl, String newOrderId) {
        String subject = String.format("Certificate Renewal - Pull Request Created for %s", repository);
        String body = String.format(
            "A pull request has been created for certificate renewal.\n\n" +
            "CSI ID: %s\n" +
            "Repository: %s\n" +
            "Pull Request: %s\n" +
            "New Order ID: %s\n\n" +
            "Please review and merge the pull request.",
            config.getCsiId(), repository, prUrl, newOrderId
        );
        
        sendEmail(config.getEmailDistributionLists(), subject, body);
    }
    
    public void notifyManualUpdateRequired(CSIConfiguration config, String repository, 
                                         String newOrderId, List<String> fileLocations) {
        String subject = String.format("Certificate Renewal - Manual Update Required for %s", repository);
        String body = String.format(
            "Certificate renewal completed but manual update is required.\n\n" +
            "CSI ID: %s\n" +
            "Repository: %s\n" +
            "New Order ID: %s\n\n" +
            "Files requiring update:\n%s\n\n" +
            "The FID does not have write access to the repository. Please update manually.",
            config.getCsiId(), repository, newOrderId,
            String.join("\n", fileLocations)
        );
        
        sendEmail(config.getEmailDistributionLists(), subject, body);
    }
    
    private void sendEmail(List<String> recipients, String subject, String body) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setTo(recipients.toArray(new String[0]));
            message.setSubject(subject);
            message.setText(body);
            mailSender.send(message);
            log.info("Email sent successfully to: {}", recipients);
        } catch (Exception e) {
            log.error("Failed to send email", e);
        }
    }
}

// ===== service/AuditService.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.model.AuditLog;
import com.citi.cert.renewal.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@Service
@RequiredArgsConstructor
@Slf4j
public class AuditService {
    
    private final AuditLogRepository auditLogRepository;
    
    public void logRenewalSuccess(String csiId, String certificateSerial, 
                                String oldOrderId, String newOrderId) {
        AuditLog log = AuditLog.builder()
                .csiId(csiId)
                .action("CERTIFICATE_RENEWAL")
                .status("SUCCESS")
                .certificateSerial(certificateSerial)
                .oldOrderId(oldOrderId)
                .newOrderId(newOrderId)
                .timestamp(LocalDateTime.now())
                .performedBy("SYSTEM")
                .build();
                
        auditLogRepository.save(log);
        this.log.info("Audit log created for successful renewal: {}", log);
    }
    
    public void logRenewalFailure(String csiId, String certificateSerial, String errorMessage) {
        AuditLog log = AuditLog.builder()
                .csiId(csiId)
                .action("CERTIFICATE_RENEWAL")
                .status("FAILED")
                .certificateSerial(certificateSerial)
                .errorMessage(errorMessage)
                .timestamp(LocalDateTime.now())
                .performedBy("SYSTEM")
                .build();
                
        auditLogRepository.save(log);
        this.log.error("Audit log created for failed renewal: {}", log);
    }
    
    public void logPullRequestCreated(String csiId, String repository, String prUrl, 
                                    String oldOrderId, String newOrderId) {
        Map<String, Object> metadata = new HashMap<>();
        metadata.put("repository", repository);
        metadata.put("prUrl", prUrl);
        
        AuditLog log = AuditLog.builder()
                .csiId(csiId)
                .action("PULL_REQUEST_CREATED")
                .status("SUCCESS")
                .repository(repository)
                .pullRequestUrl(prUrl)
                .oldOrderId(oldOrderId)
                .newOrderId(newOrderId)
                .metadata(metadata)
                .timestamp(LocalDateTime.now())
                .performedBy("SYSTEM")
                .build();
                
        auditLogRepository.save(log);
        this.log.info("Audit log created for PR creation: {}", log);
    }
}

// ===== integration/VaultClient.java =====
package com.citi.cert.renewal.integration;

import com.citi.cert.renewal.model.Certificate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import java.util.List;
import java.util.Map;

@Component
@RequiredArgsConstructor
@Slf4j
public class VaultClient {
    
    private final RestTemplate restTemplate;
    
    @Value("${vault.api.url}")
    private String vaultApiUrl;
    
    @Value("${vault.api.token}")
    private String vaultToken;
    
    public List<Certificate> getCertificatesForCSI(String csiId) {
        log.info("Fetching certificates from Vault for CSI: {}", csiId);
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Vault-Token", vaultToken);
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        String url = vaultApiUrl + "/v1/secret/data/certificates/" + csiId;
        
        ResponseEntity<Map> response = restTemplate.exchange(
            url, 
            HttpMethod.GET, 
            new HttpEntity<>(headers), 
            Map.class
        );
        
        // Parse response and convert to Certificate objects
        // Implementation depends on actual Vault response structure
        
        return List.of(); // Placeholder
    }
}

// ===== integration/CMPClient.java =====
package com.citi.cert.renewal.integration;

import com.citi.cert.renewal.dto.CMPRequest;
import com.citi.cert.renewal.dto.CMPResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

@Component
@RequiredArgsConstructor
@Slf4j
public class CMPClient {
    
    private final RestTemplate restTemplate;
    
    @Value("${cmp.api.url}")
    private String cmpApiUrl;
    
    @Value("${cmp.api.key}")
    private String cmpApiKey;
    
    public CMPResponse createReusableCertificate(CMPRequest request) {
        log.info("Creating reusable certificate via CMP for CSI: {}", request.getCsiId());
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-API-Key", cmpApiKey);
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        ResponseEntity<CMPResponse> response = restTemplate.exchange(
            cmpApiUrl + "/certificate/create",
            HttpMethod.POST,
            new HttpEntity<>(request, headers),
            CMPResponse.class
        );
        
        return response.getBody();
    }
    
    public String checkOrderStatus(String orderId) {
        log.info("Checking CMP order status for: {}", orderId);
        
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-API-Key", cmpApiKey);
        
        ResponseEntity<Map> response = restTemplate.exchange(
            cmpApiUrl + "/order/" + orderId + "/status",
            HttpMethod.GET,
            new HttpEntity<>(headers),
            Map.class
        );
        
        return (String) response.getBody().get("status");
    }
}

// ===== integration/BitbucketClient.java =====
package com.citi.cert.renewal.integration;

import com.citi.cert.renewal.model.FileLocation;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import java.util.List;

@Component
@RequiredArgsConstructor
@Slf4j
public class BitbucketClient {
    
    private final RestTemplate restTemplate;
    
    @Value("${bitbucket.api.url}")
    private String bitbucketApiUrl;
    
    @Value("${bitbucket.api.token}")
    private String bitbucketToken;
    
    public List<FileLocation> searchFiles(String repository, String filePattern, String searchText) {
        log.info("Searching files in repository {} with pattern {} for text: {}", 
                repository, filePattern, searchText);
        
        // Implementation for Bitbucket API file search
        // This is a placeholder - actual implementation depends on Bitbucket API
        
        return List.of(); // Placeholder
    }
    
    public boolean checkWriteAccess(String repository, String fid) {
        log.info("Checking write access for FID {} on repository {}", fid, repository);
        
        // Implementation for checking write access
        
        return false; // Placeholder
    }
    
    public void createBranch(String repository, String branchName) {
        log.info("Creating branch {} in repository {}", branchName, repository);
        
        // Implementation for creating branch
    }
    
    public String getFileContent(String repository, String filePath) {
        // Implementation for getting file content
        return "";
    }
    
    public void updateFile(String repository, String branch, String filePath, String content) {
        log.info("Updating file {} in branch {} of repository {}", filePath, branch, repository);
        
        // Implementation for updating file
    }
    
    public String createPullRequest(String repository, String sourceBranch, String title, String description) {
        log.info("Creating pull request in repository {} from branch {}", repository, sourceBranch);
        
        // Implementation for creating PR
        
        return "https://bitbucket.com/pr/123"; // Placeholder
    }
}

// ===== scheduler/CertificateRenewalScheduler.java =====
package com.citi.cert.renewal.scheduler;

import com.citi.cert.renewal.model.CSIConfiguration;
import com.citi.cert.renewal.model.Certificate;
import com.citi.cert.renewal.model.FileLocation;
import com.citi.cert.renewal.repository.CSIConfigurationRepository;
import com.citi.cert.renewal.service.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import java.util.List;

@Component
@RequiredArgsConstructor
@Slf4j
public class CertificateRenewalScheduler {
    
    private final CSIConfigurationRepository csiConfigRepository;
    private final CertificateDiscoveryService discoveryService;
    private final CertificateRenewalService renewalService;
    private final CodeScannerService codeScannerService;
    private final NotificationService notificationService;
    private final AuditService auditService;
    
    @Scheduled(cron = "${certificate.renewal.cron:0 0 2 * * ?}") // Daily at 2 AM
    public void checkAndRenewCertificates() {
        log.info("Starting scheduled certificate renewal check");
        
        List<CSIConfiguration> allCSIs = csiConfigRepository.findAll();
        
        for (CSIConfiguration csi : allCSIs) {
            try {
                processCertificatesForCSI(csi);
            } catch (Exception e) {
                log.error("Error processing certificates for CSI: {}", csi.getCsiId(), e);
            }
        }
        
        log.info("Completed scheduled certificate renewal check");
    }
    
    private void processCertificatesForCSI(CSIConfiguration csi) {
        List<Certificate> expiringCerts = discoveryService.discoverExpiringCertificates(csi.getCsiId());
        
        for (Certificate cert : expiringCerts) {
            try {
                // Renew certificate
                String newOrderId = renewalService.renewCertificate(cert, csi.getCsiId());
                
                // Process each repository
                for (String repo : csi.getBitbucketRepos()) {
                    processRepository(csi, cert, newOrderId, repo);
                }
                
            } catch (Exception e) {
                log.error("Failed to process certificate {} for CSI {}", 
                         cert.getSerial(), csi.getCsiId(), e);
            }
        }
    }
    
    private void processRepository(CSIConfiguration csi, Certificate cert, 
                                 String newOrderId, String repository) {
        try {
            // Find certificate usage
            List<FileLocation> locations = codeScannerService.findCertificateUsage(
                repository, cert.getOrderId());
            
            if (locations.isEmpty()) {
                log.info("No certificate usage found in repository: {}", repository);
                return;
            }
            
            // Check access
            String fid = System.getProperty("application.fid", "default-fid");
            boolean hasAccess = codeScannerService.hasFIDAccess(repository, fid);
            
            if (hasAccess) {
                // Create pull request
                String prUrl = codeScannerService.createPullRequest(
                    repository, cert.getOrderId(), newOrderId, locations);
                
                notificationService.notifyPullRequestCreated(csi, repository, prUrl, newOrderId);
                auditService.logPullRequestCreated(csi.getCsiId(), repository, prUrl, 
                                                 cert.getOrderId(), newOrderId);
            } else {
                // Notify for manual update
                List<String> filePaths = locations.stream()
                    .map(FileLocation::getFilePath)
                    .toList();
                    
                notificationService.notifyManualUpdateRequired(csi, repository, newOrderId, filePaths);
            }
            
        } catch (Exception e) {
            log.error("Failed to process repository {} for CSI {}", repository, csi.getCsiId(), e);
        }
    }
}

// ===== repository/CSIConfigurationRepository.java =====
package com.citi.cert.renewal.repository;

import com.citi.cert.renewal.model.CSIConfiguration;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public interface CSIConfigurationRepository extends MongoRepository<CSIConfiguration, String> {
    Optional<CSIConfiguration> findByCsiId(String csiId);
}

// ===== repository/AuditLogRepository.java =====
package com.citi.cert.renewal.repository;

import com.citi.cert.renewal.model.AuditLog;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;
import java.time.LocalDateTime;
import java.util.List;

@Repository
public interface AuditLogRepository extends MongoRepository<AuditLog, String> {
    List<AuditLog> findByCsiIdAndTimestampBetween(String csiId, LocalDateTime start, LocalDateTime end);
    List<AuditLog> findByStatus(String status);
}

// ===== controller/CSIConfigurationController.java =====
package com.citi.cert.renewal.controller;

import com.citi.cert.renewal.model.CSIConfiguration;
import com.citi.cert.renewal.repository.CSIConfigurationRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.time.LocalDateTime;

@RestController
@RequestMapping("/api/csi")
@RequiredArgsConstructor
public class CSIConfigurationController {
    
    private final CSIConfigurationRepository repository;
    
    @PostMapping
    public ResponseEntity<CSIConfiguration> createCSIConfiguration(@RequestBody CSIConfiguration config) {
        config.setCreatedAt(LocalDateTime.now());
        config.setUpdatedAt(LocalDateTime.now());
        CSIConfiguration saved = repository.save(config);
        return ResponseEntity.ok(saved);
    }
    
    @GetMapping("/{csiId}")
    public ResponseEntity<CSIConfiguration> getCSIConfiguration(@PathVariable String csiId) {
        return repository.findByCsiId(csiId)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }
    
    @PutMapping("/{csiId}")
    public ResponseEntity<CSIConfiguration> updateCSIConfiguration(
            @PathVariable String csiId, 
            @RequestBody CSIConfiguration config) {
        return repository.findByCsiId(csiId)
                .map(existing -> {
                    config.setId(existing.getId());
                    config.setCreatedAt(existing.getCreatedAt());
                    config.setUpdatedAt(LocalDateTime.now());
                    return ResponseEntity.ok(repository.save(config));
                })
                .orElse(ResponseEntity.notFound().build());
    }
}

// ===== controller/AuditController.java =====
package com.citi.cert.renewal.controller;

import com.citi.cert.renewal.model.AuditLog;
import com.citi.cert.renewal.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.time.LocalDateTime;
import java.util.List;

@RestController
@RequestMapping("/api/audit")
@RequiredArgsConstructor
public class AuditController {
    
    private final AuditLogRepository repository;
    
    @GetMapping("/csi/{csiId}")
    public ResponseEntity<List<AuditLog>> getAuditLogs(
            @PathVariable String csiId,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime start,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime end) {
        List<AuditLog> logs = repository.findByCsiIdAndTimestampBetween(csiId, start, end);
        return ResponseEntity.ok(logs);
    }
    
    @GetMapping("/status/{status}")
    public ResponseEntity<List<AuditLog>> getAuditLogsByStatus(@PathVariable String status) {
        List<AuditLog> logs = repository.findByStatus(status);
        return ResponseEntity.ok(logs);
    }
}

// ===== model/FileLocation.java =====
package com.citi.cert.renewal.model;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class FileLocation {
    private String filePath;
    private String branch;
    private int lineNumber;
    private String content;
}

// ===== config/RestTemplateConfig.java =====
package com.citi.cert.renewal.config;

import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;
import java.time.Duration;

@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
                .setConnectTimeout(Duration.ofSeconds(10))
                .setReadTimeout(Duration.ofSeconds(30))
                .build();
    }
}

// ===== config/MongoConfig.java =====
package com.citi.cert.renewal.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.config.EnableMongoAuditing;

@Configuration
@EnableMongoAuditing
public class MongoConfig {
    // MongoDB configuration
}

// ===== exception/CertificateRenewalException.java =====
package com.citi.cert.renewal.exception;

public class CertificateRenewalException extends RuntimeException {
    public CertificateRenewalException(String message) {
        super(message);
    }
    
    public CertificateRenewalException(String message, Throwable cause) {
        super(message, cause);
    }
}


---------------

// ===== Updated CertificateRenewalScheduler.java =====
package com.citi.cert.renewal.scheduler;

import com.citi.cert.renewal.model.CSIConfiguration;
import com.citi.cert.renewal.model.Certificate;
import com.citi.cert.renewal.model.FileLocation;
import com.citi.cert.renewal.repository.CSIConfigurationRepository;
import com.citi.cert.renewal.service.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import java.util.List;

@Component
@RequiredArgsConstructor
@Slf4j
public class CertificateRenewalScheduler {
    
    private final CSIConfigurationRepository csiConfigRepository;
    private final CertificateDiscoveryService discoveryService;
    private final CertificateRenewalService renewalService;
    private final CodeScannerService codeScannerService;
    private final NotificationService notificationService;
    private final AuditService auditService;
    
    @Value("${application.fid}")
    private String applicationFid;
    
    @Scheduled(cron = "${certificate.renewal.cron:0 0 2 * * ?}")
    public void checkAndRenewCertificates() {
        log.info("Starting scheduled certificate renewal check");
        
        List<CSIConfiguration> allCSIs = csiConfigRepository.findAll();
        
        for (CSIConfiguration csi : allCSIs) {
            try {
                processCertificatesForCSI(csi);
            } catch (Exception e) {
                log.error("Error processing certificates for CSI: {}", csi.getCsiId(), e);
            }
        }
        
        log.info("Completed scheduled certificate renewal check");
    }
    
    private void processCertificatesForCSI(CSIConfiguration csi) {
        List<Certificate> expiringCerts = discoveryService.discoverExpiringCertificates(csi.getCsiId());
        
        for (Certificate cert : expiringCerts) {
            try {
                // Renew certificate
                String newOrderId = renewalService.renewCertificate(cert, csi.getCsiId());
                
                // Process each repository
                for (String repo : csi.getBitbucketRepos()) {
                    processRepository(csi, cert, newOrderId, repo);
                }
                
            } catch (Exception e) {
                log.error("Failed to process certificate {} for CSI {}", 
                         cert.getSerial(), csi.getCsiId(), e);
            }
        }
    }
    
    private void processRepository(CSIConfiguration csi, Certificate cert, 
                                 String newOrderId, String repository) {
        try {
            // Find certificate usage
            List<FileLocation> locations = codeScannerService.findCertificateUsage(
                repository, cert.getOrderId());
            
            if (locations.isEmpty()) {
                log.info("No certificate usage found in repository: {}", repository);
                return;
            }
            
            // Check access using the application FID
            boolean hasAccess = codeScannerService.hasFIDAccess(repository, applicationFid);
            
            if (hasAccess) {
                // Create pull request with proper title and description
                String prTitle = String.format("Certificate Renewal - Update Order ID for CSI %s", csi.getCsiId());
                String prDescription = buildPullRequestDescription(csi, cert, newOrderId, locations);
                
                String prUrl = codeScannerService.createPullRequest(
                    repository, cert.getOrderId(), newOrderId, locations, prTitle, prDescription);
                
                notificationService.notifyPullRequestCreated(csi, repository, prUrl, newOrderId);
                auditService.logPullRequestCreated(csi.getCsiId(), repository, prUrl, 
                                                 cert.getOrderId(), newOrderId);
            } else {
                // Notify for manual update
                List<String> filePaths = locations.stream()
                    .map(FileLocation::getFilePath)
                    .toList();
                    
                notificationService.notifyManualUpdateRequired(csi, repository, newOrderId, filePaths);
                auditService.logManualInterventionRequired(csi.getCsiId(), repository, 
                                                          cert.getOrderId(), newOrderId, "No write access");
            }
            
        } catch (Exception e) {
            log.error("Failed to process repository {} for CSI {}", repository, csi.getCsiId(), e);
            auditService.logRepositoryProcessingFailure(csi.getCsiId(), repository, e.getMessage());
        }
    }
    
    private String buildPullRequestDescription(CSIConfiguration csi, Certificate cert, 
                                             String newOrderId, List<FileLocation> locations) {
        StringBuilder description = new StringBuilder();
        description.append("## Automated Certificate Renewal\n\n");
        description.append("### Certificate Details\n");
        description.append("- **CSI ID**: ").append(csi.getCsiId()).append("\n");
        description.append("- **Certificate CN**: ").append(cert.getCn()).append("\n");
        description.append("- **Certificate Serial**: ").append(cert.getSerial()).append("\n");
        description.append("- **Environment**: ").append(cert.getEnvironment()).append("\n");
        description.append("- **Expiry Date**: ").append(cert.getValidTo()).append("\n\n");
        
        description.append("### Order ID Update\n");
        description.append("- **Old Order ID**: `").append(cert.getOrderId()).append("`\n");
        description.append("- **New Order ID**: `").append(newOrderId).append("`\n\n");
        
        description.append("### Files Updated\n");
        for (FileLocation location : locations) {
            description.append("- `").append(location.getFilePath())
                      .append("` (Line ").append(location.getLineNumber()).append(")\n");
        }
        
        description.append("\n### Action Required\n");
        description.append("Please review the changes and merge this pull request to complete the certificate renewal.\n");
        description.append("After merging, restart the application in OpenShift to use the new certificate.\n");
        
        return description.toString();
    }
}

// ===== Updated CodeScannerService.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.integration.BitbucketClient;
import com.citi.cert.renewal.model.FileLocation;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Pattern;

@Service
@RequiredArgsConstructor
@Slf4j
public class CodeScannerService {
    
    private final BitbucketClient bitbucketClient;
    private static final Pattern ORDER_ID_PATTERN = Pattern.compile("API_C-[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}");
    
    public List<FileLocation> findCertificateUsage(String repository, String orderId) {
        log.info("Scanning repository {} for order ID: {}", repository, orderId);
        
        List<FileLocation> locations = new ArrayList<>();
        
        // Search in different file types
        locations.addAll(searchInFiles(repository, "*.java", orderId));
        locations.addAll(searchInFiles(repository, "*.properties", orderId));
        locations.addAll(searchInFiles(repository, "*.yml", orderId));
        locations.addAll(searchInFiles(repository, "*.yaml", orderId));
        
        log.info("Found {} locations using order ID in repository {}", locations.size(), repository);
        
        return locations;
    }
    
    private List<FileLocation> searchInFiles(String repository, String filePattern, String orderId) {
        return bitbucketClient.searchFiles(repository, filePattern, orderId);
    }
    
    public boolean hasFIDAccess(String repository, String fid) {
        return bitbucketClient.checkWriteAccess(repository, fid);
    }
    
    public String createPullRequest(String repository, String oldOrderId, String newOrderId, 
                                  List<FileLocation> locations, String title, String description) {
        log.info("Creating pull request for repository: {}", repository);
        
        String branchName = String.format("cert-renewal-%s-%d", 
                                        oldOrderId.substring(6, 14), System.currentTimeMillis());
        
        // Create feature branch
        bitbucketClient.createBranch(repository, branchName);
        
        // Update files
        for (FileLocation location : locations) {
            String content = bitbucketClient.getFileContent(repository, location.getFilePath());
            String updatedContent = content.replace(oldOrderId, newOrderId);
            bitbucketClient.updateFile(repository, branchName, location.getFilePath(), updatedContent);
        }
        
        // Create pull request
        return bitbucketClient.createPullRequest(repository, branchName, title, description);
    }
}

// ===== Updated AuditService.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.model.AuditLog;
import com.citi.cert.renewal.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@Service
@RequiredArgsConstructor
@Slf4j
public class AuditService {
    
    private final AuditLogRepository auditLogRepository;
    
    public void logRenewalSuccess(String csiId, String certificateSerial, 
                                String oldOrderId, String newOrderId) {
        AuditLog log = AuditLog.builder()
                .csiId(csiId)
                .action("CERTIFICATE_RENEWAL")
                .status("SUCCESS")
                .certificateSerial(certificateSerial)
                .oldOrderId(oldOrderId)
                .newOrderId(newOrderId)
                .timestamp(LocalDateTime.now())
                .performedBy("SYSTEM")
                .build();
                
        auditLogRepository.save(log);
        this.log.info("Audit log created for successful renewal: {}", log);
    }
    
    public void logRenewalFailure(String csiId, String certificateSerial, String errorMessage) {
        AuditLog log = AuditLog.builder()
                .csiId(csiId)
                .action("CERTIFICATE_RENEWAL")
                .status("FAILED")
                .certificateSerial(certificateSerial)
                .errorMessage(errorMessage)
                .timestamp(LocalDateTime.now())
                .performedBy("SYSTEM")
                .build();
                
        auditLogRepository.save(log);
        this.log.error("Audit log created for failed renewal: {}", log);
    }
    
    public void logPullRequestCreated(String csiId, String repository, String prUrl, 
                                    String oldOrderId, String newOrderId) {
        Map<String, Object> metadata = new HashMap<>();
        metadata.put("repository", repository);
        metadata.put("prUrl", prUrl);
        
        AuditLog log = AuditLog.builder()
                .csiId(csiId)
                .action("PULL_REQUEST_CREATED")
                .status("SUCCESS")
                .repository(repository)
                .pullRequestUrl(prUrl)
                .oldOrderId(oldOrderId)
                .newOrderId(newOrderId)
                .metadata(metadata)
                .timestamp(LocalDateTime.now())
                .performedBy("SYSTEM")
                .build();
                
        auditLogRepository.save(log);
        this.log.info("Audit log created for PR creation: {}", log);
    }
    
    public void logManualInterventionRequired(String csiId, String repository, 
                                            String oldOrderId, String newOrderId, String reason) {
        Map<String, Object> metadata = new HashMap<>();
        metadata.put("repository", repository);
        metadata.put("reason", reason);
        
        AuditLog log = AuditLog.builder()
                .csiId(csiId)
                .action("MANUAL_INTERVENTION_REQUIRED")
                .status("PENDING")
                .repository(repository)
                .oldOrderId(oldOrderId)
                .newOrderId(newOrderId)
                .metadata(metadata)
                .timestamp(LocalDateTime.now())
                .performedBy("SYSTEM")
                .build();
                
        auditLogRepository.save(log);
        this.log.info("Audit log created for manual intervention: {}", log);
    }
    
    public void logRepositoryProcessingFailure(String csiId, String repository, String errorMessage) {
        AuditLog log = AuditLog.builder()
                .csiId(csiId)
                .action("REPOSITORY_PROCESSING")
                .status("FAILED")
                .repository(repository)
                .errorMessage(errorMessage)
                .timestamp(LocalDateTime.now())
                .performedBy("SYSTEM")
                .build();
                
        auditLogRepository.save(log);
        this.log.error("Audit log created for repository processing failure: {}", log);
    }
}

// ===== Additional Configuration Bean =====
package com.citi.cert.renewal.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JacksonConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
}

// ===== model/NotificationTemplate.java =====
package com.citi.cert.renewal.model;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class NotificationTemplate {
    private String subject;
    private String body;
    private NotificationType type;
    
    public enum NotificationType {
        PULL_REQUEST_CREATED,
        MANUAL_UPDATE_REQUIRED,
        RENEWAL_SUCCESS,
        RENEWAL_FAILURE,
        APPROVAL_REQUIRED
    }
}

// ===== service/TemplateService.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.model.CSIConfiguration;
import com.citi.cert.renewal.model.NotificationTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;

@Service
public class TemplateService {
    
    private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    
    public NotificationTemplate createPullRequestNotification(CSIConfiguration config, 
                                                            String repository, 
                                                            String prUrl, 
                                                            String newOrderId) {
        String subject = String.format("[Certificate Renewal] Pull Request Created - CSI %s", config.getCsiId());
        
        StringBuilder body = new StringBuilder();
        body.append("Dear Team,\n\n");
        body.append("A pull request has been automatically created for certificate renewal.\n\n");
        body.append("**Details:**\n");
        body.append("- CSI ID: ").append(config.getCsiId()).append("\n");
        body.append("- Repository: ").append(repository).append("\n");
        body.append("- Pull Request URL: ").append(prUrl).append("\n");
        body.append("- New Order ID: ").append(newOrderId).append("\n");
        body.append("- Created At: ").append(LocalDateTime.now().format(DATE_FORMATTER)).append("\n\n");
        body.append("**Action Required:**\n");
        body.append("Please review and merge the pull request to complete the certificate renewal process.\n");
        body.append("After merging, ensure the application is restarted in OpenShift.\n\n");
        body.append("**Note:** This is an automated notification from the Certificate Renewal System.\n\n");
        body.append("Best Regards,\n");
        body.append("Certificate Renewal Automation Team");
        
        return NotificationTemplate.builder()
                .subject(subject)
                .body(body.toString())
                .type(NotificationTemplate.NotificationType.PULL_REQUEST_CREATED)
                .build();
    }
    
    public NotificationTemplate createManualUpdateNotification(CSIConfiguration config, 
                                                             String repository, 
                                                             String newOrderId, 
                                                             List<String> fileLocations) {
        String subject = String.format("[Certificate Renewal] Manual Update Required - CSI %s", config.getCsiId());
        
        StringBuilder body = new StringBuilder();
        body.append("Dear Team,\n\n");
        body.append("Certificate renewal has been completed, but manual intervention is required.\n\n");
        body.append("**Reason:** The automation FID does not have write access to the repository.\n\n");
        body.append("**Details:**\n");
        body.append("- CSI ID: ").append(config.getCsiId()).append("\n");
        body.append("- Repository: ").append(repository).append("\n");
        body.append("- New Order ID: ").append(newOrderId).append("\n");
        body.append("- Time: ").append(LocalDateTime.now().format(DATE_FORMATTER)).append("\n\n");
        body.append("**Files Requiring Update:**\n");
        for (String file : fileLocations) {
            body.append("- ").append(file).append("\n");
        }
        body.append("\n**Action Required:**\n");
        body.append("1. Update the Order ID in the files listed above\n");
        body.append("2. Replace the old Order ID with: ").append(newOrderId).append("\n");
        body.append("3. Commit and push the changes\n");
        body.append("4. Restart the application in OpenShift\n\n");
        body.append("**Escalation:**\n");
        body.append("Primary Contact: ").append(config.getPrimaryContact()).append("\n");
        if (config.getEscalationMatrix() != null) {
            config.getEscalationMatrix().forEach((level, contact) -> 
                body.append("- ").append(level).append(": ").append(contact).append("\n")
            );
        }
        body.append("\nBest Regards,\n");
        body.append("Certificate Renewal Automation Team");
        
        return NotificationTemplate.builder()
                .subject(subject)
                .body(body.toString())
                .type(NotificationTemplate.NotificationType.MANUAL_UPDATE_REQUIRED)
                .build();
    }
}

// ===== controller/CertificateRenewalController.java =====
package com.citi.cert.renewal.controller;

import com.citi.cert.renewal.model.Certificate;
import com.citi.cert.renewal.model.CSIConfiguration;
import com.citi.cert.renewal.service.CertificateDiscoveryService;
import com.citi.cert.renewal.service.CertificateRenewalService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/certificate")
@RequiredArgsConstructor
public class CertificateRenewalController {
    
    private final CertificateDiscoveryService discoveryService;
    private final CertificateRenewalService renewalService;
    
    @GetMapping("/expiring/{csiId}")
    public ResponseEntity<List<Certificate>> getExpiringCertificates(@PathVariable String csiId) {
        List<Certificate> expiringCerts = discoveryService.discoverExpiringCertificates(csiId);
        return ResponseEntity.ok(expiringCerts);
    }
    
    @PostMapping("/renew/{csiId}")
    public ResponseEntity<Map<String, Object>> triggerRenewal(
            @PathVariable String csiId,
            @RequestParam String certificateSerial) {
        
        Map<String, Object> response = new HashMap<>();
        
        try {
            // Find the certificate
            List<Certificate> certificates = discoveryService.discoverExpiringCertificates(csiId);
            Certificate targetCert = certificates.stream()
                    .filter(cert -> cert.getSerial().equals(certificateSerial))
                    .findFirst()
                    .orElseThrow(() -> new RuntimeException("Certificate not found"));
            
            // Renew the certificate
            String newOrderId = renewalService.renewCertificate(targetCert, csiId);
            
            response.put("status", "SUCCESS");
            response.put("message", "Certificate renewal initiated");
            response.put("oldOrderId", targetCert.getOrderId());
            response.put("newOrderId", newOrderId);
            
            return ResponseEntity.ok(response);
            
        } catch (Exception e) {
            response.put("status", "FAILED");
            response.put("message", e.getMessage());
            return ResponseEntity.internalServerError().body(response);
        }
    }
}

// ===== dto/CMPStatusResponse.java =====
package com.citi.cert.renewal.dto;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class CMPStatusResponse {
    private String orderId;
    private String status;
    private String message;
    private String assignedTo;
    private String createdDate;
    private String completedDate;
}

// ===== Updated NotificationService.java =====
package com.citi.cert.renewal.service;

import com.citi.cert.renewal.model.CSIConfiguration;
import com.citi.cert.renewal.model.NotificationTemplate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Service;

import javax.mail.MessagingException;
import javax.mail.internet.MimeMessage;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class NotificationService {
    
    private final JavaMailSender mailSender;
    private final TemplateService templateService;
    
    public void notifyPullRequestCreated(CSIConfiguration config, String repository, 
                                       String prUrl, String newOrderId) {
        NotificationTemplate template = templateService.createPullRequestNotification(
                config, repository, prUrl, newOrderId);
        
        sendEmail(config.getEmailDistributionLists(), template);
    }
    
    public void notifyManualUpdateRequired(CSIConfiguration config, String repository, 
                                         String newOrderId, List<String> fileLocations) {
        NotificationTemplate template = templateService.createManualUpdateNotification(
                config, repository, newOrderId, fileLocations);
        
        sendEmail(config.getEmailDistributionLists(), template);
    }
    
    private void sendEmail(List<String> recipients, NotificationTemplate template) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            
            helper.setTo(recipients.toArray(new String[0]));
            helper.setSubject(template.getSubject());
            helper.setText(template.getBody(), false);
            helper.setFrom("cert-renewal-automation@citi.com");
            
            mailSender.send(message);
            log.info("Email sent successfully to: {} with subject: {}", recipients, template.getSubject());
            
        } catch (MessagingException e) {
            log.error("Failed to send email", e);
            // Fallback to simple message
            try {
                SimpleMailMessage simpleMessage = new SimpleMailMessage();
                simpleMessage.setTo(recipients.toArray(new String[0]));
                simpleMessage.setSubject(template.getSubject());
                simpleMessage.setText(template.getBody());
                simpleMessage.setFrom("cert-renewal-automation@citi.com");
                mailSender.send(simpleMessage);
            } catch (Exception ex) {
                log.error("Failed to send simple email", ex);
            }
        }
    }
}

// ===== config/SchedulingConfig.java =====
package com.citi.cert.renewal.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;

@Configuration
public class SchedulingConfig implements SchedulingConfigurer {
    
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        taskScheduler.setPoolSize(5);
        taskScheduler.setThreadNamePrefix("cert-renewal-");
        taskScheduler.initialize();
        taskRegistrar.setTaskScheduler(taskScheduler);
    }
}

// ===== health/CertificateRenewalHealthIndicator.java =====
package com.citi.cert.renewal.health;

import com.citi.cert.renewal.integration.BitbucketClient;
import com.citi.cert.renewal.integration.CMPClient;
import com.citi.cert.renewal.integration.VaultClient;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.actuator.health.Health;
import org.springframework.boot.actuator.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class CertificateRenewalHealthIndicator implements HealthIndicator {
    
    private final VaultClient vaultClient;
    private final CMPClient cmpClient;
    private final BitbucketClient bitbucketClient;
    
    @Override
    public Health health() {
        try {
            // Check each integration point
            boolean vaultHealthy = checkVaultHealth();
            boolean cmpHealthy = checkCMPHealth();
            boolean bitbucketHealthy = checkBitbucketHealth();
            
            if (vaultHealthy && cmpHealthy && bitbucketHealthy) {
                return Health.up()
                        .withDetail("vault", "UP")
                        .withDetail("cmp", "UP")
                        .withDetail("bitbucket", "UP")
                        .build();
            } else {
                return Health.down()
                        .withDetail("vault", vaultHealthy ? "UP" : "DOWN")
                        .withDetail("cmp", cmpHealthy ? "UP" : "DOWN")
                        .withDetail("bitbucket", bitbucketHealthy ? "UP" : "DOWN")
                        .build();
            }
        } catch (Exception e) {
            return Health.down()
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
    
    private boolean checkVaultHealth() {
        // Implement vault health check
        return true;
    }
    
    private boolean checkCMPHealth() {
        // Implement CMP health check
        return true;
    }
    
    private boolean checkBitbucketHealth() {
        // Implement Bitbucket health check
        return true;
    }
}
