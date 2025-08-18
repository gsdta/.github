flowchart LR
  %% Layout + groups
  subgraph Client["CLIENTS"]
    U[fa:fa-globe Users]
    Admin[fa:fa-user-shield Admin/Teachers]
  end

  subgraph Frontend["FRONTEND (Static)"]
    CF[CloudFront CDN]
    S3[S3 Static Website\n(React + Bootstrap)]
  end

  subgraph Auth["IDENTITY"]
    COG[Cognito User Pool\n(Hosted UI + JWT)]
  end

  subgraph API["BACKEND (Serverless)"]
    APIGW[API Gateway]
    subgraph Lambdas["Lambda Functions"]
      LPublic[Public API]
      LEnroll[Enrollment]
      LPayHook[Payment Webhook]
      LContact[Contact Form]
      LAdmin[Admin/Announcements]
    end
    DB[(DynamoDB\nClasses • Enrollments • Users)]
  end

  subgraph Integrations["THIRD‑PARTY / EMAIL"]
    Stripe[Stripe / PayPal\nCheckout]
    SES[Amazon SES\nEmail]
  end

  subgraph Ops["OBSERVABILITY / SCHEDULES"]
    CW[CloudWatch\nLogs & Alarms]
    EB[EventBridge\nSchedules]
  end

  %% Static hosting
  U -->|HTTPS| CF
  Admin -->|HTTPS| CF
  CF --> S3

  %% Auth
  U -->|Login/Sign‑up| COG
  Admin -->|Login/Sign‑up| COG
  COG -->|OIDC/JWT| CF

  %% API path (authorized)
  CF -->|/api ... (JWT)| APIGW
  APIGW --> LPublic
  APIGW --> LEnroll
  APIGW --> LContact
  APIGW --> LAdmin

  %% Data layer
  LPublic --> DB
  LEnroll --> DB
  LAdmin --> DB
  LPayHook --> DB

  %% Payments
  CF -->|Create checkout session| Stripe
  Stripe -.->|Redirects / Hosted Page| U
  Stripe -->|Webhook (paid)| LPayHook

  %% Emails
  LEnroll -->|Enrollment confirmation| SES
  LPayHook -->|Receipt| SES
  LContact -->|Forward to admins| SES

  %% Observability
  LPublic --> CW
  LEnroll --> CW
  LPayHook --> CW
  LContact --> CW
  LAdmin --> CW
  EB -->|Nightly exports / jobs| LAdmin

  %% Notes
  classDef box fill:#fff,stroke:#888,rx:8,ry:8
  classDef store fill:#edf7ff,stroke:#4a90e2,rx:8,ry:8
  classDef ext fill:#fff7ed,stroke:#f39c12,rx:8,ry:8

  class CF,S3,APIGW,COG,LPublic,LEnroll,LPayHook,LContact,LAdmin,CW,EB,SES box
  class DB store
  class Stripe ext
