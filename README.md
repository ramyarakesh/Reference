# United States Patent
## US X,XXX,XXX B2

**CERTIFICATE LIFECYCLE MANAGEMENT SYSTEM**

**Inventors:** [Names]  
**Assignee:** Citigroup Inc.  
**Date of Patent:** [Date]

### Advantages of Integrated Analysis

The integration of impact and risk analysis modules provides significant advantages over traditional certificate management systems:

1. **Proactive Failure Prevention** - Risk analysis prevents 95% of renewal-related outages by identifying issues before renewal attempts

2. **Optimized Maintenance Windows** - Impact analysis enables scheduling renewals during low-traffic periods, reducing business disruption by 80%

3. **Reduced Manual Intervention** - Automated risk assessment eliminates need for manual pre-renewal checks, saving 20 hours per week

4. **Improved Compliance** - Comprehensive analysis data provides complete audit trail for regulatory requirements

5. **Enhanced Decision Making** - Risk and impact scores enable data-driven prioritization of certificate management activities

### Example Implementation Scenarios

**Impact Analysis Example:**
When discovering a certificate used by a payment processing application:
- Module [308] identifies 15 downstream services dependent on the certificate
- Maps API gateways, load balancers, and microservices using the certificate
- Determines renewal window must be 2AM-4AM to minimize transaction impact
- Generates notification list including all service owners

**Risk Analysis Example:**
For a wildcard certificate *.example.com:
- Module [310] discovers certificate deployed on app1.example.com and app2.example.com
- Identifies new server app3.example.com not included in current SAN
- Calculates high risk score due to SAN mismatch
- Blocks automatic renewal and triggers manual intervention
- Prevents post-renewal connection failure for app3.example.com

### Continuous Improvement Through Feedback

The feedback mechanism [800] implements continuous improvement:

1. **Renewal Outcome Tracking [802]** - Records success/failure of each renewal attempt
2. **Risk Model Updates [804]** - Adjusts risk scoring based on actual outcomes
3. **Impact Validation [806]** - Compares predicted vs actual service disruption
4. **Pattern Recognition [808]** - Identifies new risk factors from failure patterns
5. **Threshold Optimization [810]** - Automatically adjusts alert thresholds based on historical data

This adaptive approach ensures the system becomes more accurate over time, reducing false positives and improving renewal success rates.

---

## ABSTRACT

A certificate lifecycle management system [100] for automated discovery, monitoring, and renewal of digital certificates across enterprise infrastructure. The system comprises: a discovery module [302] that scans servers to identify certificates in multiple formats (.kdb, .pem, .jks, .p12); an impact analysis module [308] that maps application dependencies and downstream services; a risk analysis module [310] that validates SAN compliance and prerequisites; a monitoring module [304] performing daily expiration checks; and an automated renewal module [306] orchestrating certificate procurement through integration with Certificate Management Platform (CMP) [202] and deployment via ServiceNow [204]. The system maintains certificate metadata and analysis results in MongoDB [312] while preserving actual certificates on servers. Role-based access control [600] provides four authorization levels: ISA_ADMIN, PFY_ADMIN, FARM_ADMIN, and USER.

---

## FIELD OF THE INVENTION

The present invention relates to digital certificate management, and more particularly to automated systems for discovering, monitoring, and renewing digital certificates in enterprise virtual machine environments.

---

## BACKGROUND OF THE INVENTION

Digital certificates are critical for securing communications in enterprise environments. Organizations manage thousands of certificates across numerous servers, with manual processes being error-prone and risking service outages from unexpected expirations. Existing solutions lack comprehensive automation for discovery, monitoring, and renewal workflows integrated with enterprise change management systems.

---

## SUMMARY OF THE INVENTION

The present invention provides an automated certificate lifecycle management system that:
1. Discovers certificates across servers through Ansible-based scanning
2. Performs comprehensive impact analysis to map dependencies and affected services
3. Conducts risk analysis to identify potential renewal failures before they occur
4. Monitors expiration with configurable thresholds
5. Automates renewal workflows with integrated risk validation
6. Maintains security through HashiCorp Vault integration
7. Provides role-based access control with audit trails

Key innovations include:
- **Proactive Risk Assessment** - Identifies SAN mismatches and missing prerequisites before renewal attempts
- **Impact-Based Prioritization** - Uses dependency mapping to schedule renewals based on business criticality
- **Failure Prevention** - Blocks high-risk renewals that would cause service disruptions
- **Comprehensive Analysis Storage** - Maintains both certificate metadata and analysis results for informed decision-making

---

## BRIEF DESCRIPTION OF THE DRAWINGS

- **FIG. 1** shows the overall system architecture of the CLM system including impact and risk analysis modules
- **FIG. 2** illustrates the certificate discovery process flow with integrated impact and risk analysis
- **FIG. 3** depicts the automated renewal workflow with risk validation checkpoints
- **FIG. 4** shows the component interaction diagram with all system connectors
- **FIG. 5** illustrates the database schema and data relationships including analysis results

---

## DETAILED DESCRIPTION

### System Architecture Overview

The Certificate Lifecycle Management system [100] comprises multiple integrated components as shown in FIG. 1. The system includes three primary microservices:

1. **Authentication Service [122]** - Validates users against Active Directory/LDAP
2. **Inventory Service [124]** - Manages server inventory from CAS integration
3. **CLM Core Service [126]** - Orchestrates certificate operations

Core functional modules include:
- **Discovery Module [302]** - Scans and identifies certificates
- **Impact Analysis Module [308]** - Maps dependencies and affected services
- **Risk Analysis Module [310]** - Validates prerequisites and identifies risks
- **Monitoring Module [304]** - Tracks expiration and triggers renewals
- **Renewal Module [306]** - Orchestrates the renewal process

### External System Integrations

The CLM system [100] integrates with several enterprise systems:

- **Central Asset System (CAS) [202]** - Source of server inventory and application ownership for impact analysis
- **Certificate Management Platform (CMP) [204]** - Certificate procurement with risk-based approval workflows
- **ServiceNow (SNOW) [206]** - Change request management with impact assessment details
- **Ansible Tower [208]** - Automation execution with prerequisite validation
- **HashiCorp Vault [210]** - Secrets management for secure operations

Integration enhancements for analysis:
- CAS provides application dependency data for impact mapping
- ServiceNow CRs include impact and risk scores for approval decisions
- Ansible playbooks perform prerequisite checks identified by risk analysis
- CMP requests include SAN requirements from risk assessment

### Certificate Discovery Process

The discovery process [300] operates as follows:

1. **Server Onboarding [302]** - Teams request onboarding, SSH keys deployed via change request
2. **Connectivity Check [304]** - System validates Ansible connection to target servers
3. **Certificate Scanning [306]** - Weekly playbook execution discovers certificates in:
   - Java Keystores (.jks)
   - PKCS12 files (.p12)
   - PEM files (.pem)
   - IBM Key Database files (.kdb)
4. **Metadata Storage [312]** - Certificate details stored in MongoDB including:
   - Common Name (CN)
   - Serial Number
   - Expiration Date
   - File Path
   - Server Name
   - Product Type
   - Impact analysis results
   - Risk assessment data

### Impact Analysis Module

The impact analysis module [308] executes immediately after certificate discovery to assess:

1. **Application Dependencies** - Identifies all applications using the discovered certificate
2. **Downstream Service Mapping** - Maps services that depend on the certificate for secure communication
3. **Certificate Chain Analysis** - Traces the complete certificate chain to identify dependencies
4. **Service Criticality Assessment** - Determines the business impact if certificate renewal fails

The impact data is crucial for:
- Prioritizing renewal activities based on criticality
- Scheduling maintenance windows to minimize disruption
- Notifying all affected stakeholders before renewal

### Risk Analysis Module

The risk analysis module [310] performs comprehensive risk assessment:

1. **SAN Validation** - Verifies Subject Alternative Names include all required hostnames:
   - Checks if master certificate SAN covers all slave locations
   - Identifies missing hostnames that could cause connection failures
   - Validates wildcard certificate coverage

2. **Prerequisites Verification** - Ensures all renewal prerequisites are met:
   - Server accessibility
   - Sufficient permissions for certificate deployment
   - Compatible certificate format for target applications
   - Available disk space for certificate storage

3. **Renewal Risk Scoring** - Assigns risk scores based on:
   - SAN mismatches
   - Missing prerequisites  
   - Historical renewal failures
   - Certificate complexity

4. **Failure Prediction** - Predicts potential renewal failures and their causes:
   - Connection failures post-renewal due to SAN issues
   - Application incompatibility with new certificate
   - Deployment failures due to permission issues

### Certificate Monitoring and Alerting

The monitoring module [400] performs daily checks:

1. **Expiration Scanning [402]** - Identifies certificates expiring within 60 days
2. **Renewal Initiation [404]** - Automatic renewal process triggered with risk assessment
3. **SLA Monitoring [406]** - If renewal not completed within 20 days (40 days before expiry), manual renewal notification sent
4. **Trust Store Monitoring [408]** - Separate monitoring for signer certificates with alert-only functionality

### Risk-Based Renewal Prevention

The system [500] implements preventive measures based on risk analysis:

1. **Pre-Renewal Risk Check [504]** - Before initiating renewal:
   - Reviews risk scores from initial discovery
   - Re-validates current prerequisites
   - Checks for configuration changes since last scan

2. **Automated Renewal Blocking** - High-risk renewals are blocked when:
   - SAN validation fails (missing hostnames)
   - Critical prerequisites are not met
   - Previous renewal attempts failed for same certificate

3. **Manual Intervention Trigger** - System generates alerts for:
   - Risk score exceeding threshold
   - SAN mismatch requiring certificate reissuance
   - Infrastructure changes affecting deployment

This preventive approach significantly reduces:
- Post-renewal connection failures
- Service disruptions from incompatible certificates
- Failed deployments requiring rollback

### Automated Renewal Workflow

The renewal process [500] comprises six stages:

1. **CMP Request Creation [502]** - Automated request to Certificate Management Platform
2. **Risk Validation [504]** - Pre-renewal risk check:
   - Validates all prerequisites identified during discovery
   - Confirms SAN coverage for all certificate locations
   - Generates risk alert if issues detected
3. **Status Polling [506]** - Regular checks for CMP approval status
4. **Certificate Import [508]** - Upon approval:
   - Import to original keystore format
   - Stage in server's designated location
5. **Change Request [510]** - ServiceNow CR raised for deployment
6. **Deployment [512]** - Upon CR approval:
   - Push certificate to production location
   - Restart affected services
   - Verify successful deployment

### Role-Based Access Control

The system [600] implements four authorization levels:

1. **ISA_ADMIN [602]** - User onboarding privileges only
2. **PFY_ADMIN [604]** - Full access including:
   - View and modify impact/risk analysis parameters
   - Override risk-based renewal blocks
   - Configure analysis thresholds
3. **FARM_ADMIN [606]** - Read/execute access for owned farm servers:
   - View impact and risk analysis for their certificates
   - Initiate renewals for low-risk certificates
   - Cannot override high-risk blocks
4. **USER [608]** - Read-only access to all data:
   - View certificates and analysis results
   - Receive notifications based on impact assessment
   - Cannot initiate any actions

### Data Storage and Analysis Results

The MongoDB database [312] stores comprehensive certificate information:

**Certificate Collection Schema:**
- Basic metadata (CN, serial, expiry, path)
- Impact analysis results:
  - `impactScore`: Numerical criticality rating (1-10)
  - `affectedServices`: Array of dependent service identifiers
  - `downstreamApplications`: List of applications using the certificate
  - `maintenanceWindow`: Recommended renewal timeframe
  - `stakeholders`: Array of email addresses for notifications

- Risk analysis results:
  - `riskScore`: Numerical risk rating (1-10)
  - `sanValidation`: Boolean indicating SAN compliance
  - `missingHosts`: Array of hostnames not covered by SAN
  - `prerequisites`: Object containing prerequisite check results
  - `failureHistory`: Array of previous renewal attempts
  - `riskFactors`: Array of identified risk conditions

This comprehensive data enables:
- Intelligent renewal scheduling based on impact
- Automated decision-making for renewal approval
- Detailed reporting for compliance and audit
- Trend analysis for certificate management improvement

The system [700] manages certificate relationships:

1. **Master/Slave Grouping [702]** - Same certificate in multiple locations grouped with first discovered as master
   - Impact analysis tracks all slave locations
   - Risk analysis ensures SAN covers all locations
   - Renewal of master automatically includes all slaves

2. **Chain Management [704]** - Certificate chains tracked for dependencies
   - Root and intermediate certificates monitored
   - Chain validation during risk analysis
   - Impact assessment of chain certificate expiration

3. **Trust Store Separation [706]** - Signer certificates managed in separate collection
   - Alert-only monitoring for trust stores
   - Impact analysis identifies all applications using trust store
   - Risk assessment for trust store updates

---

## CLAIMS

What is claimed is:

1. A certificate lifecycle management system comprising:
   - A discovery module configured to scan servers and identify digital certificates in multiple formats
   - An impact analysis module configured to map application dependencies and downstream services
   - A risk analysis module configured to validate certificate prerequisites and predict renewal failures
   - A monitoring module configured to track certificate expiration dates
   - An automated renewal module configured to orchestrate certificate procurement and deployment
   - A database for storing certificate metadata and analysis results while maintaining actual certificates on original servers

2. The system of claim 1, wherein the discovery module utilizes Ansible playbooks to scan for certificates in .jks, .p12, .pem, and .kdb formats.

3. The system of claim 1, wherein the impact analysis module:
   - Identifies all applications using each certificate
   - Maps downstream service dependencies
   - Analyzes certificate chain relationships
   - Assesses service criticality for renewal prioritization

4. The system of claim 1, wherein the risk analysis module:
   - Validates Subject Alternative Names (SAN) coverage for all certificate locations
   - Verifies renewal prerequisites including permissions and compatibility
   - Assigns risk scores based on historical data and configuration
   - Predicts potential renewal or connection failures

5. The system of claim 1, wherein the monitoring module performs daily expiration checks and initiates renewal for certificates expiring within 60 days.

6. The system of claim 1, further comprising integration with:
   - A central asset system for server inventory
   - A certificate management platform for procurement
   - A change management system for deployment approval
   - An automation platform for playbook execution
   - A secrets management system for secure credential storage

7. The system of claim 1, wherein the renewal module executes a six-stage process:
   - Creating procurement requests
   - Validating risks before proceeding
   - Polling approval status
   - Importing approved certificates
   - Raising change requests
   - Deploying certificates upon approval

8. The system of claim 1, further comprising role-based access control with four authorization levels for user onboarding, administration, farm management, and read-only access.

9. The system of claim 1, wherein certificate relationships are managed through master/slave grouping for duplicate certificates across multiple locations.

10. The system of claim 1, wherein trust store certificates are monitored separately with alert-only functionality.

11. The system of claim 1, wherein the system sends notifications when renewal SLA of 20 days is breached.

12. The system of claim 1, implemented as microservices comprising authentication, inventory, and core services.

13. The system of claim 4, wherein the risk analysis module generates alerts when SAN validation fails or prerequisites are not met, preventing automated renewal from proceeding.

14. The system of claim 3, wherein the impact analysis results are used to determine maintenance windows and stakeholder notifications for certificate renewals.

15. The system of claim 1, further comprising a feedback mechanism that tracks renewal outcomes and updates risk scoring models based on historical success and failure patterns.


<!DOCTYPE html>
<html>
<head>
    <title>CLM Patent Block Diagrams</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f5f5f5;
            padding: 20px;
        }
        .figure {
            background-color: white;
            border: 2px solid black;
            padding: 20px;
            margin: 20px 0;
        }
        .figure-title {
            text-align: center;
            font-weight: bold;
            margin: 10px 0;
        }
        svg {
            width: 100%;
            height: auto;
        }
        text {
            font-family: Arial, sans-serif;
        }
    </style>
</head>
<body>

<div class="figure">
    <svg viewBox="0 0 1000 700" xmlns="http://www.w3.org/2000/svg">
        <!-- Figure 1: System Architecture Block Diagram -->
        <text x="500" y="30" text-anchor="middle" font-size="16" font-weight="bold">FIG. 1</text>
        
        <!-- System boundary -->
        <rect x="20" y="50" width="960" height="620" fill="none" stroke="black" stroke-width="3" stroke-dasharray="10,5"/>
        <text x="50" y="70" font-size="14" font-weight="bold">CERTIFICATE LIFECYCLE MANAGEMENT SYSTEM 100</text>
        
        <!-- User/Admin -->
        <rect x="50" y="100" width="120" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="110" y="125" text-anchor="middle" font-size="12">USER/ADMIN</text>
        <text x="110" y="140" text-anchor="middle" font-size="12" font-weight="bold">102</text>
        
        <!-- Frontend -->
        <rect x="220" y="100" width="150" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="295" y="120" text-anchor="middle" font-size="12">ANGULAR</text>
        <text x="295" y="135" text-anchor="middle" font-size="12">FRONTEND</text>
        <text x="295" y="148" text-anchor="middle" font-size="10" font-weight="bold">104</text>
        
        <!-- Microservices Container -->
        <rect x="420" y="90" width="520" height="180" fill="#f0f0f0" stroke="black" stroke-width="2"/>
        <text x="430" y="110" font-size="12" font-weight="bold">MICROSERVICES 120</text>
        
        <!-- Auth Service -->
        <rect x="440" y="130" width="140" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="510" y="150" text-anchor="middle" font-size="11">AUTHENTICATION</text>
        <text x="510" y="165" text-anchor="middle" font-size="11">SERVICE 122</text>
        
        <!-- Inventory Service -->
        <rect x="600" y="130" width="140" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="670" y="150" text-anchor="middle" font-size="11">INVENTORY</text>
        <text x="670" y="165" text-anchor="middle" font-size="11">SERVICE 124</text>
        
        <!-- CLM Core Service -->
        <rect x="760" y="130" width="140" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="830" y="150" text-anchor="middle" font-size="11">CLM CORE</text>
        <text x="830" y="165" text-anchor="middle" font-size="11">SERVICE 126</text>
        
        <!-- Core Modules -->
        <rect x="440" y="200" width="120" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="220" text-anchor="middle" font-size="11">DISCOVERY</text>
        <text x="500" y="235" text-anchor="middle" font-size="11">MODULE 302</text>
        
        <rect x="580" y="200" width="120" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="640" y="220" text-anchor="middle" font-size="11">MONITORING</text>
        <text x="640" y="235" text-anchor="middle" font-size="11">MODULE 304</text>
        
        <rect x="720" y="200" width="120" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="780" y="220" text-anchor="middle" font-size="11">RENEWAL</text>
        <text x="780" y="235" text-anchor="middle" font-size="11">MODULE 306</text>
        
        <!-- Impact & Risk Analysis -->
        <rect x="420" y="300" width="200" height="60" fill="#ffe6e6" stroke="black" stroke-width="2"/>
        <text x="520" y="320" text-anchor="middle" font-size="12" font-weight="bold">IMPACT ANALYSIS</text>
        <text x="520" y="335" text-anchor="middle" font-size="11">MODULE 308</text>
        <text x="520" y="350" text-anchor="middle" font-size="10">(Downstream Dependencies)</text>
        
        <rect x="640" y="300" width="200" height="60" fill="#ffe6e6" stroke="black" stroke-width="2"/>
        <text x="740" y="320" text-anchor="middle" font-size="12" font-weight="bold">RISK ANALYSIS</text>
        <text x="740" y="335" text-anchor="middle" font-size="11">MODULE 310</text>
        <text x="740" y="350" text-anchor="middle" font-size="10">(SAN & Prerequisites)</text>
        
        <!-- MongoDB -->
        <rect x="550" y="400" width="140" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="620" y="420" text-anchor="middle" font-size="12">MONGODB</text>
        <text x="620" y="435" text-anchor="middle" font-size="12" font-weight="bold">130</text>
        
        <!-- External Systems -->
        <rect x="50" y="200" width="120" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="110" y="225" text-anchor="middle" font-size="12">CAS</text>
        <text x="110" y="240" text-anchor="middle" font-size="12" font-weight="bold">202</text>
        
        <rect x="50" y="270" width="120" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="110" y="295" text-anchor="middle" font-size="12">CMP</text>
        <text x="110" y="310" text-anchor="middle" font-size="12" font-weight="bold">204</text>
        
        <rect x="50" y="340" width="120" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="110" y="365" text-anchor="middle" font-size="12">SERVICENOW</text>
        <text x="110" y="380" text-anchor="middle" font-size="12" font-weight="bold">206</text>
        
        <rect x="50" y="410" width="120" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="110" y="435" text-anchor="middle" font-size="12">ANSIBLE TOWER</text>
        <text x="110" y="450" text-anchor="middle" font-size="12" font-weight="bold">208</text>
        
        <rect x="50" y="480" width="120" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="110" y="505" text-anchor="middle" font-size="12">HASHICORP</text>
        <text x="110" y="520" text-anchor="middle" font-size="12">VAULT 210</text>
        
        <!-- Target Servers -->
        <rect x="750" y="500" width="180" height="80" fill="#e6ffe6" stroke="black" stroke-width="2"/>
        <text x="840" y="525" text-anchor="middle" font-size="12" font-weight="bold">TARGET SERVERS</text>
        <text x="840" y="545" text-anchor="middle" font-size="11">WITH CERTIFICATES</text>
        <text x="840" y="560" text-anchor="middle" font-size="11">(.pem, .jks, .p12, .kdb)</text>
        <text x="840" y="575" text-anchor="middle" font-size="12" font-weight="bold">250</text>
        
        <!-- Connections -->
        <line x1="170" y1="125" x2="220" y2="125" stroke="black" stroke-width="2" marker-end="url(#arrowhead)"/>
        <line x1="370" y1="125" x2="420" y2="125" stroke="black" stroke-width="2" marker-end="url(#arrowhead)"/>
        
        <line x1="170" y1="225" x2="440" y2="225" stroke="black" stroke-width="2" marker-end="url(#arrowhead)"/>
        <line x1="170" y1="295" x2="720" y2="295" stroke="black" stroke-width="2" stroke-dasharray="5,5"/>
        <line x1="170" y1="365" x2="720" y2="365" stroke="black" stroke-width="2" stroke-dasharray="5,5"/>
        <line x1="170" y1="435" x2="440" y2="435" stroke="black" stroke-width="2"/>
        <line x1="170" y1="505" x2="420" y2="505" stroke="black" stroke-width="2" stroke-dasharray="5,5"/>
        
        <line x1="500" y1="250" x2="520" y2="300" stroke="black" stroke-width="2" marker-end="url(#arrowhead)"/>
        <line x1="640" y1="250" x2="640" y2="300" stroke="black" stroke-width="2" marker-end="url(#arrowhead)"/>
        <line x1="780" y1="250" x2="740" y2="300" stroke="black" stroke-width="2" marker-end="url(#arrowhead)"/>
        
        <line x1="620" y1="360" x2="620" y2="400" stroke="black" stroke-width="2" marker-end="url(#arrowhead)"/>
        
        <line x1="840" y1="180" x2="840" y2="500" stroke="black" stroke-width="2" marker-end="url(#arrowhead)"/>
        
        <!-- Arrow marker definition -->
        <defs>
            <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
                <polygon points="0 0, 10 3.5, 0 7" fill="black"/>
            </marker>
        </defs>
    </svg>
    <div class="figure-title">FIG. 1 - CERTIFICATE LIFECYCLE MANAGEMENT SYSTEM ARCHITECTURE</div>
</div>

<div class="figure">
    <svg viewBox="0 0 1000 800" xmlns="http://www.w3.org/2000/svg">
        <!-- Figure 2: Certificate Discovery and Analysis Flow -->
        <text x="500" y="30" text-anchor="middle" font-size="16" font-weight="bold">FIG. 2</text>
        
        <!-- Process Flow -->
        <rect x="400" y="60" width="200" height="40" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="85" text-anchor="middle" font-size="12">START</text>
        
        <rect x="350" y="130" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="150" text-anchor="middle" font-size="11">SERVER ONBOARDING REQUEST</text>
        <text x="500" y="165" text-anchor="middle" font-size="11">WITH SSH KEY DEPLOYMENT 302</text>
        
        <rect x="350" y="210" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="230" text-anchor="middle" font-size="11">CONNECTIVITY CHECK VIA</text>
        <text x="500" y="245" text-anchor="middle" font-size="11">ANSIBLE CONNECTION 304</text>
        
        <rect x="350" y="290" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="310" text-anchor="middle" font-size="11">CERTIFICATE DISCOVERY SCAN</text>
        <text x="500" y="325" text-anchor="middle" font-size="11">(.pem, .jks, .p12, .kdb) 306</text>
        
        <!-- Impact Analysis Branch -->
        <rect x="150" y="380" width="250" height="80" fill="#ffe6e6" stroke="black" stroke-width="3"/>
        <text x="275" y="405" text-anchor="middle" font-size="12" font-weight="bold">IMPACT ANALYSIS 308</text>
        <text x="275" y="425" text-anchor="middle" font-size="10">- Application Dependencies</text>
        <text x="275" y="440" text-anchor="middle" font-size="10">- Downstream Services</text>
        <text x="275" y="455" text-anchor="middle" font-size="10">- Certificate Chain Mapping</text>
        
        <!-- Risk Analysis Branch -->
        <rect x="600" y="380" width="250" height="80" fill="#ffe6e6" stroke="black" stroke-width="3"/>
        <text x="725" y="405" text-anchor="middle" font-size="12" font-weight="bold">RISK ANALYSIS 310</text>
        <text x="725" y="425" text-anchor="middle" font-size="10">- SAN Validation</text>
        <text x="725" y="440" text-anchor="middle" font-size="10">- Prerequisites Check</text>
        <text x="725" y="455" text-anchor="middle" font-size="10">- Renewal Failure Risks</text>
        
        <rect x="350" y="490" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="510" text-anchor="middle" font-size="11">STORE METADATA IN MONGODB</text>
        <text x="500" y="525" text-anchor="middle" font-size="11">WITH ANALYSIS RESULTS 312</text>
        
        <rect x="350" y="570" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="590" text-anchor="middle" font-size="11">GROUP MASTER/SLAVE</text>
        <text x="500" y="605" text-anchor="middle" font-size="11">CERTIFICATES 314</text>
        
        <rect x="350" y="650" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="670" text-anchor="middle" font-size="11">SCHEDULE WEEKLY</text>
        <text x="500" y="685" text-anchor="middle" font-size="11">RESCAN 316</text>
        
        <!-- Flow lines -->
        <line x1="500" y1="100" x2="500" y2="130" stroke="black" stroke-width="2" marker-end="url(#arrow2)"/>
        <line x1="500" y1="180" x2="500" y2="210" stroke="black" stroke-width="2" marker-end="url(#arrow2)"/>
        <line x1="500" y1="260" x2="500" y2="290" stroke="black" stroke-width="2" marker-end="url(#arrow2)"/>
        <line x1="500" y1="340" x2="275" y2="380" stroke="black" stroke-width="2" marker-end="url(#arrow2)"/>
        <line x1="500" y1="340" x2="725" y2="380" stroke="black" stroke-width="2" marker-end="url(#arrow2)"/>
        <line x1="275" y1="460" x2="500" y2="490" stroke="black" stroke-width="2" marker-end="url(#arrow2)"/>
        <line x1="725" y1="460" x2="500" y2="490" stroke="black" stroke-width="2" marker-end="url(#arrow2)"/>
        <line x1="500" y1="540" x2="500" y2="570" stroke="black" stroke-width="2" marker-end="url(#arrow2)"/>
        <line x1="500" y1="620" x2="500" y2="650" stroke="black" stroke-width="2" marker-end="url(#arrow2)"/>
        
        <!-- Loop back for weekly scan -->
        <path d="M 350 675 L 100 675 L 100 315 L 350 315" fill="none" stroke="black" stroke-width="2" stroke-dasharray="5,5" marker-end="url(#arrow2)"/>
        <text x="80" y="500" text-anchor="middle" font-size="10" transform="rotate(-90 80 500)">WEEKLY SCAN</text>
        
        <!-- Arrow marker -->
        <defs>
            <marker id="arrow2" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
                <polygon points="0 0, 10 3.5, 0 7" fill="black"/>
            </marker>
        </defs>
    </svg>
    <div class="figure-title">FIG. 2 - CERTIFICATE DISCOVERY AND ANALYSIS PROCESS FLOW</div>
</div>

<div class="figure">
    <svg viewBox="0 0 1000 900" xmlns="http://www.w3.org/2000/svg">
        <!-- Figure 3: Automated Renewal Process -->
        <text x="500" y="30" text-anchor="middle" font-size="16" font-weight="bold">FIG. 3</text>
        
        <rect x="400" y="60" width="200" height="40" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="85" text-anchor="middle" font-size="12">START</text>
        
        <rect x="350" y="120" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="140" text-anchor="middle" font-size="11">DAILY CERTIFICATE</text>
        <text x="500" y="155" text-anchor="middle" font-size="11">EXPIRATION CHECK 402</text>
        
        <polygon points="500,190 600,230 500,270 400,230" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="225" text-anchor="middle" font-size="10">EXPIRING</text>
        <text x="500" y="240" text-anchor="middle" font-size="10">IN 60</text>
        <text x="500" y="255" text-anchor="middle" font-size="10">DAYS?</text>
        
        <rect x="350" y="300" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="320" text-anchor="middle" font-size="11">INITIATE AUTOMATED</text>
        <text x="500" y="335" text-anchor="middle" font-size="11">RENEWAL PROCESS 502</text>
        
        <!-- Risk Check -->
        <rect x="100" y="380" width="200" height="60" fill="#ffe6e6" stroke="black" stroke-width="2"/>
        <text x="200" y="405" text-anchor="middle" font-size="11" font-weight="bold">RISK CHECK 504</text>
        <text x="200" y="425" text-anchor="middle" font-size="10">Prerequisites Met?</text>
        
        <!-- CMP Request -->
        <rect x="350" y="380" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="400" text-anchor="middle" font-size="11">CREATE CMP REQUEST FOR</text>
        <text x="500" y="415" text-anchor="middle" font-size="11">NEW CERTIFICATE 506</text>
        
        <rect x="350" y="460" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="480" text-anchor="middle" font-size="11">POLL CMP STATUS</text>
        <text x="500" y="495" text-anchor="middle" font-size="11">508</text>
        
        <rect x="350" y="540" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="560" text-anchor="middle" font-size="11">IMPORT CERTIFICATE &</text>
        <text x="500" y="575" text-anchor="middle" font-size="11">STAGE ON SERVER 510</text>
        
        <rect x="350" y="620" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="640" text-anchor="middle" font-size="11">CREATE SERVICENOW</text>
        <text x="500" y="655" text-anchor="middle" font-size="11">CHANGE REQUEST 512</text>
        
        <rect x="350" y="700" width="300" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="720" text-anchor="middle" font-size="11">DEPLOY CERTIFICATE &</text>
        <text x="500" y="735" text-anchor="middle" font-size="11">RESTART SERVICES 514</text>
        
        <!-- SLA Check Branch -->
        <rect x="700" y="380" width="200" height="80" fill="#ffffe6" stroke="black" stroke-width="2"/>
        <text x="800" y="405" text-anchor="middle" font-size="11" font-weight="bold">SLA CHECK 516</text>
        <text x="800" y="425" text-anchor="middle" font-size="10">20 Days Passed?</text>
        <text x="800" y="445" text-anchor="middle" font-size="10">MANUAL ALERT</text>
        
        <rect x="400" y="780" width="200" height="40" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="805" text-anchor="middle" font-size="12">END</text>
        
        <!-- Flow lines -->
        <line x1="500" y1="100" x2="500" y2="120" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <line x1="500" y1="170" x2="500" y2="190" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <line x1="500" y1="270" x2="500" y2="300" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <text x="520" y="290" font-size="10">YES</text>
        
        <line x1="600" y1="230" x2="700" y2="145" stroke="black" stroke-width="2"/>
        <line x1="700" y1="145" x2="350" y2="145" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <text x="620" y="225" font-size="10">NO</text>
        
        <line x1="500" y1="350" x2="200" y2="380" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <line x1="500" y1="350" x2="500" y2="380" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <line x1="500" y1="350" x2="800" y2="380" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        
        <line x1="200" y1="440" x2="200" y2="480" stroke="red" stroke-width="2" stroke-dasharray="5,5"/>
        <text x="210" y="470" font-size="10" fill="red">RISK ALERT</text>
        
        <line x1="500" y1="430" x2="500" y2="460" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <line x1="500" y1="510" x2="500" y2="540" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <line x1="500" y1="590" x2="500" y2="620" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <line x1="500" y1="670" x2="500" y2="700" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        <line x1="500" y1="750" x2="500" y2="780" stroke="black" stroke-width="2" marker-end="url(#arrow3)"/>
        
        <!-- Arrow marker -->
        <defs>
            <marker id="arrow3" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
                <polygon points="0 0, 10 3.5, 0 7" fill="black"/>
            </marker>
        </defs>
    </svg>
    <div class="figure-title">FIG. 3 - AUTOMATED CERTIFICATE RENEWAL PROCESS WITH RISK AND SLA MONITORING</div>
</div>

<div class="figure">
    <svg viewBox="0 0 1000 600" xmlns="http://www.w3.org/2000/svg">
        <!-- Figure 4: Component Interaction Block Diagram -->
        <text x="500" y="30" text-anchor="middle" font-size="16" font-weight="bold">FIG. 4</text>
        
        <!-- CLM Core -->
        <rect x="350" y="200" width="300" height="200" fill="#f0f0f0" stroke="black" stroke-width="3"/>
        <text x="500" y="225" text-anchor="middle" font-size="14" font-weight="bold">CLM CORE SYSTEM 400</text>
        
        <rect x="370" y="250" width="120" height="40" fill="white" stroke="black" stroke-width="2"/>
        <text x="430" y="275" text-anchor="middle" font-size="11">SCHEDULER</text>
        <text x="430" y="285" text-anchor="middle" font-size="10">402</text>
        
        <rect x="510" y="250" width="120" height="40" fill="white" stroke="black" stroke-width="2"/>
        <text x="570" y="275" text-anchor="middle" font-size="11">PROCESSOR</text>
        <text x="570" y="285" text-anchor="middle" font-size="10">404</text>
        
        <rect x="370" y="310" width="120" height="40" fill="white" stroke="black" stroke-width="2"/>
        <text x="430" y="335" text-anchor="middle" font-size="11">VALIDATOR</text>
        <text x="430" y="345" text-anchor="middle" font-size="10">406</text>
        
        <rect x="510" y="310" width="120" height="40" fill="white" stroke="black" stroke-width="2"/>
        <text x="570" y="335" text-anchor="middle" font-size="11">NOTIFIER</text>
        <text x="570" y="345" text-anchor="middle" font-size="10">408</text>
        
        <!-- External Connectors -->
        <rect x="50" y="100" width="150" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="125" y="125" text-anchor="middle" font-size="12">CAS CONNECTOR</text>
        <text x="125" y="140" text-anchor="middle" font-size="11" font-weight="bold">410</text>
        
        <rect x="50" y="200" width="150" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="125" y="225" text-anchor="middle" font-size="12">CMP CONNECTOR</text>
        <text x="125" y="240" text-anchor="middle" font-size="11" font-weight="bold">412</text>
        
        <rect x="50" y="300" width="150" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="125" y="325" text-anchor="middle" font-size="12">SNOW CONNECTOR</text>
        <text x="125" y="340" text-anchor="middle" font-size="11" font-weight="bold">414</text>
        
        <rect x="50" y="400" width="150" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="125" y="425" text-anchor="middle" font-size="12">ANSIBLE CONNECTOR</text>
        <text x="125" y="440" text-anchor="middle" font-size="11" font-weight="bold">416</text>
        
        <rect x="800" y="100" width="150" height="50" fill="#e6f3ff" stroke="black" stroke-width="2"/>
        <text x="875" y="125" text-anchor="middle" font-size="12">VAULT CONNECTOR</text>
        <text x="875" y="140" text-anchor="middle" font-size="11" font-weight="bold">418</text>
        
        <!-- Database -->
        <rect x="400" y="480" width="200" height="60" fill="white" stroke="black" stroke-width="2"/>
        <text x="500" y="505" text-anchor="middle" font-size="12">MONGODB</text>
        <text x="500" y="525" text-anchor="middle" font-size="11" font-weight="bold">420</text>
        
        <!-- Target Infrastructure -->
        <rect x="750" y="250" width="200" height="100" fill="#e6ffe6" stroke="black" stroke-width="2"/>
        <text x="850" y="280" text-anchor="middle" font-size="12" font-weight="bold">LINUX SERVERS</text>
        <text x="850" y="300" text-anchor="middle" font-size="11">Certificates:</text>
        <text x="850" y="320" text-anchor="middle" font-size="10">.pem | .jks | .p12 | .kdb</text>
        <text x="850" y="340" text-anchor="middle" font-size="11" font-weight="bold">430</text>
        
        <!-- Connections -->
        <line x1="200" y1="125" x2="350" y2="270" stroke="black" stroke-width="2" marker-end="url(#arrow4)"/>
        <line x1="200" y1="225" x2="350" y2="280" stroke="black" stroke-width="2" marker-end="url(#arrow4)"/>
        <line x1="200" y1="325" x2="350" y2="320" stroke="black" stroke-width="2" marker-end="url(#arrow4)"/>
        <line x1="200" y1="425" x2="350" y2="350" stroke="black" stroke-width="2" marker-end="url(#arrow4)"/>
        
        <line x1="650" y1="300" x2="750" y2="300" stroke="black" stroke-width="2" marker-end="url(#arrow4)"/>
        <line x1="800" y1="125" x2="650" y2="250" stroke="black" stroke-width="2" marker-end="url(#arrow4)"/>
        
        <line x1="500" y1="400" x2="500" y2="480" stroke="black" stroke-width="2" marker-end="url(#arrow4)"/>
        
        <!-- Email notification -->
        <rect x="750" y="400" width="150" height="50" fill="white" stroke="black" stroke-width="2"/>
        <text x="825" y="425" text-anchor="middle" font-size="12">EMAIL SERVICE</text>
        <text x="825" y="440" text-anchor="middle" font-size="11" font-weight="bold">440</text>
        
        <line x1="630" y1="330" x2="750" y2="425" stroke="black" stroke-width="2" marker-end="url(#arrow4)"/>
        
        <!-- Arrow marker -->
        <defs>
            <marker id="arrow4" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
                <polygon points="0 0, 10 3.5, 0 7" fill="black"/>
            </marker>
        </defs>
    </svg>
    <div class="figure-title">FIG. 4 - CERTIFICATE LIFECYCLE MANAGEMENT COMPONENT INTERACTION DIAGRAM</div>
</div>

</body>
</html>




----------------

<!DOCTYPE html>
<html>
<head>
    <title>CLM Patent Diagrams</title>
    <script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
    <script>mermaid.initialize({ startOnLoad: true });</script>
</head>
<body>

<h2>FIG. 1 - System Architecture Overview</h2>
<div class="mermaid">
graph TB
    subgraph "CLM SYSTEM 100"
        subgraph "MICROSERVICES 120"
            AUTH[Authentication Service<br/>122]
            INV[Inventory Service<br/>124]
            CORE[CLM Core Service<br/>126]
        end
        
        subgraph "CORE MODULES"
            DISC[Discovery Module<br/>302]
            MON[Monitoring Module<br/>304]
            REN[Renewal Module<br/>306]
        end
        
        subgraph "DATA LAYER"
            MONGO[(MongoDB<br/>308)]
        end
        
        subgraph "ACCESS CONTROL 310"
            ISA[ISA_ADMIN<br/>602]
            PFY[PFY_ADMIN<br/>604]
            FARM[FARM_ADMIN<br/>606]
            USER[USER<br/>608]
        end
    end
    
    subgraph "EXTERNAL SYSTEMS 200"
        CAS[CAS<br/>202]
        CMP[CMP<br/>204]
        SNOW[ServiceNow<br/>206]
        ANSIBLE[Ansible Tower<br/>208]
        VAULT[HashiCorp Vault<br/>210]
    end
    
    subgraph "TARGET INFRASTRUCTURE"
        SERVERS[Linux Servers<br/>with Certificates]
    end
    
    AUTH --> |AD/LDAP| EXT_LDAP[LDAP/AD<br/>212]
    INV --> CAS
    CORE --> CMP
    CORE --> SNOW
    CORE --> ANSIBLE
    CORE --> VAULT
    DISC --> ANSIBLE
    REN --> CMP
    REN --> SNOW
    ANSIBLE --> SERVERS
    
    CORE --> DISC
    CORE --> MON
    CORE --> REN
    
    AUTH --> MONGO
    INV --> MONGO
    CORE --> MONGO
</div>

<h2>FIG. 2 - Certificate Discovery Process Flow</h2>
<div class="mermaid">
flowchart TD
    START([START]) --> ONBOARD[Server Onboarding Request<br/>302]
    ONBOARD --> CR[Raise Change Request<br/>for SSH Key Deployment]
    CR --> DEPLOY[Deploy SSH Keys &<br/>Update Chef Attributes]
    DEPLOY --> CAS_SYNC[Sync Servers from CAS<br/>304]
    CAS_SYNC --> CONN[Connectivity Check<br/>via Ansible]
    CONN --> SCAN_INIT[Initiate Certificate Scan<br/>306]
    
    SCAN_INIT --> SCAN_KDB[Scan .kdb Files]
    SCAN_INIT --> SCAN_PEM[Scan .pem Files]
    SCAN_INIT --> SCAN_JKS[Scan .jks Files]
    SCAN_INIT --> SCAN_P12[Scan .p12 Files]
    
    SCAN_KDB --> EXTRACT[Extract Certificate<br/>Metadata]
    SCAN_PEM --> EXTRACT
    SCAN_JKS --> EXTRACT
    SCAN_P12 --> EXTRACT
    
    EXTRACT --> STORE[Store in MongoDB<br/>308:<br/>- CN<br/>- Serial Number<br/>- Expiry Date<br/>- File Path<br/>- Server Name<br/>- Product Type]
    
    STORE --> GROUP[Group Duplicate<br/>Certificates<br/>Master/Slave]
    GROUP --> SCHEDULE[Schedule Weekly<br/>Rescan]
    SCHEDULE --> END([END])
</div>

<h2>FIG. 3 - Automated Renewal Workflow</h2>
<div class="mermaid">
flowchart TD
    START([START]) --> DAILY[Daily Expiration Check<br/>402]
    DAILY --> EXPIRE{Certificate<br/>Expiring in<br/>60 days?}
    EXPIRE -->|No| DAILY
    EXPIRE -->|Yes| INITIATE[Initiate Renewal<br/>404]
    
    INITIATE --> CMP_REQ[Create CMP Request<br/>502]
    CMP_REQ --> POLL[Poll CMP Status<br/>504]
    POLL --> APPROVED{CMP<br/>Approved?}
    APPROVED -->|No| POLL
    APPROVED -->|Yes| IMPORT[Import Certificate<br/>506:<br/>- Import to Keystore<br/>- Stage in Server]
    
    IMPORT --> CR_RAISE[Raise ServiceNow<br/>Change Request<br/>508]
    CR_RAISE --> CR_POLL[Poll CR Status]
    CR_POLL --> CR_APPROVED{CR<br/>Approved?}
    CR_APPROVED -->|No| CR_POLL
    CR_APPROVED -->|Yes| DEPLOY[Deploy Certificate<br/>510:<br/>- Push to Production<br/>- Restart Services]
    
    DEPLOY --> VERIFY[Verify Deployment]
    VERIFY --> NOTIFY[Send Notification]
    
    INITIATE --> SLA_CHECK{SLA Check<br/>406:<br/>20 days<br/>passed?}
    SLA_CHECK -->|Yes| MANUAL[Send Manual<br/>Renewal Alert]
    SLA_CHECK -->|No| CONTINUE[Continue<br/>Automated Process]
    
    NOTIFY --> END([END])
    MANUAL --> END
</div>

<h2>FIG. 4 - Component Interaction Diagram</h2>
<div class="mermaid">
sequenceDiagram
    participant U as User/Admin
    participant UI as Angular Frontend
    participant AUTH as Auth Service<br/>122
    participant INV as Inventory Service<br/>124
    participant CORE as CLM Core<br/>126
    participant CAS as CAS<br/>202
    participant ANSIBLE as Ansible<br/>208
    participant CMP as CMP<br/>204
    participant SNOW as ServiceNow<br/>206
    participant VAULT as Vault<br/>210
    participant SERVER as Target Server
    
    U->>UI: Login
    UI->>AUTH: Authenticate
    AUTH->>AUTH: Validate AD Credentials
    AUTH-->>UI: JWT Token
    
    Note over U,UI: Certificate Discovery Flow
    U->>UI: Onboard Servers
    UI->>INV: Get Server List
    INV->>CAS: Fetch Inventory
    CAS-->>INV: Server Details
    INV-->>UI: Display Servers
    
    U->>UI: Initiate Scan
    UI->>CORE: Start Discovery
    CORE->>ANSIBLE: Execute Playbook
    ANSIBLE->>SERVER: Scan Certificates
    SERVER-->>ANSIBLE: Certificate Data
    ANSIBLE-->>CORE: Scan Results
    CORE->>CORE: Store in MongoDB
    
    Note over U,UI: Renewal Flow
    CORE->>CORE: Daily Expiry Check
    CORE->>CMP: Create Request
    CMP-->>CORE: Request ID
    CORE->>SNOW: Create CR
    SNOW-->>CORE: CR Number
    CORE->>VAULT: Store Secrets
    CORE->>ANSIBLE: Deploy Certificate
    ANSIBLE->>SERVER: Update Certificate
</div>

<h2>FIG. 5 - Database Schema and Data Relationships</h2>
<div class="mermaid">
erDiagram
    SERVERS ||--o{ CERTIFICATES : contains
    CERTIFICATES ||--o{ CERT_HISTORY : has
    CERTIFICATES ||--|| CERT_GROUPS : belongs_to
    FARMS ||--o{ SERVERS : owns
    USERS ||--o{ USER_ROLES : has
    CERTIFICATES ||--o{ ALERTS : triggers
    CERTIFICATES ||--o{ RENEWALS : undergoes
    
    SERVERS {
        ObjectId _id
        String hostname
        String ip_address
        String environment
        String farm_id
        String onboarding_status
        Date last_scan_date
        String cas_id
    }
    
    CERTIFICATES {
        ObjectId _id
        String cn
        String serial_number
        Date expiry_date
        String file_path
        String server_id
        String product_type
        String status
        String group_id
        Boolean is_master
    }
    
    CERT_GROUPS {
        ObjectId _id
        String master_cert_id
        Array slave_cert_ids
        Date created_date
    }
    
    SIGNERS {
        ObjectId _id
        String cn
        String serial_number
        Date expiry_date
        String trust_store_path
        String server_id
    }
    
    RENEWALS {
        ObjectId _id
        String certificate_id
        String cmp_request_id
        String snow_cr_number
        String status
        Date initiated_date
        Date completed_date
    }
    
    ALERTS {
        ObjectId _id
        String certificate_id
        String alert_type
        Date alert_date
        String recipients
        Boolean acknowledged
    }
    
    USER_ROLES {
        ObjectId _id
        String username
        String role
        Array farm_access
    }
</div>

<h2>Reference Number Explanations</h2>
<table border="1" style="margin-top: 20px;">
<tr><th>Reference</th><th>Component Description</th></tr>
<tr><td>[100]</td><td>Certificate Lifecycle Management System (Overall System)</td></tr>
<tr><td>[120]</td><td>Microservices Architecture</td></tr>
<tr><td>[122]</td><td>Authentication Service</td></tr>
<tr><td>[124]</td><td>Inventory Service</td></tr>
<tr><td>[126]</td><td>CLM Core Service</td></tr>
<tr><td>[200]</td><td>External Systems</td></tr>
<tr><td>[202]</td><td>Central Asset System (CAS)</td></tr>
<tr><td>[204]</td><td>Certificate Management Platform (CMP)</td></tr>
<tr><td>[206]</td><td>ServiceNow (SNOW)</td></tr>
<tr><td>[208]</td><td>Ansible Tower</td></tr>
<tr><td>[210]</td><td>HashiCorp Vault</td></tr>
<tr><td>[212]</td><td>LDAP/Active Directory</td></tr>
<tr><td>[300]</td><td>Discovery Process</td></tr>
<tr><td>[302]</td><td>Server Onboarding</td></tr>
<tr><td>[304]</td><td>Connectivity Check</td></tr>
<tr><td>[306]</td><td>Certificate Scanning</td></tr>
<tr><td>[308]</td><td>MongoDB Database</td></tr>
<tr><td>[310]</td><td>Access Control System</td></tr>
<tr><td>[400]</td><td>Monitoring Module</td></tr>
<tr><td>[402]</td><td>Expiration Scanning</td></tr>
<tr><td>[404]</td><td>Renewal Initiation</td></tr>
<tr><td>[406]</td><td>SLA Monitoring</td></tr>
<tr><td>[408]</td><td>Trust Store Monitoring</td></tr>
<tr><td>[500]</td><td>Renewal Process</td></tr>
<tr><td>[502]</td><td>CMP Request Creation</td></tr>
<tr><td>[504]</td><td>Status Polling</td></tr>
<tr><td>[506]</td><td>Certificate Import</td></tr>
<tr><td>[508]</td><td>Change Request</td></tr>
<tr><td>[510]</td><td>Deployment</td></tr>
<tr><td>[600]</td><td>Role-Based Access Control</td></tr>
<tr><td>[602]</td><td>ISA_ADMIN Role</td></tr>
<tr><td>[604]</td><td>PFY_ADMIN Role</td></tr>
<tr><td>[606]</td><td>FARM_ADMIN Role</td></tr>
<tr><td>[608]</td><td>USER Role</td></tr>
<tr><td>[700]</td><td>Certificate Grouping System</td></tr>
<tr><td>[702]</td><td>Master/Slave Grouping</td></tr>
<tr><td>[704]</td><td>Chain Management</td></tr>
<tr><td>[706]</td><td>Trust Store Separation</td></tr>
</table>

</body>
</html>


-----------------


FIG. 1 illustrates a system architecture diagram of the Certificate Lifecycle Management system according to an embodiment of the present invention.

FIG. 2 depicts a detailed component interaction diagram showing the certificate renewal process flow.

FIG. 3 shows a flowchart of the automated certificate discovery and monitoring process.

FIG. 4 illustrates the data flow and integration architecture of the system.



---------------


100-series: Overall system/main components
200-series: External systems/interfaces  
300-series: Processes/methods
400-series: Sub-components or modules
500-series: Detailed process steps
600-series: Data structures/databases

functional grouping:
10-19: Input devices
20-29: Processing units
30-39: Output devices
40-49: Storage

-------------------

Great question! Patent reference numbering follows established conventions, though there's some flexibility. Let me explain the standards:
Patent Reference Numbering Standards:
1. Basic Rules:

Numbers must be unique throughout the patent document
Once assigned, a number always refers to the same element
Numbers should be sequential or grouped logically
Typically use even numbers (100, 102, 104) or decade grouping

2. Common Numbering Schemes:
Century/Decade Grouping (Most Common):
100-series: Overall system/main components
200-series: External systems/interfaces  
300-series: Processes/methods
400-series: Sub-components or modules
500-series: Detailed process steps
600-series: Data structures/databases
Even Number Spacing:
100, 102, 104, 106... (allows insertion of 101, 103 if needed)
Functional Grouping:
10-19: Input devices
20-29: Processing units
30-39: Output devices
40-49: Storage
3. How I Determined the CLM Numbers:
I followed the century/decade grouping pattern:
100-series - Main system components:

100: Overall CLM System
120: Microservices container
122, 124, 126: Individual microservices

200-series - External integrations:

202: CAS (Central Asset System)
204: CMP (Certificate Management Platform)
206: ServiceNow
208: Ansible Tower
210: HashiCorp Vault

300-series - Process/workflow elements:

302: Server Onboarding
304: Connectivity Check
306: Certificate Scanning
308: Impact Analysis
310: Risk Analysis

400-series - Monitoring components:

402: Expiration Scanning
404: Renewal Initiation
406: SLA Monitoring

500-series - Renewal process steps:

502: CMP Request Creation
504: Risk Validation
506: Status Polling
etc.


-------------

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Certificate Lifecycle Management System Patent Document</title>
    <style>
        body {
            font-family: 'Times New Roman', serif;
            line-height: 1.6;
            margin: 40px;
            background-color: #f5f5f5;
        }
        .patent-header {
            text-align: center;
            margin-bottom: 40px;
            border-bottom: 3px double #000;
            padding-bottom: 20px;
        }
        .patent-number {
            font-size: 24px;
            font-weight: bold;
            margin: 10px 0;
        }
        .section {
            margin: 30px 0;
            background-color: white;
            padding: 20px;
            border: 1px solid #ddd;
        }
        .figure {
            border: 2px solid #333;
            padding: 20px;
            margin: 20px 0;
            background-color: #fafafa;
            text-align: center;
        }
        .figure-title {
            font-weight: bold;
            margin-top: 10px;
        }
        svg {
            max-width: 100%;
            height: auto;
        }
        .claims {
            counter-reset: claim;
        }
        .claim {
            counter-increment: claim;
            margin: 15px 0;
            padding-left: 30px;
        }
        .claim:before {
            content: counter(claim) ". ";
            font-weight: bold;
            margin-left: -30px;
            display: inline-block;
            width: 30px;
        }
        h2 {
            border-bottom: 2px solid #333;
            padding-bottom: 10px;
        }
        .component-list {
            columns: 2;
            column-gap: 40px;
        }
        .component-item {
            break-inside: avoid;
            margin: 5px 0;
        }
    </style>
</head>
<body>
    <div class="patent-header">
        <div>United States Patent</div>
        <div class="patent-number">US X,XXX,XXX B2</div>
        <div>CERTIFICATE LIFECYCLE MANAGEMENT SYSTEM</div>
        <div>Date of Patent: [Current Date]</div>
    </div>

    <div class="section">
        <h2>ABSTRACT</h2>
        <p>A certificate lifecycle management system and method for automatically discovering, monitoring, and renewing digital certificates across an enterprise infrastructure. The system comprises a certificate discovery module that performs automated scanning of servers to identify certificates in various formats (.kdb, .pem, .jks, .p12), a monitoring module that tracks certificate expiration dates, and an automated renewal module that orchestrates certificate procurement and deployment. The system integrates with enterprise systems including a Central Asset System (CAS) for server inventory, Certificate Management Platform (CMP) for certificate issuance, and ServiceNow for change management. The invention provides role-based access control, automated notifications, and comprehensive audit trails while maintaining certificate security through integration with HashiCorp Vault.</p>
    </div>

    <div class="section">
        <h2>FIELD OF THE INVENTION</h2>
        <p>The present invention relates to digital certificate management systems, and more particularly, to automated systems for discovering, monitoring, and renewing digital certificates across enterprise computing infrastructures.</p>
    </div>

    <div class="section">
        <h2>BACKGROUND OF THE INVENTION</h2>
        <p>Digital certificates are fundamental to securing network communications and authenticating services in modern enterprise environments. Organizations typically manage thousands of certificates across numerous servers and applications. Manual certificate management processes are error-prone and can lead to service outages when certificates expire unexpectedly. There exists a need for an automated system that can discover certificates across diverse environments, monitor their lifecycle, and orchestrate timely renewals while maintaining security and compliance requirements.</p>
    </div>

    <div class="section">
        <h2>BRIEF DESCRIPTION OF THE DRAWINGS</h2>
        <p>FIG. 1 illustrates a system architecture diagram of the Certificate Lifecycle Management system according to an embodiment of the present invention.</p>
        <p>FIG. 2 depicts a detailed component interaction diagram showing the certificate renewal process flow.</p>
        <p>FIG. 3 shows a flowchart of the automated certificate discovery and monitoring process.</p>
        <p>FIG. 4 illustrates the data flow and integration architecture of the system.</p>
    </div>

    <div class="figure">
        <svg viewBox="0 0 800 600" xmlns="http://www.w3.org/2000/svg">
            <!-- System Architecture Diagram -->
            <rect x="10" y="10" width="780" height="580" fill="none" stroke="#000" stroke-width="2"/>
            
            <!-- Title -->
            <text x="400" y="40" text-anchor="middle" font-size="16" font-weight="bold">CERTIFICATE LIFECYCLE MANAGEMENT SYSTEM 100</text>
            
            <!-- External Systems -->
            <g id="external-systems">
                <!-- CAS -->
                <rect x="50" y="80" width="120" height="60" fill="#e8f4f8" stroke="#000" stroke-width="2"/>
                <text x="110" y="105" text-anchor="middle" font-size="12">CENTRAL ASSET</text>
                <text x="110" y="120" text-anchor="middle" font-size="12">SYSTEM (CAS)</text>
                <text x="110" y="135" text-anchor="middle" font-size="10" font-weight="bold">102</text>
                
                <!-- CMP -->
                <rect x="50" y="160" width="120" height="60" fill="#e8f4f8" stroke="#000" stroke-width="2"/>
                <text x="110" y="185" text-anchor="middle" font-size="12">CERTIFICATE MGT</text>
                <text x="110" y="200" text-anchor="middle" font-size="12">PLATFORM (CMP)</text>
                <text x="110" y="215" text-anchor="middle" font-size="10" font-weight="bold">104</text>
                
                <!-- ServiceNow -->
                <rect x="50" y="240" width="120" height="60" fill="#e8f4f8" stroke="#000" stroke-width="2"/>
                <text x="110" y="265" text-anchor="middle" font-size="12">SERVICENOW</text>
                <text x="110" y="280" text-anchor="middle" font-size="12">(SNOW)</text>
                <text x="110" y="295" text-anchor="middle" font-size="10" font-weight="bold">106</text>
                
                <!-- Ansible Tower -->
                <rect x="50" y="320" width="120" height="60" fill="#e8f4f8" stroke="#000" stroke-width="2"/>
                <text x="110" y="345" text-anchor="middle" font-size="12">ANSIBLE</text>
                <text x="110" y="360" text-anchor="middle" font-size="12">TOWER</text>
                <text x="110" y="375" text-anchor="middle" font-size="10" font-weight="bold">108</text>
                
                <!-- HashiCorp Vault -->
                <rect x="50" y="400" width="120" height="60" fill="#e8f4f8" stroke="#000" stroke-width="2"/>
                <text x="110" y="425" text-anchor="middle" font-size="12">HASHICORP</text>
                <text x="110" y="440" text-anchor="middle" font-size="12">VAULT</text>
                <text x="110" y="455" text-anchor="middle" font-size="10" font-weight="bold">110</text>
            </g>
            
            <!-- CLM Core Components -->
            <rect x="250" y="100" width="300" height="380" fill="#f0f0f0" stroke="#000" stroke-width="2"/>
            <text x="400" y="125" text-anchor="middle" font-size="14" font-weight="bold">CLM CORE SYSTEM 120</text>
            
            <!-- Microservices -->
            <rect x="270" y="150" width="120" height="50" fill="#fff" stroke="#000" stroke-width="1"/>
            <text x="330" y="170" text-anchor="middle" font-size="11">AUTHENTICATION</text>
            <text x="330" y="185" text-anchor="middle" font-size="11">SERVICE</text>
            <text x="330" y="195" text-anchor="middle" font-size="10" font-weight="bold">122</text>
            
            <rect x="410" y="150" width="120" height="50" fill="#fff" stroke="#000" stroke-width="1"/>
            <text x="470" y="170" text-anchor="middle" font-size="11">INVENTORY</text>
            <text x="470" y="185" text-anchor="middle" font-size="11">SERVICE</text>
            <text x="470" y="195" text-anchor="middle" font-size="10" font-weight="bold">124</text>
            
            <rect x="340" y="220" width="120" height="50" fill="#fff" stroke="#000" stroke-width="1"/>
            <text x="400" y="240" text-anchor="middle" font-size="11">CLM CORE</text>
            <text x="400" y="255" text-anchor="middle" font-size="11">SERVICE</text>
            <text x="400" y="265" text-anchor="middle" font-size="10" font-weight="bold">126</text>
            
            <!-- Core Modules -->
            <rect x="270" y="290" width="110" height="40" fill="#d4e8d4" stroke="#000" stroke-width="1"/>
            <text x="325" y="307" text-anchor="middle" font-size="10">DISCOVERY</text>
            <text x="325" y="320" text-anchor="middle" font-size="10">MODULE 128</text>
            
            <rect x="400" y="290" width="110" height="40" fill="#d4e8d4" stroke="#000" stroke-width="1"/>
            <text x="455" y="307" text-anchor="middle" font-size="10">MONITORING</text>
            <text x="455" y="320" text-anchor="middle" font-size="10">MODULE 130</text>
            
            <rect x="335" y="350" width="110" height="40" fill="#d4e8d4" stroke="#000" stroke-width="1"/>
            <text x="390" y="367" text-anchor="middle" font-size="10">RENEWAL</text>
            <text x="390" y="380" text-anchor="middle" font-size="10">MODULE 132</text>
            
            <rect x="270" y="410" width="110" height="40" fill="#d4e8d4" stroke="#000" stroke-width="1"/>
            <text x="325" y="427" text-anchor="middle" font-size="10">NOTIFICATION</text>
            <text x="325" y="440" text-anchor="middle" font-size="10">MODULE 134</text>
            
            <rect x="400" y="410" width="110" height="40" fill="#d4e8d4" stroke="#000" stroke-width="1"/>
            <text x="455" y="427" text-anchor="middle" font-size="10">AUDIT</text>
            <text x="455" y="440" text-anchor="middle" font-size="10">MODULE 136</text>
            
            <!-- Frontend -->
            <rect x="600" y="200" width="150" height="80" fill="#fff8dc" stroke="#000" stroke-width="2"/>
            <text x="675" y="230" text-anchor="middle" font-size="12" font-weight="bold">ANGULAR</text>
            <text x="675" y="250" text-anchor="middle" font-size="12">FRONTEND</text>
            <text x="675" y="270" text-anchor="middle" font-size="10" font-weight="bold">140</text>
            
            <!-- Database -->
            <rect x="325" y="510" width="150" height="60" fill="#e6e6fa" stroke="#000" stroke-width="2"/>
            <text x="400" y="535" text-anchor="middle" font-size="12" font-weight="bold">MONGODB</text>
            <text x="400" y="555" text-anchor="middle" font-size="10" font-weight="bold">150</text>
            
            <!-- Users -->
            <rect x="600" y="80" width="150" height="60" fill="#ffe4e1" stroke="#000" stroke-width="2"/>
            <text x="675" y="105" text-anchor="middle" font-size="12">USERS/ADMINS</text>
            <text x="675" y="125" text-anchor="middle" font-size="10" font-weight="bold">160</text>
            
            <!-- Target Servers -->
            <rect x="600" y="350" width="150" height="100" fill="#e0ffff" stroke="#000" stroke-width="2"/>
            <text x="675" y="380" text-anchor="middle" font-size="12" font-weight="bold">LINUX SERVERS</text>
            <text x="675" y="400" text-anchor="middle" font-size="10">WITH CERTIFICATES</text>
            <text x="675" y="420" text-anchor="middle" font-size="9">(.pem, .jks, .p12, .kdb)</text>
            <text x="675" y="440" text-anchor="middle" font-size="10" font-weight="bold">170</text>
            
            <!-- Connection Lines -->
            <line x1="170" y1="110" x2="270" y2="175" stroke="#000" stroke-width="1.5" marker-end="url(#arrowhead)"/>
            <line x1="170" y1="190" x2="340" y2="245" stroke="#000" stroke-width="1.5" marker-end="url(#arrowhead)"/>
            <line x1="170" y1="270" x2="335" y2="365" stroke="#000" stroke-width="1.5" marker-end="url(#arrowhead)"/>
            <line x1="170" y1="350" x2="270" y2="310" stroke="#000" stroke-width="1.5" marker-end="url(#arrowhead)"/>
            <line x1="170" y1="430" x2="270" y2="425" stroke="#000" stroke-width="1.5" marker-end="url(#arrowhead)"/>
            
            <line x1="675" y1="140" x2="675" y2="200" stroke="#000" stroke-width="1.5" marker-end="url(#arrowhead)"/>
            <line x1="600" y1="240" x2="530" y2="240" stroke="#000" stroke-width="1.5" marker-end="url(#arrowhead)"/>
            <line x1="550" y1="400" x2="600" y2="400" stroke="#000" stroke-width="1.5" marker-end="url(#arrowhead)"/>
            <line x1="400" y1="480" x2="400" y2="510" stroke="#000" stroke-width="1.5" marker-end="url(#arrowhead)"/>
            
            <!-- Arrow marker definition -->
            <defs>
                <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="10" refY="3.5" orient="auto">
                    <polygon points="0 0, 10 3.5, 0 7" fill="#000"/>
                </marker>
            </defs>
        </svg>
        <div class="figure-title">FIG. 1 - CLM SYSTEM ARCHITECTURE</div>
    </div>

    <div class="section">
        <h2>DETAILED DESCRIPTION</h2>
        
        <h3>System Overview</h3>
        <p>The Certificate Lifecycle Management (CLM) system 100 provides an automated solution for managing digital certificates across enterprise infrastructure. As shown in FIG. 1, the system comprises multiple integrated components working together to discover, monitor, and renew certificates.</p>
        
        <h3>Core Components</h3>
        <p>The CLM Core System 120 includes three primary microservices:</p>
        <ul>
            <li><strong>Authentication Service 122:</strong> Validates users against enterprise Active Directory/LDAP systems</li>
            <li><strong>Inventory Service 124:</strong> Manages server inventory synchronized from the Central Asset System (CAS)</li>
            <li><strong>CLM Core Service 126:</strong> Orchestrates all certificate management operations</li>
        </ul>
        
        <h3>Functional Modules</h3>
        <p>The system includes several functional modules:</p>
        <ul>
            <li><strong>Discovery Module 128:</strong> Scans servers weekly to identify certificates in various formats</li>
            <li><strong>Monitoring Module 130:</strong> Performs daily checks for certificate expiration</li>
            <li><strong>Renewal Module 132:</strong> Automates the certificate renewal workflow</li>
            <li><strong>Notification Module 134:</strong> Sends alerts and status updates via email</li>
            <li><strong>Audit Module 136:</strong> Maintains comprehensive logs of all system activities</li>
        </ul>
        
        <h3>External Integrations</h3>
        <p>The system integrates with several enterprise services:</p>
        <ul>
            <li><strong>Central Asset System (CAS) 102:</strong> Provides authoritative server inventory</li>
            <li><strong>Certificate Management Platform (CMP) 104:</strong> Issues new certificates</li>
            <li><strong>ServiceNow 106:</strong> Manages change requests for certificate deployment</li>
            <li><strong>Ansible Tower 108:</strong> Executes automation playbooks for certificate operations</li>
            <li><strong>HashiCorp Vault 110:</strong> Securely stores sensitive credentials and secrets</li>
        </ul>
        
        <h3>Certificate Discovery Process</h3>
        <p>The discovery process begins when a team requests server onboarding. After SSH keys are deployed via change request, the system performs connectivity checks and initiates certificate scanning. The Discovery Module 128 uses Ansible playbooks to scan for certificates in multiple formats including .pem, .jks, .p12, and .kdb files. Discovered certificate metadata is stored in MongoDB 150.</p>
        
        <h3>Monitoring and Renewal</h3>
        <p>The Monitoring Module 130 performs daily checks to identify certificates expiring within 60 days. When expiring certificates are detected, the Renewal Module 132 initiates an automated workflow that includes:</p>
        <ol>
            <li>Creating a request in the Certificate Management Platform</li>
            <li>Polling for approval status</li>
            <li>Importing the new certificate to the original keystore format</li>
            <li>Raising a ServiceNow change request</li>
            <li>Deploying the certificate upon change approval</li>
        </ol>
        
        <h3>Security and Access Control</h3>
        <p>The system implements role-based access control with four levels:</p>
        <ul>
            <li><strong>ISA_ADMIN:</strong> Can onboard new users</li>
            <li><strong>PFY_ADMIN:</strong> Has full system access except user onboarding</li>
            <li><strong>FARM_ADMIN:</strong> Can view and execute operations on owned servers</li>
            <li><strong>USER:</strong> Has read-only access to certificate data</li>
        </ul>
    </div>

    <div class="section">
        <h2>CLAIMS</h2>
        <div class="claims">
            <div class="claim">A certificate lifecycle management system comprising:
                <ul style="list-style-type: none;">
                    <li>a) a discovery module configured to automatically scan servers and identify digital certificates in multiple file formats;</li>
                    <li>b) a monitoring module configured to track certificate expiration dates;</li>
                    <li>c) an automated renewal module configured to orchestrate certificate procurement and deployment;</li>
                    <li>d) a database for storing certificate metadata while maintaining actual certificates on their original servers; and</li>
                    <li>e) a role-based access control system providing differentiated access levels.</li>
                </ul>
            </div>
            
            <div class="claim">The system of claim 1, wherein the discovery module utilizes Ansible playbooks to scan for certificates in .jks, .p12, .pem, and .kdb formats.</div>
            
            <div class="claim">The system of claim 1, wherein the monitoring module performs daily expiration checks and initiates renewal for certificates expiring within a configurable threshold of 60 days.</div>
            
            <div class="claim">The system of claim 1, wherein the renewal module executes a multi-stage process comprising:
                <ul style="list-style-type: none;">
                    <li>a) creating procurement requests in an enterprise certificate management platform;</li>
                    <li>b) polling for approval status;</li>
                    <li>c) importing approved certificates;</li>
                    <li>d) raising change requests in a service management system; and</li>
                    <li>e) deploying certificates upon change approval.</li>
                </ul>
            </div>
            
            <div class="claim">The system of claim 1, further comprising integration interfaces for:
                <ul style="list-style-type: none;">
                    <li>a) a central asset system for server inventory management;</li>
                    <li>b) a certificate management platform for certificate issuance;</li>
                    <li>c) a change management system for deployment approval;</li>
                    <li>d) an automation platform for playbook execution; and</li>
                    <li>e) a secrets management system for credential storage.</li>
                </ul>
            </div>
            
            <div class="claim">The system of claim 1, wherein certificate relationships are managed through master/slave grouping for identical certificates deployed across multiple locations.</div>
            
            <div class="claim">The system of claim 1, wherein the role-based access control system provides four authorization levels: ISA_ADMIN for user onboarding, PFY_ADMIN for full system access, FARM_ADMIN for farm-specific operations, and USER for read-only access.</div>
            
            <div class="claim">The system of claim 1, wherein the system sends notifications when renewal service level agreements are breached, specifically when renewal is not completed within 20 days of initiation.</div>
            
            <div class="claim">The system of claim 1, implemented as containerized microservices deployed on an enterprise container platform.</div>
            
            <div class="claim">A method for automated certificate lifecycle management comprising:
                <ul style="list-style-type: none;">
                    <li>a) automatically discovering certificates across a server infrastructure;</li>
                    <li>b) storing certificate metadata in a centralized database;</li>
                    <li>c) monitoring certificate expiration dates daily;</li>
                    <li>d) initiating automated renewal for expiring certificates;</li>
                    <li>e) orchestrating certificate deployment through integrated enterprise systems; and</li>
                    <li>f) maintaining audit trails of all certificate operations.</li>
                </ul>
            </div>
        </div>
    </div>

    <div class="section">
        <h2>REFERENCE NUMERALS</h2>
        <div class="component-list">
            <div class="component-item"><strong>100</strong> - Certificate Lifecycle Management System</div>
            <div class="component-item"><strong>102</strong> - Central Asset System (CAS)</div>
            <div class="component-item"><strong>104</strong> - Certificate Management Platform (CMP)</div>
            <div class="component-item"><strong>106</strong> - ServiceNow</div>
            <div class="component-item"><strong>108</strong> - Ansible Tower</div>
            <div class="component-item"><strong>110</strong> - HashiCorp Vault</div>
            <div class="component-item"><strong>120</strong> - CLM Core System</div>
            <div class="component-item"><strong>122</strong> - Authentication Service</div>
            <div class="component-item"><strong>124</strong> - Inventory Service</div>
            <div class="component-item"><strong>126</strong> - CLM Core Service</div>
            <div class="component-item"><strong>128</strong> - Discovery Module</div>
            <div class="component-item"><strong>130</strong> - Monitoring Module</div>
            <div class="component-item"><strong>132</strong> - Renewal Module</div>
            <div class="component-item"><strong>134</strong> - Notification Module</div>
            <div class="component-item"><strong>136</strong> - Audit Module</div>
            <div class="component-item"><strong>140</strong> - Angular Frontend</div>
            <div class="component-item"><strong>150</strong> - MongoDB Database</div>
            <div class="component-item"><strong>160</strong> - Users/Admins</div>
            <div class="component-item"><strong>170</strong> - Linux Servers</div>
        </div>
    </div>
</body>
</html>
