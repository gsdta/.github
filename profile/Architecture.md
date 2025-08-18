# Greater San Diego Tamil Academy — System Sequence Diagrams

A consolidated set of Mermaid sequence diagrams for our low-cost, serverless AWS architecture (S3+CloudFront frontend, API Gateway + Lambda, Cognito, DynamoDB, SES, Stripe/PayPal).

## Table of Contents
- [1) Public page load (cached via CloudFront)](#1-public-page-load-cached-via-cloudfront)
- [2) Sign‑up + sign‑in (Cognito Hosted UI)](#2-sign-up--sign-in-cognito-hosted-ui)
- [3) Enrollment flow (student joins a class)](#3-enrollment-flow-student-joins-a-class)
- [4) Payment (Stripe or PayPal) and reconciliation webhook](#4-payment-stripe-or-paypal-and-reconciliation-webhook)
- [5) Teacher posts announcement (secured API)](#5-teacher-posts-announcement-secured-api)
- [6) Contact‑us form (serverless email)](#6-contact-us-form-serverless-email)
- [7) CI/CD (separate repos; minimal‑cost pipeline)](#7-cicd-separate-repos-minimal-cost-pipeline)
- [8) Backups & logs (observability and cost control)](#8-backups--logs-observability-and-cost-control)

> Tip: GitHub and many Markdown tools render Mermaid automatically. If needed, enable Mermaid support in your viewer (e.g., VS Code + Markdown Preview Mermaid support).

---

## 1) Public page load (cached via CloudFront)
```mermaid
sequenceDiagram
  autonumber
  actor User
  participant DNS as Route53 (DNS)
  participant CF as CloudFront (CDN)
  participant S3 as S3 (Static Site)
  participant API as API Gateway
  participant L1 as Lambda (Public APIs)
  participant DB as DynamoDB

  User->>DNS: Resolve academy.org
  DNS-->>User: A/AAAA (CloudFront)
  User->>CF: GET / (index.html)
  alt Cache HIT
    CF-->>User: 200 HTML/Assets
  else Cache MISS
    CF->>S3: GET /index.html
    S3-->>CF: 200
    CF-->>User: 200 (cached)
  end
  User->>CF: GET /api/public/classes
  CF->>API: /public/classes
  API->>L1: Invoke
  L1->>DB: Query Classes
  DB-->>L1: Items
  L1-->>API: 200 JSON
  API-->>CF: 200
  CF-->>User: 200 JSON (cache per TTL)
```

---

## 2) Sign‑up + sign‑in (Cognito Hosted UI)
```mermaid
sequenceDiagram
  autonumber
  actor User
  participant FE as React App (S3/CloudFront)
  participant Cognito as Amazon Cognito (Hosted UI)
  participant API as API Gateway
  participant L as Lambda (Auth Handler)
  participant DB as DynamoDB (Users)

  User->>FE: Click "Sign Up / Sign In"
  FE->>Cognito: Redirect (OAuth2/OIDC)
  Cognito-->>User: Auth form (email/OTP or password)
  User->>Cognito: Submit credentials
  Cognito-->>FE: Redirect with ID/Access token
  FE->>API: /me (Authorization: Bearer <token>)
  API->>L: Invoke (JWT verify)
  L->>DB: Upsert/Fetch profile
  DB-->>L: Profile
  L-->>API: 200 Profile
  API-->>FE: 200 Profile JSON
  FE-->>User: Logged‑in state
```

---

## 3) Enrollment flow (student joins a class)
```mermaid
sequenceDiagram
  autonumber
  actor Parent as Parent/User
  participant FE as React App
  participant API as API Gateway
  participant L as Lambda (Enrollment)
  participant DB as DynamoDB (Classes/Enrollments)
  participant SES as Amazon SES (Email)

  Parent->>FE: Select Class -> "Enroll"
  FE->>API: POST /enroll {classId, studentInfo} (JWT)
  API->>L: Invoke
  L->>DB: Validate capacity & create enrollment
  DB-->>L: OK (enrollmentId)
  L->>SES: Send confirmation email
  SES-->>Parent: Email: Enrollment confirmation
  L-->>API: 201 {enrollmentId, status:"PENDING_PAYMENT"}
  API-->>FE: 201
  FE-->>Parent: Show payment CTA
```

---

## 4) Payment (Stripe or PayPal) and reconciliation webhook
```mermaid
sequenceDiagram
  autonumber
  actor Parent
  participant FE as React App
  participant Pay as Stripe/PayPal
  participant API as API Gateway
  participant LPay as Lambda (Payment Webhook)
  participant DB as DynamoDB
  participant SES as Amazon SES

  Parent->>FE: Click "Pay"
  FE->>Pay: Create Checkout Session (amount, metadata: enrollmentId)
  Pay-->>Parent: Hosted checkout page
  Parent->>Pay: Complete payment
  Pay-->>Parent: Success page
  Pay-->>LPay: Webhook (payment_intent.succeeded)
  LPay->>DB: Update enrollment status = PAID
  LPay->>SES: Send receipt/confirmation email
  SES-->>Parent: Email: Payment receipt
```
> Note: If using PayPal, adapt event names/metadata accordingly. Keep webhook handler idempotent.

---

## 5) Teacher posts announcement (secured API)
```mermaid
sequenceDiagram
  autonumber
  actor Teacher
  participant FE as React App (Admin)
  participant API as API Gateway
  participant L as Lambda (Announcements)
  participant DB as DynamoDB (Announcements)
  participant SES as Amazon SES (Optional)
  participant CF as CloudFront (Cache)

  Teacher->>FE: Write announcement
  FE->>API: POST /admin/announcements (JWT, role=teacher)
  API->>L: Invoke (authorize)
  L->>DB: PutItem(announcement)
  DB-->>L: OK
  L-->>API: 201
  API-->>FE: 201
  FE-->>Teacher: Success
  note over CF: Public GET /announcements cached <short TTL>\nAdmin POST triggers cache invalidation if needed
```

---

## 6) Contact‑us form (serverless email)
```mermaid
sequenceDiagram
  autonumber
  actor Visitor
  participant FE as React App
  participant API as API Gateway
  participant L as Lambda (Contact)
  participant SES as Amazon SES
  participant Admin as Admin Inbox

  Visitor->>FE: Submit contact form
  FE->>API: POST /contact {name,email,message}
  API->>L: Invoke
  L->>SES: Send email to admin@academy.org
  SES-->>Admin: New contact email
  L-->>API: 202 Accepted
  API-->>FE: 202
  FE-->>Visitor: "Thanks, we'll be in touch!"
```

---

## 7) CI/CD (separate repos; minimal‑cost pipeline)
```mermaid
sequenceDiagram
  autonumber
  actor Dev as Developer
  participant GH1 as GitHub (infra repo: CDK)
  participant GH2 as GitHub (frontend repo)
  participant GH3 as GitHub (backend repo)
  participant GHA as GitHub Actions
  participant AWS as AWS (CDK/SAM/CLI)
  participant S3 as S3 (Static Site)
  participant CF as CloudFront
  participant L as Lambda
  participant API as API Gateway

  Dev->>GH1: Push infra changes
  GH1->>GHA: Workflow: cdk synth & deploy
  GHA->>AWS: cdk deploy (Route53, ACM, S3, CF, API, Cognito, DynamoDB, SES)
  AWS-->>GHA: Deployed

  Dev->>GH2: Push frontend
  GH2->>GHA: Workflow: build (npm ci && npm run build)
  GHA->>S3: Upload build/
  GHA->>CF: Create invalidation
  CF-->>GHA: OK

  Dev->>GH3: Push backend
  GH3->>GHA: Workflow: package & deploy (SAM/zip)
  GHA->>L: Update Lambda code
  L-->>GHA: OK
  GHA->>API: (optional) Stage deploy
  API-->>GHA: OK
```
> Cost note: GitHub Actions free tier is sufficient for low-volume deployments. Avoid always-on build servers.

---

## 8) Backups & logs (observability and cost control)
```mermaid
sequenceDiagram
  autonumber
  participant L as Lambda (All services)
  participant CW as CloudWatch (Logs/Metrics)
  participant EB as EventBridge (Schedules)
  participant S3 as S3 (Backups/Exports)
  participant DB as DynamoDB (Point‑in‑Time)
  participant Admin as Ops/Admin

  L->>CW: Put logs/metrics
  EB->>L: Nightly trigger (export reports)
  L->>S3: Write CSV/JSON (attendance/finance)
  DB-->>Admin: PITR enabled (restore if needed)
  Admin->>CW: View dashboards/alarms (errors, latency, 4xx/5xx)
```
> Enable DynamoDB Point-in-Time Recovery (PITR). Use metric filters/alarms sparingly to avoid costs.

---

### Notes
- **Separate repos:** infra (CDK), frontend (React), backend (Lambdas). Keep IAM least-privilege across pipelines.
- **Cost minimization:** Prefer CloudFront caching, DynamoDB on-demand, SES in-region, and idempotent Lambdas. Avoid NAT gateways.
- **Security:** Use Cognito groups/roles, API Gateway authorizers, and parameterize secrets in SSM Parameter Store.
