graph TB
    subgraph "CURRENT BAU PROCESS (Manual)"
        A1[Manual: Maintain Excel/Spreadsheet<br/>Certificate Inventory] --> A2[CAS Email Notification<br/>90 days before expiry]
        A2 --> A3{Team Notices<br/>Email?}
        A3 -->|No| A4[Certificate Expires<br/>⚠️ SERVICE OUTAGE]
        A3 -->|Yes| A5[Manual: Investigate Impact<br/>Check Configs & Docs]
        A5 --> A6[Manual: Contact Teams<br/>to Find All Locations]
        A6 --> A7{Found All<br/>Locations?}
        A7 -->|No| A4
        A7 -->|Yes| A8[Manual: Raise<br/>Marketplace Order]
        A8 --> A9[Manual: Submit to CA<br/>for Certificate Renewal]
        A9 --> A10[Manual: Receive<br/>New Certificate]
        A10 --> A11[Manual: Create Change Request<br/>in ServiceNow]
        A11 --> A12[Manual: Wait for<br/>CR Approval]
        A12 --> A13[Manual: SSH to Each Server]
        A13 --> A14[Manual: Copy Certificate Files<br/>to Target Paths]
        A14 --> A15[Manual: Update Trust Stores]
        A15 --> A16[Manual: Restart Services]
        A16 --> A17[Manual: Run CLI Commands<br/>for Validation]
        A17 --> A18[Manual: Test Connectivity<br/>& Check Logs]
        A18 --> A19[Manual: Sign-off<br/>& Documentation]
        
        style A1 fill:#ffcccc,stroke:#cc0000
        style A2 fill:#ffcccc,stroke:#cc0000
        style A4 fill:#ff9999,stroke:#ff0000,stroke-width:3px
        style A5 fill:#ffcccc,stroke:#cc0000
        style A6 fill:#ffcccc,stroke:#cc0000
        style A8 fill:#ffcccc,stroke:#cc0000
        style A9 fill:#ffcccc,stroke:#cc0000
        style A10 fill:#ffcccc,stroke:#cc0000
        style A11 fill:#ffcccc,stroke:#cc0000
        style A12 fill:#ffcccc,stroke:#cc0000
        style A13 fill:#ffcccc,stroke:#cc0000
        style A14 fill:#ffcccc,stroke:#cc0000
        style A15 fill:#ffcccc,stroke:#cc0000
        style A16 fill:#ffcccc,stroke:#cc0000
        style A17 fill:#ffcccc,stroke:#cc0000
        style A18 fill:#ffcccc,stroke:#cc0000
        style A19 fill:#ffcccc,stroke:#cc0000
    end
    
    subgraph "CLM-ENABLED PROCESS (Automated)"
        B1[One-time: Raise Marketplace Order<br/>for CLM Access] --> B2[One-time: CSI Onboarding<br/>Provide Team DL + SNOW DL]
        B2 --> B3[AUTO: Sync Servers<br/>from Drift]
        B3 --> B4[AUTO: Validate Ansible<br/>Connectivity via SSH]
        B4 --> B5[AUTO: Comprehensive VM Scan<br/>Discover All Certificates]
        B5 --> B6[AUTO: Map Certificate<br/>to All Locations]
        B6 --> B7[AUTO: Group Related<br/>Certificates]
        B7 --> B8[AUTO: Apply Impact Assessment<br/>+ AppD Traffic Analysis]
        B8 --> B9[AUTO: Continuous Monitoring<br/>Weekly Reports]
        B9 --> B10{Certificate<br/>60 Days to<br/>Expiry?}
        B10 -->|No| B9
        B10 -->|Yes| B11[AUTO: Initiate Renewal Workflow<br/>Proactive Notification]
        B11 --> B12[AUTO: Create Marketplace Order<br/>for Renewal]
        B12 --> B13[AUTO: CA Issues<br/>New Certificate]
        B13 --> B14[MANUAL: Admin Provides<br/>Valid CR Number]
        B14 --> B15[AUTO: Validate CR<br/>in ServiceNow]
        B15 --> B16[AUTO: Check Maintenance<br/>Window Schedule]
        B16 --> B17[AUTO: Ansible SSH Connection<br/>to Target Servers]
        B17 --> B18[AUTO: Deploy Certificate<br/>to All Locations]
        B18 --> B19[AUTO: Update Trust Stores<br/>Maintain Integrity]
        B19 --> B20[AUTO: Restart Services<br/>as per Configuration]
        B20 --> B21[AUTO: Verify Certificate<br/>Copied Successfully]
        B21 --> B22[AUTO: Check Service<br/>Restart Status]
        B22 --> B23[AUTO: Update Dashboard<br/>& Notify Stakeholders]
        B23 --> B24[AUTO: Generate<br/>Change Records]
        B24 --> B9
        
        style B1 fill:#e6f3ff,stroke:#1976d2,stroke-width:2px
        style B2 fill:#e6f3ff,stroke:#1976d2,stroke-width:2px
        style B3 fill:#ccffcc,stroke:#4caf50
        style B4 fill:#ccffcc,stroke:#4caf50
        style B5 fill:#ccffcc,stroke:#4caf50
        style B6 fill:#ccffcc,stroke:#4caf50
        style B7 fill:#ccffcc,stroke:#4caf50
        style B8 fill:#ccffcc,stroke:#4caf50
        style B9 fill:#ccffcc,stroke:#4caf50
        style B11 fill:#ccffcc,stroke:#4caf50
        style B12 fill:#ccffcc,stroke:#4caf50
        style B13 fill:#ccffcc,stroke:#4caf50
        style B14 fill:#fff9cc,stroke:#ff9800,stroke-width:2px
        style B15 fill:#ccffcc,stroke:#4caf50
        style B16 fill:#ccffcc,stroke:#4caf50
        style B17 fill:#ccffcc,stroke:#4caf50
        style B18 fill:#ccffcc,stroke:#4caf50
        style B19 fill:#ccffcc,stroke:#4caf50
        style B20 fill:#ccffcc,stroke:#4caf50
        style B21 fill:#ccffcc,stroke:#4caf50
        style B22 fill:#ccffcc,stroke:#4caf50
        style B23 fill:#ccffcc,stroke:#4caf50
        style B24 fill:#ccffcc,stroke:#4caf50
    end
    
    subgraph "LEGEND"
        L1[Manual Step - BAU] 
        L2[Automated by CLM]
        L3[Manual Step in CLM]
        L4[One-time Setup CLM]
        L5[Critical Risk Point]
        
        style L1 fill:#ffcccc,stroke:#cc0000
        style L2 fill:#ccffcc,stroke:#4caf50
        style L3 fill:#fff9cc,stroke:#ff9800
        style L4 fill:#e6f3ff,stroke:#1976d2
        style L5 fill:#ff9999,stroke:#ff0000,stroke-width:3px
    end
    
    subgraph "KEY METRICS"
        M1["<b>BAU Process:</b><br/>• 19 Manual Steps<br/>• Hours per Certificate<br/>• High Risk of Missed Renewals<br/>• Tribal Knowledge Dependent"]
        M2["<b>CLM Process:</b><br/>• 1 Manual Step (CR Input)<br/>• Minutes per Certificate<br/>• Zero Missed Renewals<br/>• Fully Documented & Automated"]
        
        style M1 fill:#ffe6e6,stroke:#cc0000
        style M2 fill:#e6ffe6,stroke:#4caf50
    end
