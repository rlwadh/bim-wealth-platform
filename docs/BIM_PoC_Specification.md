# BIM Wealth Platform — Proof of Concept Backend Specification

**Version 1.0 · February 2026**  
**Prepared by ADHOCON GmbH**

---

## 1. PoC Scope & Objectives

This specification defines a working Proof of Concept that demonstrates the core platform capabilities with real backend logic, suitable for investor demonstrations and initial client testing.

### In Scope (PoC)
- User authentication with biometric enrollment (voice + face)
- Client onboarding flow with KYC/AML checks
- Portfolio dashboard with real-time valuations
- AI conversational assistant (Claude API integration)
- Product catalogue covering all 18+ product types
- Advisor booking and live chat handoff
- Compliance audit trail
- FCA & BaFin regulatory checks (automated)

### Out of Scope (PoC)
- Live trading / fund order execution
- Production payment processing
- Full white-label partner API integrations
- Mobile native apps (PWA only for PoC)

---

## 2. Backend Architecture

### Technology Stack

| Component | Technology | Justification |
|-----------|-----------|---------------|
| Runtime | Node.js 20 + NestJS | TypeScript-first, modular microservice framework |
| Database | PostgreSQL 16 | ACID compliance, JSON support, regulatory audit |
| Cache | Redis 7 | Session management, rate limiting, real-time data |
| Search | Elasticsearch 8 | Audit trail search, full-text compliance queries |
| AI | Anthropic Claude API | Conversational AI, document analysis, recommendations |
| Voice | Azure Cognitive Services | Speech-to-text, voice biometric enrollment |
| Face | AWS Rekognition | Facial recognition, liveness detection |
| Auth | Auth0 / Keycloak | OAuth 2.0, MFA, biometric binding |
| Queue | Bull (Redis-backed) | Background jobs: KYC checks, report generation |
| Storage | S3-compatible | Documents, biometric templates, reports |
| API Docs | Swagger/OpenAPI 3.1 | Auto-generated from NestJS decorators |

### Service Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    API GATEWAY (Kong)                     │
│         Rate Limiting · Auth · Request Routing            │
├─────────┬──────────┬──────────┬──────────┬──────────────┤
│ Account │ Product  │ Advisory │Compliance│  Reporting   │
│ Service │ Service  │ Service  │ Service  │  Service     │
├─────────┴──────────┴──────────┴──────────┴──────────────┤
│                  Message Broker (Redis/Bull)              │
├─────────────────────────────────────────────────────────┤
│  PostgreSQL  │   Redis    │  Elasticsearch  │    S3     │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Data Models

### Core Entities

**Client**
```typescript
interface Client {
  id: UUID;
  externalRef: string;
  title: string;
  firstName: string;
  lastName: string;
  dateOfBirth: Date;
  email: string;
  phone: string;
  address: Address;
  nationalInsuranceNumber?: string; // UK
  taxId?: string; // DE
  kycStatus: 'pending' | 'verified' | 'expired' | 'failed';
  kycVerifiedAt?: Date;
  riskProfile: RiskProfile;
  biometrics: {
    voiceEnrolled: boolean;
    voiceTemplateId?: string;
    faceEnrolled: boolean;
    faceTemplateId?: string;
  };
  advisorId?: UUID;
  segment: 'standard' | 'premium' | 'hnw' | 'uhnw';
  jurisdiction: 'UK' | 'DE' | 'EU';
  createdAt: Date;
  updatedAt: Date;
}
```

**Product Holding**
```typescript
interface ProductHolding {
  id: UUID;
  clientId: UUID;
  productType: ProductType; // enum of all 18+ product types
  productName: string;
  status: 'draft' | 'pending' | 'active' | 'suspended' | 'closed';
  currentValue?: number;
  currency: 'GBP' | 'EUR' | 'USD';
  contributions: number;
  withdrawals: number;
  performancePercent?: number;
  riskRating?: number;
  inceptionDate: Date;
  maturityDate?: Date;
  taxWrapper: 'ISA' | 'SIPP' | 'GIA' | 'Bond' | 'Pension' | 'Trust';
  regulatoryFlags: string[];
  metadata: Record<string, any>; // product-specific fields
}
```

**Advisory Session**
```typescript
interface AdvisorySession {
  id: UUID;
  clientId: UUID;
  advisorId?: UUID;
  mode: 'ai' | 'human' | 'hybrid';
  channel: 'chat' | 'voice' | 'video';
  messages: Message[];
  context: {
    portfolioSnapshot: any;
    recentActions: any[];
    complianceFlags: string[];
  };
  suitabilityChecked: boolean;
  startedAt: Date;
  endedAt?: Date;
}
```

---

## 4. API Endpoints

### Authentication & Onboarding
```
POST   /api/v1/auth/register          # New client registration
POST   /api/v1/auth/login             # Email/password login
POST   /api/v1/auth/biometric/voice   # Voice authentication
POST   /api/v1/auth/biometric/face    # Face authentication
POST   /api/v1/auth/mfa/verify        # MFA verification
POST   /api/v1/onboarding/kyc         # Submit KYC documents
GET    /api/v1/onboarding/kyc/status  # Check KYC status
POST   /api/v1/onboarding/risk-profile # Submit risk questionnaire
```

### Portfolio & Products
```
GET    /api/v1/portfolio              # Full portfolio overview
GET    /api/v1/portfolio/summary      # KPI summary (total value, yield, etc.)
GET    /api/v1/portfolio/allocation   # Asset allocation breakdown
GET    /api/v1/products               # List all available products
GET    /api/v1/products/:id           # Product details
POST   /api/v1/products/:id/apply     # Apply for a product
GET    /api/v1/holdings               # Client's product holdings
GET    /api/v1/holdings/:id           # Specific holding details
GET    /api/v1/holdings/:id/performance # Performance history
POST   /api/v1/holdings/:id/contribute # Make a contribution
POST   /api/v1/holdings/:id/withdraw  # Request withdrawal
```

### Advisory
```
POST   /api/v1/advisory/session       # Start advisory session
POST   /api/v1/advisory/message       # Send message (AI or human)
POST   /api/v1/advisory/handoff       # Transfer AI → human advisor
GET    /api/v1/advisory/history       # Session history
POST   /api/v1/advisory/book          # Book advisor appointment
GET    /api/v1/advisors               # List available advisors
GET    /api/v1/advisors/:id/slots     # Advisor availability
```

### Compliance
```
GET    /api/v1/compliance/status      # Overall compliance status
GET    /api/v1/compliance/audit       # Audit trail (paginated)
POST   /api/v1/compliance/suitability # Run suitability check
GET    /api/v1/compliance/regulatory  # FCA/BaFin status
POST   /api/v1/compliance/report      # Generate compliance report
```

### Corporate
```
POST   /api/v1/corporate/employer     # Register employer
GET    /api/v1/corporate/schemes      # List pension schemes
POST   /api/v1/corporate/enrol        # Enrol employee
GET    /api/v1/corporate/wellbeing    # Financial wellbeing scores
```

---

## 5. AI Integration Architecture

### Claude API Integration

```typescript
// Advisory AI Service
class AdvisoryAIService {
  async processMessage(
    clientId: string,
    message: string,
    sessionContext: SessionContext
  ): Promise<AIResponse> {
    
    // 1. Enrich context with client data
    const clientProfile = await this.getClientProfile(clientId);
    const portfolio = await this.getPortfolioSummary(clientId);
    const complianceFlags = await this.getComplianceFlags(clientId);
    
    // 2. Build system prompt with regulatory guardrails
    const systemPrompt = this.buildSystemPrompt({
      clientProfile,
      portfolio,
      complianceFlags,
      jurisdiction: clientProfile.jurisdiction,
      advisorCapabilities: this.getAdvisorCapabilities(),
    });
    
    // 3. Call Claude API
    const response = await this.anthropic.messages.create({
      model: "claude-sonnet-4-5-20250929",
      max_tokens: 1000,
      system: systemPrompt,
      messages: sessionContext.messages,
    });
    
    // 4. Post-process for compliance
    const filtered = await this.complianceFilter(response);
    
    // 5. Check if handoff to human is needed
    if (this.needsHumanHandoff(filtered)) {
      await this.initiateHandoff(clientId, sessionContext);
    }
    
    return filtered;
  }
}
```

### AI Guardrails
- Never provide specific investment advice (regulatory requirement)
- Always include appropriate risk warnings
- Detect vulnerability indicators and flag for advisor review
- Log all AI interactions for compliance audit
- Jurisdiction-aware responses (FCA vs BaFin language)

---

## 6. Biometric Integration

### Voice Recognition Flow
```
1. Enrollment: Client reads 3 prompted phrases → voiceprint template created
2. Authentication: Client speaks passphrase → compared against template
3. Continuous: During calls, ongoing voice verification in background
4. Liveness: Random challenge-response to prevent replay attacks
```

### Facial Recognition Flow
```
1. Enrollment: Client captures face via webcam/mobile → template created
2. Document Match: ID document photo compared to live face (KYC)
3. Authentication: Face scan at login for biometric MFA
4. Liveness: Blink/head-turn detection to prevent photo attacks
```

---

## 7. Compliance Engine

### Automated Checks
| Check | Frequency | Regulation |
|-------|-----------|------------|
| KYC/AML screening | On onboarding + annual | FCA SYSC, BaFin GwG |
| Sanctions screening | Daily batch + real-time | UK/EU sanctions lists |
| Suitability assessment | Per transaction | FCA COBS, BaFin WpHG |
| Client categorisation | On onboarding + change | MiFID II |
| Vulnerable customer flag | Per interaction (AI) | FCA Consumer Duty |
| CASS reconciliation | Daily | FCA CASS |
| Transaction monitoring | Real-time | AML regulations |
| Data retention check | Monthly | UK/EU GDPR |

---

## 8. Deployment Architecture

### Infrastructure (AWS)
```
Region: eu-west-2 (London) + eu-central-1 (Frankfurt)

├── VPC (Private Subnets)
│   ├── EKS Cluster (Kubernetes)
│   │   ├── Account Service (2 replicas)
│   │   ├── Product Service (2 replicas)
│   │   ├── Advisory Service (3 replicas)
│   │   ├── Compliance Service (2 replicas)
│   │   └── Reporting Service (1 replica)
│   ├── RDS PostgreSQL (Multi-AZ)
│   ├── ElastiCache Redis (Cluster)
│   └── Elasticsearch (3-node cluster)
├── S3 (Document storage, encrypted)
├── CloudFront (CDN for frontend)
├── WAF (Web Application Firewall)
├── CloudHSM (Key management)
└── CloudWatch + X-Ray (Monitoring)
```

### Data Residency
- UK client data: eu-west-2 (London)
- DE/EU client data: eu-central-1 (Frankfurt)
- Cross-region replication only for disaster recovery (encrypted)

---

## 9. Security Measures

- **Encryption**: AES-256 at rest, TLS 1.3 in transit, field-level for PII
- **Authentication**: OAuth 2.0 + PKCE, biometric MFA, device trust
- **Authorisation**: RBAC with attribute-based policies
- **Network**: Zero-trust, mTLS between services, private subnets
- **Monitoring**: SIEM integration, anomaly detection, 24/7 alerting
- **Compliance**: SOC 2 Type II target, ISO 27001 alignment
- **Pen Testing**: Annual third-party + continuous automated scanning
- **Incident Response**: Documented playbooks, <1hr response SLA

---

## 10. Development Timeline (PoC)

| Week | Deliverable |
|------|-------------|
| 1–2 | Project setup, database schema, auth service, CI/CD pipeline |
| 3–4 | Client onboarding flow, KYC integration, risk profiling |
| 5–6 | Product catalogue, portfolio service, dashboard API |
| 7–8 | AI advisory integration (Claude API), chat interface |
| 9–10 | Biometric enrollment (voice + face), MFA integration |
| 11–12 | Compliance engine, audit trail, regulatory reporting |
| 13–14 | Corporate module, white-label partner stubs |
| 15–16 | Integration testing, security audit, PoC deployment |

**Total PoC Timeline: 16 weeks**

---

*This document is confidential and intended for BIM stakeholders and investors only.*  
*© 2026 ADHOCON GmbH. All rights reserved.*
