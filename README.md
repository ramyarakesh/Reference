// BitbucketService.java - Enhanced version with JGit
package com.citi.certmanagement.service;

import com.citi.certmanagement.config.AppConfig;
import com.citi.certmanagement.model.Certificate;
import com.citi.certmanagement.model.CsiMetadata;
import com.citi.certmanagement.model.RenewalStatus;
import com.citi.certmanagement.scanner.CodeScanner;
import com.citi.certmanagement.scanner.JavaFileScanner;
import com.citi.certmanagement.scanner.PropertiesFileScanner;
import com.citi.certmanagement.scanner.YamlFileScanner;
import com.google.common.util.concurrent.RateLimiter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.eclipse.jgit.api.Git;
import org.eclipse.jgit.api.errors.GitAPIException;
import org.eclipse.jgit.transport.UsernamePasswordCredentialsProvider;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;

import java.io.File;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

@Service
@Slf4j
public class BitbucketService {
    
    private final AppConfig appConfig;
    private final RateLimiter rateLimiter;
    
    @Qualifier("defaultRestTemplate")
    private final RestTemplate restTemplate;
    
    @Value("${cert-renewal.git.workspace:/tmp/cert-renewal-workspace}")
    private String gitWorkspace;
    
    private final Map<String, CodeScanner> scanners = new ConcurrentHashMap<>();
    
    public BitbucketService(AppConfig appConfig, RateLimiter rateLimiter, RestTemplate restTemplate) {
        this.appConfig = appConfig;
        this.rateLimiter = rateLimiter;
        this.restTemplate = restTemplate;
        
        // Initialize scanners
        scanners.put("java", new JavaFileScanner());
        scanners.put("properties", new PropertiesFileScanner());
        scanners.put("yml", new YamlFileScanner());
        scanners.put("yaml", new YamlFileScanner());
    }
    
    public List<Certificate.BitbucketLocation> scanForCertificate(Certificate cert, CsiMetadata csi) {
        List<Certificate.BitbucketLocation> locations = new ArrayList<>();
        
        if (csi.getFidUsername() == null || !appConfig.getBitbucket().isEnabled()) {
            log.warn("Bitbucket scanning disabled or FID not configured for CSI {}", csi.getCsiId());
            return locations;
        }
        
        String orderIdPattern = cert.getTicketId(); // API_C-*
        
        for (String repository : csi.getRepositories()) {
            try {
                List<Certificate.BitbucketLocation> repoLocations = 
                    searchRepository(repository, orderIdPattern, csi);
                locations.addAll(repoLocations);
            } catch (Exception e) {
                log.error("Failed to scan repository {}", repository, e);
            }
        }
        
        return locations;
    }
    
    public String updateCertificateReferences(Certificate cert, CsiMetadata csi, String newOrderId) 
            throws Exception {
        
        if (cert.getCertificateLocations() == null || cert.getCertificateLocations().isEmpty()) {
            throw new RuntimeException("No certificate locations found");
        }
        
        String branchName = "cert-renewal-" + cert.getCn().replaceAll("[^a-zA-Z0-9]", "-") + 
                           "-" + System.currentTimeMillis();
        
        // Group locations by repository
        Map<String, List<Certificate.BitbucketLocation>> locationsByRepo = 
            cert.getCertificateLocations().stream()
                .collect(Collectors.groupingBy(Certificate.BitbucketLocation::getRepository));
        
        String pullRequestUrl = null;
        List<RenewalStatus.UpdatedFile> allUpdatedFiles = new ArrayList<>();
        
        for (Map.Entry<String, List<Certificate.BitbucketLocation>> entry : locationsByRepo.entrySet()) {
            String repository = entry.getKey();
            List<Certificate.BitbucketLocation> repoLocations = entry.getValue();
            
            try {
                // Use JGit approach for updating files
                List<RenewalStatus.UpdatedFile> updatedFiles = updateFilesUsingGit(
                    repository, repoLocations, cert.getTicketId(), newOrderId, branchName, csi);
                
                allUpdatedFiles.addAll(updatedFiles);
                
                // Create pull request for this repository
                if (pullRequestUrl == null) {
                    pullRequestUrl = createPullRequest(repository, branchName, cert, newOrderId, csi);
                }
                
            } catch (Exception e) {
                log.error("Failed to update repository {}", repository, e);
                // Clean up workspace
                cleanupWorkspace(repository);
                throw e;
            }
        }
        
        return pullRequestUrl;
    }
    
    private List<RenewalStatus.UpdatedFile> updateFilesUsingGit(
            String repository, 
            List<Certificate.BitbucketLocation> locations,
            String oldOrderId, 
            String newOrderId, 
            String branchName,
            CsiMetadata csi) throws Exception {
        
        List<RenewalStatus.UpdatedFile> updatedFiles = new ArrayList<>();
        String repoPath = gitWorkspace + "/" + repository.replace("/", "_");
        File repoDir = new File(repoPath);
        
        // Clean up any existing workspace
        if (repoDir.exists()) {
            deleteDirectory(repoDir);
        }
        
        Git git = null;
        
        try {
            // Clone repository
            String cloneUrl = buildCloneUrl(repository);
            UsernamePasswordCredentialsProvider credentialsProvider = 
                new UsernamePasswordCredentialsProvider(csi.getFidUsername(), csi.getFidPassword());
            
            log.info("Cloning repository {} to {}", repository, repoPath);
            git = Git.cloneRepository()
                .setURI(cloneUrl)
                .setDirectory(repoDir)
                .setCredentialsProvider(credentialsProvider)
                .call();
            
            // Create and checkout new branch
            git.checkout()
                .setCreateBranch(true)
                .setName(branchName)
                .call();
            
            // Update files
            for (Certificate.BitbucketLocation location : locations) {
                Path filePath = Paths.get(repoDir.getAbsolutePath(), location.getFilePath());
                
                if (Files.exists(filePath)) {
                    // Read file content
                    String content = new String(Files.readAllBytes(filePath), StandardCharsets.UTF_8);
                    String originalContent = content;
                    
                    // Replace order ID
                    String newContent = content.replace(oldOrderId, newOrderId);
                    
                    if (!content.equals(newContent)) {
                        // Write updated content
                        Files.write(filePath, newContent.getBytes(StandardCharsets.UTF_8));
                        
                        // Stage the file
                        git.add().addFilepattern(location.getFilePath()).call();
                        
                        RenewalStatus.UpdatedFile updatedFile = new RenewalStatus.UpdatedFile();
                        updatedFile.setRepository(repository);
                        updatedFile.setFilePath(location.getFilePath());
                        updatedFile.setOldContent(originalContent);
                        updatedFile.setNewContent(newContent);
                        updatedFile.setUpdatedAt(LocalDateTime.now());
                        
                        updatedFiles.add(updatedFile);
                        
                        log.info("Updated file: {}", location.getFilePath());
                    }
                }
            }
            
            if (!updatedFiles.isEmpty()) {
                // Commit changes
                String commitMessage = String.format(
                    "Update certificate Order ID from %s to %s\n\n" +
                    "Automated certificate renewal update by CLM system",
                    oldOrderId, newOrderId
                );
                
                git.commit()
                    .setMessage(commitMessage)
                    .setAuthor("CLM System", "clm-system@citi.com")
                    .call();
                
                // Push branch
                git.push()
                    .setCredentialsProvider(credentialsProvider)
                    .call();
                
                log.info("Pushed changes to branch {}", branchName);
            }
            
        } finally {
            if (git != null) {
                git.close();
            }
            // Clean up workspace
            cleanupWorkspace(repository);
        }
        
        return updatedFiles;
    }
    
    private String buildCloneUrl(String repository) {
        // Format: https://username@bitbucket-host/scm/PROJECT/repo.git
        String[] parts = repository.split("/");
        String project = parts[0];
        String repo = parts[1];
        
        return String.format("%s/scm/%s/%s.git", 
            appConfig.getBitbucket().getBaseUrl().replace("https://", "https://"),
            project, repo);
    }
    
    private void cleanupWorkspace(String repository) {
        try {
            String repoPath = gitWorkspace + "/" + repository.replace("/", "_");
            File repoDir = new File(repoPath);
            if (repoDir.exists()) {
                deleteDirectory(repoDir);
            }
        } catch (Exception e) {
            log.warn("Failed to cleanup workspace for {}", repository, e);
        }
    }
    
    private void deleteDirectory(File directory) throws IOException {
        if (directory.exists()) {
            Files.walk(directory.toPath())
                .sorted(Comparator.reverseOrder())
                .map(Path::toFile)
                .forEach(File::delete);
        }
    }
    
    public boolean checkWriteAccess(List<Certificate.BitbucketLocation> locations, CsiMetadata csi) {
        if (locations.isEmpty() || csi.getFidUsername() == null) {
            return false;
        }
        
        // Check write access for the first repository
        Certificate.BitbucketLocation firstLocation = locations.get(0);
        String checkUrl = String.format("%s/rest/api/1.0/projects/%s/repos/%s/permissions/users",
                                      appConfig.getBitbucket().getBaseUrl(),
                                      extractProjectKey(firstLocation.getRepository()),
                                      extractRepoSlug(firstLocation.getRepository()));
        
        HttpHeaders headers = createAuthHeaders(csi);
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        rateLimiter.acquire();
        
        try {
            ResponseEntity<Map> response = restTemplate.exchange(
                checkUrl + "?filter=" + csi.getFidUsername(), 
                HttpMethod.GET, entity, Map.class);
            
            if (response.getStatusCode() == HttpStatus.OK) {
                Map<String, Object> body = response.getBody();
                List<Map<String, Object>> values = (List<Map<String, Object>>) body.get("values");
                
                for (Map<String, Object> user : values) {
                    String permission = (String) user.get("permission");
                    if ("REPO_WRITE".equals(permission) || "REPO_ADMIN".equals(permission)) {
                        return true;
                    }
                }
            }
        } catch (Exception e) {
            log.error("Failed to check write access", e);
        }
        
        return false;
    }
    
    private String createPullRequest(String repository, String branchName, 
                                   Certificate cert, String newOrderId, CsiMetadata csi) throws Exception {
        
        String url = String.format("%s/rest/api/1.0/projects/%s/repos/%s/pull-requests",
                                 appConfig.getBitbucket().getBaseUrl(),
                                 extractProjectKey(repository),
                                 extractRepoSlug(repository));
        
        Map<String, Object> request = Map.of(
            "title", "Certificate Renewal: Update Order ID for " + cert.getCn(),
            "description", String.format(
                "## Automated Certificate Renewal Update\n\n" +
                "### Certificate Details\n" +
                "- **CN**: %s\n" +
                "- **Environment**: %s\n" +
                "- **CSI ID**: %d\n\n" +
                "### Changes\n" +
                "- **Old Order ID**: `%s`\n" +
                "- **New Order ID**: `%s`\n\n" +
                "### Certificate Information\n" +
                "- **Expiry Date**: %s\n" +
                "- **Reference ID**: %s\n" +
                "- **Renewal Provider**: %s\n\n" +
                "---\n" +
                "*This PR was automatically generated by the Certificate Lifecycle Management (CLM) system.*",
                cert.getCn(), cert.getEnvironment(), cert.getAppid(),
                cert.getTicketId(), newOrderId, 
                cert.getValidTo(),
                cert.getReferenceId() != null ? cert.getReferenceId() : "N/A",
                cert.getRenewalProvider() != null ? cert.getRenewalProvider() : "Default"
            ),
            "state", "OPEN",
            "open", true,
            "closed", false,
            "fromRef", Map.of(
                "id", "refs/heads/" + branchName,
                "repository", Map.of(
                    "slug", extractRepoSlug(repository),
                    "project", Map.of("key", extractProjectKey(repository))
                )
            ),
            "toRef", Map.of(
                "id", "refs/heads/master",
                "repository", Map.of(
                    "slug", extractRepoSlug(repository),
                    "project", Map.of("key", extractProjectKey(repository))
                )
            ),
            "reviewers", new ArrayList<>()
        );
        
        HttpHeaders headers = createAuthHeaders(csi);
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(request, headers);
        
        rateLimiter.acquire();
        
        ResponseEntity<Map> response = restTemplate.postForEntity(url, entity, Map.class);
        
        if (response.getStatusCode() == HttpStatus.CREATED || response.getStatusCode() == HttpStatus.OK) {
            Map<String, Object> body = response.getBody();
            Map<String, Object> links = (Map<String, Object>) body.get("links");
            List<Map<String, Object>> selfLinks = (List<Map<String, Object>>) links.get("self");
            
            if (!selfLinks.isEmpty()) {
                return (String) selfLinks.get(0).get("href");
            }
        }
        
        throw new RuntimeException("Failed to create pull request: " + response.getStatusCode());
    }
    
    // Keep existing helper methods...
    private List<Certificate.BitbucketLocation> searchRepository(String repository, 
                                                                String searchPattern, 
                                                                CsiMetadata csi) {
        List<Certificate.BitbucketLocation> locations = new ArrayList<>();
        
        String searchUrl = String.format("%s/rest/api/1.0/projects/%s/repos/%s/browse",
                                       appConfig.getBitbucket().getBaseUrl(),
                                       extractProjectKey(repository),
                                       extractRepoSlug(repository));
        
        HttpHeaders headers = createAuthHeaders(csi);
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        rateLimiter.acquire();
        
        try {
            // Search for files recursively
            searchFilesRecursively(searchUrl, "", searchPattern, repository, csi, locations);
        } catch (Exception e) {
            log.error("Failed to search repository {}", repository, e);
        }
        
        return locations;
    }
    
    private void searchFilesRecursively(String baseUrl, String path, String pattern, 
                                      String repository, CsiMetadata csi, 
                                      List<Certificate.BitbucketLocation> locations) {
        
        HttpHeaders headers = createAuthHeaders(csi);
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        String url = baseUrl + (path.isEmpty() ? "" : "/" + path) + "?limit=1000";
        
        try {
            ResponseEntity<Map> response = restTemplate.exchange(url, HttpMethod.GET, entity, Map.class);
            
            if (response.getStatusCode() == HttpStatus.OK) {
                Map<String, Object> body = response.getBody();
                List<Map<String, Object>> children = (List<Map<String, Object>>) body.get("children");
                
                if (children != null) {
                    for (Map<String, Object> child : children) {
                        String type = (String) child.get("type");
                        Map<String, Object> pathInfo = (Map<String, Object>) child.get("path");
                        String childPath = (String) pathInfo.get("toString");
                        
                        if ("FILE".equals(type)) {
                            String extension = getFileExtension(childPath);
                            if (scanners.containsKey(extension)) {
                                // Get file content
                                String fileContent = getFileContent(repository, childPath, csi);
                                
                                if (fileContent != null && fileContent.contains(pattern)) {
                                    Certificate.BitbucketLocation location = 
                                        createLocation(repository, childPath, fileContent, pattern);
                                    locations.add(location);
                                }
                            }
                        } else if ("DIRECTORY".equals(type)) {
                            // Recursively search subdirectories
                            searchFilesRecursively(baseUrl, childPath, pattern, repository, csi, locations);
                        }
                    }
                }
            }
        } catch (Exception e) {
            log.error("Failed to search path {} in repository {}", path, repository, e);
        }
    }
    
    // Test methods for mock functionality
    public void testBitbucketOperations(String testMode) {
        log.info("Running Bitbucket test in mode: {}", testMode);
        
        switch (testMode) {
            case "clone":
                testGitClone();
                break;
            case "pr":
                testPullRequestCreation();
                break;
            case "scan":
                testFileScan();
                break;
            default:
                log.warn("Unknown test mode: {}", testMode);
        }
    }
    
    private void testGitClone() {
        try {
            String testRepo = "TEST/test-repo";
            String testPath = gitWorkspace + "/test_clone";
            File testDir = new File(testPath);
            
            log.info("Test: Creating mock repository at {}", testPath);
            
            if (!testDir.exists()) {
                testDir.mkdirs();
            }
            
            // Initialize git repo
            Git git = Git.init().setDirectory(testDir).call();
            
            // Create test file
            File testFile = new File(testDir, "test.properties");
            Files.write(testFile.toPath(), 
                       "certificate.order.id=API_C-test-order-id\n".getBytes());
            
            git.add().addFilepattern("test.properties").call();
            git.commit().setMessage("Test commit").call();
            
            log.info("Test: Successfully created mock git repository");
            
            git.close();
            deleteDirectory(testDir);
            
        } catch (Exception e) {
            log.error("Test clone failed", e);
        }
    }
    
    private void testPullRequestCreation() {
        log.info("Test: Mock pull request creation");
        log.info("Test: PR URL would be: https://bitbucket.example.com/projects/TEST/repos/test-repo/pull-requests/123");
    }
    
    private void testFileScan() {
        log.info("Test: Scanning mock files for certificate patterns");
        
        String mockJavaContent = "private static final String ORDER_ID = \"API_C-89db54d0-a825-4a2c-a6a5-1e2f2cda4974\";";
        String mockPropsContent = "certificate.order=API_C-89db54d0-a825-4a2c-a6a5-1e2f2cda4974";
        String mockYamlContent = "orderid: API_C-89db54d0-a825-4a2c-a6a5-1e2f2cda4974";
        
        JavaFileScanner javaScanner = new JavaFileScanner();
        PropertiesFileScanner propsScanner = new PropertiesFileScanner();
        YamlFileScanner yamlScanner = new YamlFileScanner();
        
        String pattern = "API_C-89db54d0-a825-4a2c-a6a5-1e2f2cda4974";
        
        List<Certificate.BitbucketLocation> javaLocs = javaScanner.scan(mockJavaContent, pattern, "TEST/repo", "Test.java");
        List<Certificate.BitbucketLocation> propsLocs = propsScanner.scan(mockPropsContent, pattern, "TEST/repo", "app.properties");
        List<Certificate.BitbucketLocation> yamlLocs = yamlScanner.scan(mockYamlContent, pattern, "TEST/repo", "config.yaml");
        
        log.info("Test: Found {} Java locations", javaLocs.size());
        log.info("Test: Found {} Properties locations", propsLocs.size());
        log.info("Test: Found {} YAML locations", yamlLocs.size());
    }
    
    private HttpHeaders createAuthHeaders(CsiMetadata csi) {
        HttpHeaders headers = new HttpHeaders();
        String auth = csi.getFidUsername() + ":" + csi.getFidPassword();
        byte[] encodedAuth = Base64.getEncoder().encode(auth.getBytes(StandardCharsets.UTF_8));
        String authHeader = "Basic " + new String(encodedAuth);
        headers.set("Authorization", authHeader);
        return headers;
    }
    
    private Certificate.BitbucketLocation createLocation(String repository, String filePath, 
                                                       String content, String searchPattern) {
        Certificate.BitbucketLocation location = new Certificate.BitbucketLocation();
        location.setRepository(repository);
        location.setFilePath(filePath);
        location.setFileType(getFileExtension(filePath).toUpperCase());
        location.setBranch("master");
        location.setAccessible(true);
        
        // Find line number and context
        String[] lines = content.split("\n");
        for (int i = 0; i < lines.length; i++) {
            if (lines[i].contains(searchPattern)) {
                location.setLineNumber(i + 1);
                location.setContext(extractContext(lines, i));
                break;
            }
        }
        
        return location;
    }
    
    private String extractContext(String[] lines, int index) {
        StringBuilder context = new StringBuilder();
        int start = Math.max(0, index - 2);
        int end = Math.min(lines.length - 1, index + 2);
        
        for (int i = start; i <= end; i++) {
            if (i == index) {
                context.append(">>> ");
            }
            context.append(lines[i]).append("\n");
        }
        
        return context.toString();
    }
    
    private String getFileContent(String repository, String filePath, CsiMetadata csi) {
        String url = String.format("%s/rest/api/1.0/projects/%s/repos/%s/raw/%s",
                                 appConfig.getBitbucket().getBaseUrl(),
                                 extractProjectKey(repository),
                                 extractRepoSlug(repository),
                                 filePath);
        
        HttpHeaders headers = createAuthHeaders(csi);
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        rateLimiter.acquire();
        
        try {
            ResponseEntity<String> response = restTemplate.exchange(
                url, HttpMethod.GET, entity, String.class);
            
            if (response.getStatusCode() == HttpStatus.OK) {
                return response.getBody();
            }
        } catch (Exception e) {
            log.error("Failed to get file content for {}", filePath, e);
        }
        
        return null;
    }
    
    private String extractProjectKey(String repository) {
        return repository.split("/")[0];
    }
    
    private String extractRepoSlug(String repository) {
        return repository.split("/")[1];
    }
    
    private String getFileExtension(String filename) {
        int lastDot = filename.lastIndexOf('.');
        return lastDot > 0 ? filename.substring(lastDot + 1).toLowerCase() : "";
    }
}




---------



// Enhanced Certificate.java
package com.citi.certmanagement.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import lombok.Data;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

@Document(collection = "AppCerts")
@Data
public class Certificate {
    @Id
    private String id;
    private String cn;
    private String country;
    private String state;
    private String serial;
    private String environment;
    private List<String> sanList;
    private List<String> ouList;
    private String policyId;
    private String policyNickName;
    private String ticketId; // Order ID (API_C-*)
    private Integer appid; // CSI ID
    private LocalDateTime validTo;
    private LocalDateTime validFrom;
    private String issuer;
    private String certRole;
    private String vaultPath;
    
    // Reference tracking for auto-renewal detection
    private String referenceId; // Doesn't change on renewal
    private String nickname; // May or may not change
    private String distinguishedName; // Doesn't change on renewal
    
    // Provider tracking
    private String originalProvider; // Provider that originally provisioned
    private String currentProvider; // Current provider preference
    private Boolean autoRenewedByProvider; // True if auto-renewed by PP/ACM
    
    // Renewal tracking
    private LocalDateTime discoveredAt;
    private LocalDateTime lastRenewalCheck;
    private LocalDateTime renewalScheduledDate;
    private String renewalStatus;
    private String renewalProvider; // PP, CMP, ACM
    private String newOrderId;
    private String renewalWorkflowId;
    private String previousOrderId; // Track previous order ID for auto-renewal detection
    
    // Pre-scan results
    private Boolean preScanCompleted;
    private LocalDateTime preScanDate;
    private List<BitbucketLocation> certificateLocations;
    private Boolean hasWriteAccess;
    
    // PR tracking
    private String lastPullRequestUrl;
    private LocalDateTime lastPullRequestDate;
    private String pullRequestStatus; // OPEN, MERGED, DECLINED
    
    // Metadata
    private Map<String, Object> additionalData;
    private LocalDateTime lastModified;
    private Boolean isAutoRenewalCandidate; // True if originally from PP/ACM
    
    @Data
    public static class BitbucketLocation {
        private String repository;
        private String branch;
        private String filePath;
        private String fileType; // JAVA, PROPERTIES, YAML
        private Integer lineNumber;
        private String context;
        private Boolean accessible;
    }
}

// Enhanced CertificateDiscoveryService.java
package com.citi.certmanagement.service;

import com.citi.certmanagement.config.AppConfig;
import com.citi.certmanagement.model.Certificate;
import com.citi.certmanagement.model.CsiMetadata;
import com.citi.certmanagement.model.AuditLog;
import com.citi.certmanagement.repository.CertificateRepository;
import com.citi.certmanagement.repository.CsiMetadataRepository;
import com.citi.certmanagement.repository.AuditLogRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.google.common.util.concurrent.RateLimiter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import org.springframework.beans.factory.annotation.Qualifier;
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class CertificateDiscoveryService {
    
    private final CertificateRepository certificateRepository;
    private final CsiMetadataRepository csiMetadataRepository;
    private final AuditLogRepository auditLogRepository;
    private final AppConfig appConfig;
    private final RateLimiter rateLimiter;
    private final NotificationService notificationService;
    private final BitbucketService bitbucketService;
    
    @Qualifier("defaultRestTemplate")
    private final RestTemplate restTemplate;
    
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Scheduled(cron = "0 0 2 * * ?") // Run daily at 2 AM
    public void discoverCertificates() {
        log.info("Starting daily certificate discovery");
        List<CsiMetadata> activeCsis = csiMetadataRepository.findByActive(true);
        
        int totalDiscovered = 0;
        int totalExpiring = 0;
        int autoRenewed = 0;
        List<String> errors = new ArrayList<>();
        
        for (CsiMetadata csi : activeCsis) {
            try {
                log.info("Discovering certificates for CSI: {}", csi.getCsiId());
                DiscoveryResult result = discoverCertificatesForCsi(csi);
                
                totalDiscovered += result.totalDiscovered;
                totalExpiring += result.expiringCertificates;
                autoRenewed += result.autoRenewedCertificates;
                
            } catch (Exception e) {
                String error = String.format("Failed to discover certificates for CSI %d: %s", 
                                           csi.getCsiId(), e.getMessage());
                log.error(error, e);
                errors.add(error);
                
                auditLog("DISCOVERY", "CSI", csi.getCsiId().toString(), 
                        "FAILURE", Map.of("error", e.getMessage()));
            }
        }
        
        // Send summary notification
        Map<String, Object> summaryData = Map.of(
            "totalCsis", activeCsis.size(),
            "totalDiscovered", totalDiscovered,
            "totalExpiring", totalExpiring,
            "autoRenewedDetected", autoRenewed,
            "errors", errors
        );
        
        notificationService.sendDiscoverySummary(summaryData);
        
        log.info("Certificate discovery completed. Total discovered: {}, Expiring: {}, Auto-renewed: {}", 
                totalDiscovered, totalExpiring, autoRenewed);
    }
    
    private DiscoveryResult discoverCertificatesForCsi(CsiMetadata csi) {
        DiscoveryResult result = new DiscoveryResult();
        List<Certificate> discoveredCerts = new ArrayList<>();
        
        // Query Vault for certificates
        String vaultUrl = String.format("%s/%s/kv2secret/data/%d", 
                                       appConfig.getVault().getBaseUrl(),
                                       appConfig.getVault().getApiVersion(),
                                       csi.getCsiId());
        
        rateLimiter.acquire();
        
        try {
            Map<String, Object> vaultResponse = restTemplate.getForObject(vaultUrl, Map.class);
            
            if (vaultResponse != null && vaultResponse.containsKey("data")) {
                List<Map<String, Object>> certificates = 
                    (List<Map<String, Object>>) ((Map) vaultResponse.get("data")).get("certificates");
                
                for (Map<String, Object> certData : certificates) {
                    Certificate cert = mapToCertificate(certData, csi.getCsiId());
                    
                    // Check for auto-renewal by comparing with existing certificates
                    Certificate processedCert = processDiscoveredCertificate(cert, csi);
                    
                    if (processedCert.getAutoRenewedByProvider() == Boolean.TRUE) {
                        result.autoRenewedCertificates++;
                        
                        // Handle auto-renewed certificate
                        handleAutoRenewedCertificate(processedCert, csi);
                    }
                    
                    discoveredCerts.add(processedCert);
                    
                    // Check for expiring certificates
                    LocalDateTime expiryThreshold = LocalDateTime.now()
                        .plusDays(appConfig.getRenewal().getExpiryThresholdDays());
                    LocalDateTime preScanThreshold = LocalDateTime.now()
                        .plusDays(appConfig.getRenewal().getPreScanThresholdDays());
                    
                    if (processedCert.getValidTo().isBefore(expiryThreshold)) {
                        result.expiringCertificates++;
                        log.warn("Certificate expiring soon: CN={}, CSI={}, ExpiryDate={}", 
                                processedCert.getCn(), processedCert.getAppid(), processedCert.getValidTo());
                    }
                    
                    // Perform pre-scan if needed
                    if (processedCert.getValidTo().isBefore(preScanThreshold) && 
                        !Boolean.TRUE.equals(processedCert.getPreScanCompleted())) {
                        performPreScan(processedCert, csi);
                    }
                }
                
                result.totalDiscovered = discoveredCerts.size();
            }
            
            auditLog("DISCOVERY", "CSI", csi.getCsiId().toString(), 
                    "SUCCESS", Map.of("certificatesFound", result.totalDiscovered,
                                    "autoRenewed", result.autoRenewedCertificates));
            
        } catch (Exception e) {
            log.error("Failed to query Vault for CSI {}", csi.getCsiId(), e);
            throw new RuntimeException("Vault query failed", e);
        }
        
        return result;
    }
    
    private Certificate processDiscoveredCertificate(Certificate newCert, CsiMetadata csi) {
        // Look for existing certificates with same reference ID but different order ID
        List<Certificate> existingCerts = certificateRepository.findByAppid(newCert.getAppid());
        
        for (Certificate existing : existingCerts) {
            // Check if this is an auto-renewed certificate
            if (existing.getReferenceId() != null && 
                existing.getReferenceId().equals(newCert.getReferenceId()) &&
                !existing.getTicketId().equals(newCert.getTicketId()) &&
                existing.getDistinguishedName() != null &&
                existing.getDistinguishedName().equals(newCert.getDistinguishedName())) {
                
                // This is an auto-renewed certificate
                log.info("Detected auto-renewed certificate: CN={}, Old OrderID={}, New OrderID={}", 
                        newCert.getCn(), existing.getTicketId(), newCert.getTicketId());
                
                newCert.setAutoRenewedByProvider(true);
                newCert.setPreviousOrderId(existing.getTicketId());
                newCert.setOriginalProvider(existing.getOriginalProvider());
                
                // Mark old certificate as replaced
                existing.setRenewalStatus("REPLACED");
                existing.setNewOrderId(newCert.getTicketId());
                certificateRepository.save(existing);
                
                break;
            }
        }
        
        // Check if certificate already exists by serial
        Optional<Certificate> existingBySerial = certificateRepository.findBySerial(newCert.getSerial());
        
        if (existingBySerial.isPresent()) {
            // Update existing certificate
            Certificate existing = existingBySerial.get();
            updateCertificateData(existing, newCert);
            return certificateRepository.save(existing);
        } else {
            // Save new certificate
            newCert.setDiscoveredAt(LocalDateTime.now());
            newCert.setLastModified(LocalDateTime.now());
            
            // Set provider tracking
            if (newCert.getOriginalProvider() == null) {
                // Determine original provider based on order ID pattern or other criteria
                newCert.setOriginalProvider(determineOriginalProvider(newCert));
            }
            newCert.setCurrentProvider(csi.getPreferredRenewalProvider() != null ? 
                                      csi.getPreferredRenewalProvider() : 
                                      appConfig.getRenewal().getDefaultProvider());
            
            return certificateRepository.save(newCert);
        }
    }
    
    private void handleAutoRenewedCertificate(Certificate cert, CsiMetadata csi) {
        log.info("Handling auto-renewed certificate: {}", cert.getCn());
        
        // If certificate has Bitbucket locations from previous scan, update them
        if (cert.getPreviousOrderId() != null) {
            // Find the old certificate
            Optional<Certificate> oldCertOpt = certificateRepository.findByTicketId(cert.getPreviousOrderId());
            
            if (oldCertOpt.isPresent()) {
                Certificate oldCert = oldCertOpt.get();
                
                if (oldCert.getCertificateLocations() != null && !oldCert.getCertificateLocations().isEmpty()) {
                    // Copy locations to new certificate
                    cert.setCertificateLocations(oldCert.getCertificateLocations());
                    cert.setHasWriteAccess(oldCert.getHasWriteAccess());
                    cert.setPreScanCompleted(true);
                    cert.setPreScanDate(LocalDateTime.now());
                    
                    // Create PR if write access is available
                    if (Boolean.TRUE.equals(cert.getHasWriteAccess()) && csi.getFidUsername() != null) {
                        try {
                            String prUrl = bitbucketService.updateCertificateReferences(
                                cert, csi, cert.getTicketId());
                            
                            cert.setLastPullRequestUrl(prUrl);
                            cert.setLastPullRequestDate(LocalDateTime.now());
                            cert.setPullRequestStatus("OPEN");
                            
                            certificateRepository.save(cert);
                            
                            // Send notification
                            notificationService.sendAutoRenewalPullRequestCreated(csi, cert, prUrl);
                            
                        } catch (Exception e) {
                            log.error("Failed to create PR for auto-renewed certificate", e);
                            notificationService.sendAutoRenewalManualActionRequired(csi, cert);
                        }
                    } else {
                        // Send manual action notification
                        notificationService.sendAutoRenewalManualActionRequired(csi, cert);
                    }
                }
            }
        }
    }
    
    private void performPreScan(Certificate cert, CsiMetadata csi) {
        log.info("Performing pre-scan for certificate: {}", cert.getCn());
        
        try {
            if (csi.getFidUsername() != null && appConfig.getBitbucket().isEnabled()) {
                List<Certificate.BitbucketLocation> locations = 
                    bitbucketService.scanForCertificate(cert, csi);
                
                cert.setCertificateLocations(locations);
                cert.setHasWriteAccess(bitbucketService.checkWriteAccess(locations, csi));
                cert.setPreScanCompleted(true);
                cert.setPreScanDate(LocalDateTime.now());
                
                certificateRepository.save(cert);
                
                auditLog("PRE_SCAN", "CERTIFICATE", cert.getId(), 
                        "SUCCESS", Map.of("locationsFound", locations.size(),
                                        "hasWriteAccess", cert.getHasWriteAccess()));
            }
        } catch (Exception e) {
            log.error("Pre-scan failed for certificate {}", cert.getCn(), e);
            
            auditLog("PRE_SCAN", "CERTIFICATE", cert.getId(), 
                    "FAILURE", Map.of("error", e.getMessage()));
        }
    }
    
    private Certificate mapToCertificate(Map<String, Object> certData, Integer csiId) {
        Certificate cert = new Certificate();
        cert.setCn((String) certData.get("cn"));
        cert.setCountry((String) certData.get("country"));
        cert.setState((String) certData.get("state"));
        cert.setSerial((String) certData.get("serial"));
        cert.setEnvironment((String) certData.get("environment"));
        cert.setSanList((List<String>) certData.get("sanList"));
        cert.setOuList((List<String>) certData.get("ouList"));
        cert.setPolicyId((String) certData.get("policyId"));
        cert.setPolicyNickName((String) certData.get("policyNickName"));
        cert.setTicketId((String) certData.get("ticketId"));
        cert.setAppid(csiId);
        cert.setIssuer((String) certData.get("issuer"));
        cert.setCertRole((String) certData.get("certRole"));
        cert.setVaultPath((String) certData.get("vaultPath"));
        
        // Map additional fields for auto-renewal detection
        cert.setReferenceId((String) certData.get("referenceId"));
        cert.setNickname((String) certData.get("nickname"));
        cert.setDistinguishedName((String) certData.get("distinguishedName"));
        
        // Parse dates
        String validTo = (String) certData.get("validTo");
        if (validTo != null) {
            cert.setValidTo(LocalDateTime.parse(validTo));
        }
        
        String validFrom = (String) certData.get("validFrom");
        if (validFrom != null) {
            cert.setValidFrom(LocalDateTime.parse(validFrom));
        }
        
        cert.setDiscoveredAt(LocalDateTime.now());
        cert.setLastModified(LocalDateTime.now());
        
        // Calculate renewal date
        if (cert.getValidTo() != null) {
            cert.setRenewalScheduledDate(cert.getValidTo().minusDays(90));
        }
        
        // Check if this is an auto-renewal candidate
        if (isAutoRenewalProvider(cert)) {
            cert.setIsAutoRenewalCandidate(true);
        }
        
        return cert;
    }
    
    private void updateCertificateData(Certificate existing, Certificate newData) {
        // Update only changed fields
        existing.setCn(newData.getCn());
        existing.setSanList(newData.getSanList());
        existing.setReferenceId(newData.getReferenceId());
        existing.setNickname(newData.getNickname());
        existing.setDistinguishedName(newData.getDistinguishedName());
        existing.setLastModified(LocalDateTime.now());
        
        existing.setValidTo(newData.getValidTo());
        existing.setValidFrom(newData.getValidFrom());
        
        if (existing.getValidTo() != null) {
            existing.setRenewalScheduledDate(existing.getValidTo().minusDays(90));
        }
    }
    
    private String determineOriginalProvider(Certificate cert) {
        // Logic to determine original provider based on patterns
        // This could be enhanced based on specific patterns in your environment
        if (cert.getTicketId() != null && cert.getTicketId().contains("API_C")) {
            // Check for specific patterns that indicate provider
            // For now, default logic
            return "CMP";
        }
        return "UNKNOWN";
    }
    
    private boolean isAutoRenewalProvider(Certificate cert) {
        // Provisioning Portal and ACM do auto-renewal
        String provider = determineOriginalProvider(cert);
        return "PROVISIONING_PORTAL".equals(provider) || "ACM".equals(provider);
    }
    
    private void auditLog(String action, String entityType, String entityId, 
                         String status, Map<String, Object> details) {
        AuditLog log = new AuditLog();
        log.setAction(action);
        log.setEntityType(entityType);
        log.setEntityId(entityId);
        log.setStatus(status);
        log.setTimestamp(LocalDateTime.now());
        log.setDetails(details);
        log.setPerformedBy("SYSTEM");
        
        auditLogRepository.save(log);
    }
    
    public List<Certificate> getExpiringCertificates(Integer days) {
        LocalDateTime threshold = LocalDateTime.now().plusDays(days);
        return certificateRepository.findExpiringCertificates(threshold);
    }
    
    public Map<String, Object> getCertificateStats() {
        LocalDateTime now = LocalDateTime.now();
        
        return Map.of(
            "total", certificateRepository.count(),
            "expiring30Days", certificateRepository.findExpiringCertificates(now.plusDays(30)).size(),
            "expiring60Days", certificateRepository.findExpiringCertificates(now.plusDays(60)).size(),
            "expiring90Days", certificateRepository.findExpiringCertificates(now.plusDays(90)).size(),
            "expired", certificateRepository.countExpiredCertificates(now)
        );
    }
    
    // Test method for discovery
    public void testDiscovery() {
        log.info("Running discovery test with mock data");
        
        // Create mock certificate data
        Map<String, Object> mockCertData = new HashMap<>();
        mockCertData.put("cn", "test.example.com");
        mockCertData.put("serial", "123456789");
        mockCertData.put("ticketId", "API_C-test-order-id");
        mockCertData.put("referenceId", "ref-12345");
        mockCertData.put("nickname", "test-cert");
        mockCertData.put("distinguishedName", "CN=test.example.com,OU=166479,O=Test");
        mockCertData.put("validTo", LocalDateTime.now().plusDays(30).toString());
        mockCertData.put("environment", "TEST");
        
        Certificate mockCert = mapToCertificate(mockCertData, 166479);
        log.info("Test: Created mock certificate: {}", mockCert.getCn());
        
        // Simulate auto-renewal detection
        mockCert.setAutoRenewedByProvider(true);
        mockCert.setPreviousOrderId("API_C-old-order-id");
        
        log.info("Test: Detected auto-renewal - Old: {}, New: {}", 
                mockCert.getPreviousOrderId(), mockCert.getTicketId());
    }
    
    private static class DiscoveryResult {
        int totalDiscovered = 0;
        int expiringCertificates = 0;
        int autoRenewedCertificates = 0;
    }
}



------------


// Enhanced CertificateRenewalService.java
package com.citi.certmanagement.service;

import com.citi.certmanagement.config.AppConfig;
import com.citi.certmanagement.model.*;
import com.citi.certmanagement.repository.*;
import com.citi.certmanagement.provider.CertificateProvider;
import com.citi.certmanagement.provider.ProvisioningPortalProvider;
import com.citi.certmanagement.provider.CmpProvider;
import com.citi.certmanagement.exception.CertificateException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.*;

@Service
@RequiredArgsConstructor
@Slf4j
public class CertificateRenewalService {
    
    private final CertificateRepository certificateRepository;
    private final CsiMetadataRepository csiMetadataRepository;
    private final RenewalStatusRepository renewalStatusRepository;
    private final AuditLogRepository auditLogRepository;
    private final AppConfig appConfig;
    private final ProvisioningPortalProvider provisioningPortalProvider;
    private final CmpProvider cmpProvider;
    private final BitbucketService bitbucketService;
    private final NotificationService notificationService;
    private final ExecutorService executorService = Executors.newFixedThreadPool(10);
    
    @Scheduled(cron = "0 0 3 * * ?") // Run daily at 3 AM
    public void processRenewals() {
        log.info("Starting certificate renewal process");
        
        LocalDateTime expiryThreshold = LocalDateTime.now()
            .plusDays(appConfig.getRenewal().getExpiryThresholdDays());
        
        List<Certificate> expiringCerts = certificateRepository.findExpiringCertificates(expiryThreshold);
        
        // Filter out certificates that are already renewed or auto-renewed
        expiringCerts = expiringCerts.stream()
            .filter(cert -> !isAlreadyRenewed(cert))
            .collect(Collectors.toList());
        
        // Group by CSI for batch processing
        Map<Integer, List<Certificate>> certsByCsi = expiringCerts.stream()
            .collect(Collectors.groupingBy(Certificate::getAppid));
        
        List<CompletableFuture<Void>> futures = new ArrayList<>();
        
        for (Map.Entry<Integer, List<Certificate>> entry : certsByCsi.entrySet()) {
            CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                try {
                    processCsiRenewals(entry.getKey(), entry.getValue());
                } catch (Exception e) {
                    log.error("Failed to process renewals for CSI {}", entry.getKey(), e);
                }
            }, executorService);
            
            futures.add(future);
        }
        
        // Wait for all renewals to complete
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        
        log.info("Certificate renewal process completed");
        
        // Process failed renewals for retry
        retryFailedRenewals();
    }
    
    public Map<String, Object> renewCertificateManually(String certificateId) {
        log.info("Manual renewal requested for certificate: {}", certificateId);
        
        Certificate cert = certificateRepository.findById(certificateId)
            .orElseThrow(() -> new CertificateException("Certificate not found", "CERT_NOT_FOUND"));
        
        // Validate renewal eligibility
        RenewalValidationResult validation = validateRenewalEligibility(cert);
        
        if (!validation.isEligible()) {
            log.warn("Certificate {} is not eligible for renewal: {}", certificateId, validation.getReason());
            return Map.of(
                "status", "REJECTED",
                "reason", validation.getReason(),
                "certificateId", certificateId,
                "expiryDate", cert.getValidTo(),
                "daysUntilExpiry", calculateDaysUntilExpiry(cert.getValidTo())
            );
        }
        
        CsiMetadata csi = csiMetadataRepository.findByCsiId(cert.getAppid())
            .orElseThrow(() -> new CertificateException("CSI not found", "CSI_NOT_FOUND"));
        
        try {
            renewCertificate(cert, csi);
            
            return Map.of(
                "status", "INITIATED",
                "certificateId", certificateId,
                "message", "Certificate renewal has been initiated successfully",
                "timestamp", LocalDateTime.now()
            );
            
        } catch (Exception e) {
            log.error("Manual renewal failed for certificate {}", certificateId, e);
            throw new CertificateException("Failed to initiate renewal: " + e.getMessage(), "RENEWAL_FAILED", e);
        }
    }
    
    private RenewalValidationResult validateRenewalEligibility(Certificate cert) {
        RenewalValidationResult result = new RenewalValidationResult();
        
        // Check if already renewed
        if (isAlreadyRenewed(cert)) {
            result.setEligible(false);
            result.setReason("Certificate has already been renewed or replaced");
            return result;
        }
        
        // Check if renewal is in progress
        if ("IN_PROGRESS".equals(cert.getRenewalStatus())) {
            result.setEligible(false);
            result.setReason("Certificate renewal is already in progress");
            return result;
        }
        
        // Check expiry threshold
        long daysUntilExpiry = calculateDaysUntilExpiry(cert.getValidTo());
        if (daysUntilExpiry > appConfig.getRenewal().getPreScanThresholdDays()) {
            result.setEligible(false);
            result.setReason(String.format(
                "Certificate is not expiring within %d days. Current days until expiry: %d",
                appConfig.getRenewal().getPreScanThresholdDays(), daysUntilExpiry
            ));
            return result;
        }
        
        // Check if certificate has expired
        if (cert.getValidTo().isBefore(LocalDateTime.now())) {
            result.setEligible(false);
            result.setReason("Certificate has already expired. Please contact support for emergency renewal.");
            return result;
        }
        
        // Check for unmerged PRs
        if (cert.getLastPullRequestUrl() != null && "OPEN".equals(cert.getPullRequestStatus())) {
            result.setEligible(true);
            result.setWarning("A pull request is already open for this certificate. " +
                            "New renewal will create another PR.");
        } else {
            result.setEligible(true);
        }
        
        return result;
    }
    
    private boolean isAlreadyRenewed(Certificate cert) {
        // Check if certificate has been renewed
        if ("COMPLETED".equals(cert.getRenewalStatus()) || "REPLACED".equals(cert.getRenewalStatus())) {
            return true;
        }
        
        // Check if there's a newer certificate with same reference ID
        if (cert.getReferenceId() != null) {
            List<Certificate> sameCsiCerts = certificateRepository.findByAppid(cert.getAppid());
            
            for (Certificate other : sameCsiCerts) {
                if (!other.getId().equals(cert.getId()) &&
                    cert.getReferenceId().equals(other.getReferenceId()) &&
                    other.getValidFrom() != null &&
                    cert.getValidTo() != null &&
                    other.getValidFrom().isAfter(cert.getValidFrom())) {
                    // Found a newer certificate with same reference ID
                    return true;
                }
            }
        }
        
        return false;
    }
    
    private void processCsiRenewals(Integer csiId, List<Certificate> certificates) {
        log.info("Processing {} renewals for CSI {}", certificates.size(), csiId);
        
        Optional<CsiMetadata> csiOpt = csiMetadataRepository.findByCsiId(csiId);
        if (csiOpt.isEmpty()) {
            log.error("CSI metadata not found for CSI {}", csiId);
            return;
        }
        
        CsiMetadata csi = csiOpt.get();
        
        // Send renewal initiation notification
        notificationService.sendRenewalInitiation(csi, certificates);
        
        for (Certificate cert : certificates) {
            try {
                renewCertificate(cert, csi);
            } catch (Exception e) {
                log.error("Failed to renew certificate {} for CSI {}", cert.getCn(), csiId, e);
                
                // Create failed renewal status
                RenewalStatus status = createRenewalStatus(cert, "FAILED", null);
                status.setLastError(e.getMessage());
                renewalStatusRepository.save(status);
                
                // Send failure notification
                notificationService.sendRenewalFailure(csi, cert, e.getMessage());
            }
        }
    }
    
    private void renewCertificate(Certificate cert, CsiMetadata csi) throws Exception {
        log.info("Renewing certificate: CN={}, CSI={}", cert.getCn(), cert.getAppid());
        
        // Create renewal status
        RenewalStatus renewalStatus = createRenewalStatus(cert, "INITIATED", null);
        renewalStatusRepository.save(renewalStatus);
        
        // Determine renewal strategy
        RenewalStrategy strategy = determineRenewalStrategy(cert, csi);
        log.info("Using renewal strategy: {} for certificate {}", strategy, cert.getCn());
        
        try {
            switch (strategy) {
                case STANDARD_RENEWAL:
                    performStandardRenewal(cert, csi, renewalStatus);
                    break;
                    
                case PROVIDER_MIGRATION:
                    performProviderMigration(cert, csi, renewalStatus);
                    break;
                    
                case AUTO_RENEWAL_WAIT:
                    handleAutoRenewalWait(cert, csi, renewalStatus);
                    break;
                    
                default:
                    throw new RuntimeException("Unknown renewal strategy: " + strategy);
            }
            
        } catch (Exception e) {
            // Handle renewal failure
            renewalStatus.setStatus("FAILED");
            renewalStatus.setLastError(e.getMessage());
            renewalStatusRepository.save(renewalStatus);
            throw e;
        }
    }
    
    private void performStandardRenewal(Certificate cert, CsiMetadata csi, RenewalStatus renewalStatus) 
            throws Exception {
        
        String renewalProvider = determineRenewalProvider(cert, csi);
        renewalStatus.setRenewalProvider(renewalProvider);
        
        CertificateProvider provider = getProvider(renewalProvider);
        Map<String, Object> renewalResult = provider.renewCertificate(cert, csi);
        
        // Update renewal status
        renewalStatus.setStatus("IN_PROGRESS");
        renewalStatus.setWorkflowId((String) renewalResult.get("workflowId"));
        renewalStatus.setNewOrderId((String) renewalResult.get("orderId"));
        renewalStatus.setProviderRequest((Map<String, Object>) renewalResult.get("request"));
        renewalStatus.setProviderResponse((Map<String, Object>) renewalResult.get("response"));
        renewalStatusRepository.save(renewalStatus);
        
        // Poll for completion
        boolean completed = pollForCompletion(provider, renewalStatus);
        
        if (completed) {
            handleSuccessfulRenewal(cert, csi, renewalStatus);
        } else {
            throw new RuntimeException("Certificate renewal timed out");
        }
    }
    
    private void performProviderMigration(Certificate cert, CsiMetadata csi, RenewalStatus renewalStatus) 
            throws Exception {
        
        log.info("Performing provider migration for certificate: {} from {} to {}", 
                cert.getCn(), cert.getOriginalProvider(), cert.getCurrentProvider());
        
        // For CMP to PP migration, onboard to Provisioning Portal
        if ("CMP".equals(cert.getOriginalProvider()) && 
            "PROVISIONING_PORTAL".equals(cert.getCurrentProvider())) {
            
            renewalStatus.setRenewalProvider("PROVISIONING_PORTAL");
            
            // Use Provisioning Portal to create new certificate
            Map<String, Object> renewalResult = provisioningPortalProvider.renewCertificate(cert, csi);
            
            renewalStatus.setStatus("IN_PROGRESS");
            renewalStatus.setWorkflowId((String) renewalResult.get("workflowId"));
            renewalStatus.setNewOrderId((String) renewalResult.get("orderId"));
            renewalStatus.setProviderRequest((Map<String, Object>) renewalResult.get("request"));
            renewalStatus.setProviderResponse((Map<String, Object>) renewalResult.get("response"));
            renewalStatusRepository.save(renewalStatus);
            
            // Poll for completion
            boolean completed = pollForCompletion(provisioningPortalProvider, renewalStatus);
            
            if (completed) {
                // Update certificate provider info
                cert.setOriginalProvider("PROVISIONING_PORTAL");
                cert.setIsAutoRenewalCandidate(true);
                certificateRepository.save(cert);
                
                handleSuccessfulRenewal(cert, csi, renewalStatus);
                
                // Send migration success notification
                notificationService.sendProviderMigrationSuccess(csi, cert, 
                    "CMP", "PROVISIONING_PORTAL");
            } else {
                throw new RuntimeException("Provider migration timed out");
            }
        } else {
            // For other migrations, use standard renewal
            performStandardRenewal(cert, csi, renewalStatus);
        }
    }
    
    private void handleAutoRenewalWait(Certificate cert, CsiMetadata csi, RenewalStatus renewalStatus) {
        log.info("Certificate {} is eligible for auto-renewal by provider", cert.getCn());
        
        renewalStatus.setStatus("PENDING_AUTO_RENEWAL");
        renewalStatus.setRenewalProvider(cert.getOriginalProvider());
        renewalStatusRepository.save(renewalStatus);
        
        // Update certificate status
        cert.setRenewalStatus("PENDING_AUTO_RENEWAL");
        certificateRepository.save(cert);
        
        // Send notification about auto-renewal
        notificationService.sendAutoRenewalPending(csi, cert);
    }
    
    private RenewalStrategy determineRenewalStrategy(Certificate cert, CsiMetadata csi) {
        // Check if certificate is auto-renewable and close to expiry
        if (Boolean.TRUE.equals(cert.getIsAutoRenewalCandidate()) &&
            ("PROVISIONING_PORTAL".equals(cert.getOriginalProvider()) || 
             "ACM".equals(cert.getOriginalProvider()))) {
            
            long daysUntilExpiry = calculateDaysUntilExpiry(cert.getValidTo());
            
            // If expiring in less than 30 days, let provider auto-renew
            if (daysUntilExpiry < 30) {
                return RenewalStrategy.AUTO_RENEWAL_WAIT;
            }
        }
        
        // Check if provider migration is needed
        String currentPreferredProvider = csi.getPreferredRenewalProvider() != null ?
            csi.getPreferredRenewalProvider() : appConfig.getRenewal().getDefaultProvider();
        
        if (cert.getOriginalProvider() != null && 
            !cert.getOriginalProvider().equals(currentPreferredProvider)) {
            // Migration needed
            return RenewalStrategy.PROVIDER_MIGRATION;
        }
        
        // Standard renewal
        return RenewalStrategy.STANDARD_RENEWAL;
    }
    
    private boolean pollForCompletion(CertificateProvider provider, RenewalStatus renewalStatus) {
        int maxAttempts = 20; // Poll for max 10 minutes (30 seconds interval)
        int attempts = 0;
        
        while (attempts < maxAttempts) {
            try {
                Thread.sleep(30000); // Wait 30 seconds between polls
                
                Map<String, Object> status = provider.checkStatus(
                    renewalStatus.getWorkflowId(), renewalStatus.getNewOrderId());
                
                String currentStatus = (String) status.get("status");
                
                if ("DEPLOYED".equals(currentStatus) || "COMPLETED".equals(currentStatus)) {
                    // Update change log
                    List<String> changeLog = (List<String>) status.get("changeLog");
                    if (changeLog != null) {
                        renewalStatus.setChangeLog(changeLog);
                    }
                    return true;
                } else if ("FAILED".equals(currentStatus) || "REJECTED".equals(currentStatus)) {
                    renewalStatus.setLastError("Provider returned status: " + currentStatus);
                    return false;
                }
                
                attempts++;
            } catch (Exception e) {
                log.error("Error polling for completion", e);
                attempts++;
            }
        }
        
        return false;
    }
    
    private void handleSuccessfulRenewal(Certificate cert, CsiMetadata csi, RenewalStatus renewalStatus) {
        log.info("Certificate renewal completed successfully: {}", cert.getCn());
        
        // Update certificate
        cert.setRenewalStatus("COMPLETED");
        cert.setNewOrderId(renewalStatus.getNewOrderId());
        cert.setLastRenewalCheck(LocalDateTime.now());
        certificateRepository.save(cert);
        
        // Update renewal status
        renewalStatus.setStatus("COMPLETED");
        renewalStatus.setCompletedAt(LocalDateTime.now());
        renewalStatusRepository.save(renewalStatus);
        
        // Update code if needed
        if (Boolean.TRUE.equals(cert.getHasWriteAccess()) && 
            csi.getFidUsername() != null &&
            cert.getCertificateLocations() != null && 
            !cert.getCertificateLocations().isEmpty()) {
            
            try {
                String prUrl = bitbucketService.updateCertificateReferences(
                    cert, csi, renewalStatus.getNewOrderId());
                
                renewalStatus.setPullRequestUrl(prUrl);
                renewalStatus.setCodeUpdateRequired(true);
                renewalStatusRepository.save(renewalStatus);
                
                // Update certificate PR tracking
                cert.setLastPullRequestUrl(prUrl);
                cert.setLastPullRequestDate(LocalDateTime.now());
                cert.setPullRequestStatus("OPEN");
                certificateRepository.save(cert);
                
                // Send PR notification
                notificationService.sendPullRequestCreated(csi, cert, prUrl);
                
            } catch (Exception e) {
                log.error("Failed to update code references", e);
                notificationService.sendManualActionRequired(csi, cert, renewalStatus.getNewOrderId());
            }
        } else {
            // Send manual action required notification
            notificationService.sendManualActionRequired(csi, cert, renewalStatus.getNewOrderId());
        }
        
        // Audit log
        auditLog("RENEWAL", "CERTIFICATE", cert.getId(), "SUCCESS", 
                Map.of("orderId", renewalStatus.getNewOrderId(),
                      "provider", renewalStatus.getRenewalProvider()));
    }
    
    private void retryFailedRenewals() {
        log.info("Processing failed renewals for retry");
        
        List<RenewalStatus> failedRenewals = renewalStatusRepository
            .findFailedRenewalsForRetry(appConfig.getRenewal().getMaxRetries());
        
        for (RenewalStatus renewal : failedRenewals) {
            try {
                Certificate cert = certificateRepository.findById(renewal.getCertificateId())
                    .orElseThrow(() -> new RuntimeException("Certificate not found"));
                
                CsiMetadata csi = csiMetadataRepository.findByCsiId(renewal.getCsiId())
                    .orElseThrow(() -> new RuntimeException("CSI not found"));
                
                renewal.setRetryCount(renewal.getRetryCount() + 1);
                renewal.setLastRetryAt(LocalDateTime.now());
                renewalStatusRepository.save(renewal);
                
                renewCertificate(cert, csi);
                
            } catch (Exception e) {
                log.error("Retry failed for renewal {}", renewal.getId(), e);
            }
        }
    }
    
    private String determineRenewalProvider(Certificate cert, CsiMetadata csi) {
        // Check if renewal provider is specified in certificate
        if (cert.getRenewalProvider() != null) {
            return cert.getRenewalProvider();
        }
        
        // Check CSI preference
        if (csi.getPreferredRenewalProvider() != null) {
            return csi.getPreferredRenewalProvider();
        }
        
        // Use default
        return appConfig.getRenewal().getDefaultProvider();
    }
    
    private CertificateProvider getProvider(String providerName) {
        switch (providerName) {
            case "PROVISIONING_PORTAL":
                return provisioningPortalProvider;
            case "CMP":
                return cmpProvider;
            default:
                throw new IllegalArgumentException("Unknown provider: " + providerName);
        }
    }
    
    private RenewalStatus createRenewalStatus(Certificate cert, String status, String provider) {
        RenewalStatus renewalStatus = new RenewalStatus();
        renewalStatus.setCertificateId(cert.getId());
        renewalStatus.setOldOrderId(cert.getTicketId());
        renewalStatus.setCsiId(cert.getAppid());
        renewalStatus.setCn(cert.getCn());
        renewalStatus.setEnvironment(cert.getEnvironment());
        renewalStatus.setStatus(status);
        renewalStatus.setRenewalProvider(provider);
        renewalStatus.setInitiatedAt(LocalDateTime.now());
        renewalStatus.setRetryCount(0);
        return renewalStatus;
    }
    
    private long calculateDaysUntilExpiry(LocalDateTime expiryDate) {
        return java.time.Duration.between(LocalDateTime.now(), expiryDate).toDays();
    }
    
    private void auditLog(String action, String entityType, String entityId, 
                         String status, Map<String, Object> details) {
        AuditLog log = new AuditLog();
        log.setAction(action);
        log.setEntityType(entityType);
        log.setEntityId(entityId);
        log.setStatus(status);
        log.setTimestamp(LocalDateTime.now());
        log.setDetails(details);
        log.setPerformedBy("SYSTEM");
        
        auditLogRepository.save(log);
    }
    
    // Test method for renewal process
    public void testRenewalProcess(String testScenario) {
        log.info("Running renewal test scenario: {}", testScenario);
        
        switch (testScenario) {
            case "standard":
                testStandardRenewal();
                break;
            case "migration":
                testProviderMigration();
                break;
            case "auto-renewal":
                testAutoRenewalDetection();
                break;
            case "validation":
                testRenewalValidation();
                break;
            default:
                log.warn("Unknown test scenario: {}", testScenario);
        }
    }
    
    private void testStandardRenewal() {
        log.info("Test: Standard renewal process");
        
        // Create mock certificate
        Certificate mockCert = new Certificate();
        mockCert.setId("test-cert-1");
        mockCert.setCn("test.example.com");
        mockCert.setAppid(166479);
        mockCert.setTicketId("API_C-old-order-id");
        mockCert.setValidTo(LocalDateTime.now().plusDays(45));
        mockCert.setOriginalProvider("CMP");
        mockCert.setCurrentProvider("CMP");
        
        // Create mock CSI
        CsiMetadata mockCsi = new CsiMetadata();
        mockCsi.setCsiId(166479);
        mockCsi.setPreferredRenewalProvider("CMP");
        
        log.info("Test: Certificate expires in {} days", calculateDaysUntilExpiry(mockCert.getValidTo()));
        
        // Validate renewal
        RenewalValidationResult validation = validateRenewalEligibility(mockCert);
        log.info("Test: Renewal eligible: {}, Reason: {}", validation.isEligible(), validation.getReason());
        
        // Determine strategy
        RenewalStrategy strategy = determineRenewalStrategy(mockCert, mockCsi);
        log.info("Test: Renewal strategy: {}", strategy);
    }
    
    private void testProviderMigration() {
        log.info("Test: Provider migration scenario");
        
        Certificate mockCert = new Certificate();
        mockCert.setCn("migrate.example.com");
        mockCert.setOriginalProvider("CMP");
        mockCert.setCurrentProvider("PROVISIONING_PORTAL");
        mockCert.setValidTo(LocalDateTime.now().plusDays(50));
        
        CsiMetadata mockCsi = new CsiMetadata();
        mockCsi.setPreferredRenewalProvider("PROVISIONING_PORTAL");
        
        RenewalStrategy strategy = determineRenewalStrategy(mockCert, mockCsi);
        log.info("Test: Migration needed from {} to {}, Strategy: {}", 
                mockCert.getOriginalProvider(), mockCert.getCurrentProvider(), strategy);
    }
    
    private void testAutoRenewalDetection() {
        log.info("Test: Auto-renewal detection");
        
        Certificate mockCert = new Certificate();
        mockCert.setCn("auto.example.com");
        mockCert.setOriginalProvider("PROVISIONING_PORTAL");
        mockCert.setIsAutoRenewalCandidate(true);
        mockCert.setValidTo(LocalDateTime.now().plusDays(25)); // Less than 30 days
        
        CsiMetadata mockCsi = new CsiMetadata();
        
        RenewalStrategy strategy = determineRenewalStrategy(mockCert, mockCsi);
        log.info("Test: Auto-renewal candidate expiring in {} days, Strategy: {}", 
                calculateDaysUntilExpiry(mockCert.getValidTo()), strategy);
    }
    
    private void testRenewalValidation() {
        log.info("Test: Renewal validation scenarios");
        
        // Test 1: Certificate not expiring soon
        Certificate cert1 = new Certificate();
        cert1.setValidTo(LocalDateTime.now().plusDays(120));
        cert1.setRenewalStatus("PENDING");
        
        RenewalValidationResult result1 = validateRenewalEligibility(cert1);
        log.info("Test 1 - Not expiring soon: Eligible={}, Reason={}", 
                result1.isEligible(), result1.getReason());
        
        // Test 2: Certificate already renewed
        Certificate cert2 = new Certificate();
        cert2.setValidTo(LocalDateTime.now().plusDays(30));
        cert2.setRenewalStatus("COMPLETED");
        
        RenewalValidationResult result2 = validateRenewalEligibility(cert2);
        log.info("Test 2 - Already renewed: Eligible={}, Reason={}", 
                result2.isEligible(), result2.getReason());
        
        // Test 3: Certificate with open PR
        Certificate cert3 = new Certificate();
        cert3.setValidTo(LocalDateTime.now().plusDays(40));
        cert3.setLastPullRequestUrl("https://bitbucket/pr/123");
        cert3.setPullRequestStatus("OPEN");
        
        RenewalValidationResult result3 = validateRenewalEligibility(cert3);
        log.info("Test 3 - Open PR exists: Eligible={}, Warning={}", 
                result3.isEligible(), result3.getWarning());
    }
    
    private enum RenewalStrategy {
        STANDARD_RENEWAL,
        PROVIDER_MIGRATION,
        AUTO_RENEWAL_WAIT
    }
    
    @Data
    private static class RenewalValidationResult {
        private boolean eligible = true;
        private String reason;
        private String warning;
    }
}


---------


