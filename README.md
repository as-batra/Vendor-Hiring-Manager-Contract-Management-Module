# Zelosify Task Monorepo

This monorepo is maintained for task and assessment purposes.

It contains:

- `Zelosify-Backend` for backend services and APIs
- `Zelosify-Frontend` for frontend application code

Tasks to do=>
#1 Persona 1 – IT Vendor

IT Vendor User Story
Want To:
• View available contract openings
• View opening details
• Upload multiple candidate profiles (PDF, PPTX)
• Soft delete profiles
• Securely submit profiles
So That:
I can participate in filling contractual openings while ensuring secure, tenant-isolated file
sharing.

#2 Authentication & Authorization

Role
Role: IT_VENDOR
Rules
• Can view openings under their tenant
• Can upload profiles
• Can view their own uploads
• Cannot view other vendors' uploads
• Cannot view AI recommendation
• Cannot shortlist/reject
Mandatory Enforcement
• API-level RBAC
• UI-level route guards
• Tenant-based query filtering
• No bypassable endpoints

#3 Database Schema

1. Opening Model
   model Opening {
   id String @id @default(uuid())
   tenantId String
   title String
   description String?
   location String?
   contractType String?
   hiringManagerId String

experienceMin Int
experienceMax Int?
postedDate DateTime @default(now())
expectedCompletionDate DateTime?
actionDate DateTime?
status OpeningStatus @default(OPEN)
tenant Tenants @relation(fields: [tenantId], references: [tenantId])
hiringProfiles hiringProfile[]
@@index([tenantId])
}

2. Profile Model
   model hiringProfile {
   id Int @id @default(autoincrement())
   openingId String
   s3Key String @unique
   uploadedBy String
   submittedAt DateTime @default(now())
   status ProfileStatus @default(SUBMITTED)
   shortlistedBy String?
   shortlistedAt DateTime?
   rejectedBy String?
   rejectedAt DateTime?
   // AI Agent Fields
   recommended Boolean?
   recommendationScore Float?
   recommendationReason String?
   recommendationLatencyMs Int?
   recommendationVersion String?
   recommendationConfidence Float?
   recommendedAt DateTime?
   isDeleted Boolean @default(false)
   opening Opening @relation(fields: [openingId], references: [id])
   @@index([openingId])
   @@index([recommended])
   }

3.Enums
enum OpeningStatus {
OPEN
CLOSED
ON_HOLD
}
enum ProfileStatus {

SUBMITTED
SHORTLISTED
REJECTED
}

#4. Seeding Requirements

• Seed must pre-populate:
• Tenant: "Bruce Wayne Corp"
• At least 12 openings
• Different roles, experience ranges, contract types
• All seed data must belong to the same tenant.
Tenant isolation must remain intact.

#5. IT Vendor Backend APIs

1.Fetch Openings
GET /api/vendor/openings
Pagination required.
Tenant filtering mandatory.

2️.Fetch Opening Details
GET /api/vendor/openings/:id
Must include:
• Hiring Manager name
• Experience range
• Profiles count
• Uploaded profiles list

3️.Presign Upload URLs
POST /api/vendor/openings/:id/profiles/presign
Profiles stored under:
<bucket>/<tenantId>/<openingId>/<timestamp>\_<filename>
No frontend → direct S3️ access.

4️.Submit Profiles
POST /api/vendor/openings/:id/profiles/upload
Must use Prisma transaction.

#6. IT Vendor Frontend

1. Route
   /vendor/openings
   Table Columns
   • Title
   • Location
   • Contract Type
   • Posted Date
   • Hiring Manager Name

2. Opening Details
   /vendor/openings/:id
   Must include:
   • Opening details
   • Upload section
   • Drag-drop support
   • Multiple upload

• Soft delete support
• File preview via backend
• Light & dark theme

Persona 2️ – Hiring Manager

#1. Hiring Manager User Story
I Want To:
• View my openings
• View submitted profiles
• See AI recommendation per profile
• See score + explanation
• Shortlist or reject
So That:
I can efficiently filter high-quality candidates using AI-assisted decision support.

#2. Authentication & Authorization
Role
Role: HIRING_MANAGER
Rules

Action Allowed
View own openings
View submitted profiles
Upload profiles
See other manager openings

Action Allowed
Shortlist/Reject

Mandatory condition:
opening.hiringManagerId === loggedInUser.id

AI Recommendation Agent (MANDATORY LLM
Tool-Using Agent, use gemini/groq for free models)

#1. This system must implement a real LLM-based Agent with tool-calling capability.
The agent must:

1. Use an LLM that supports tool/function calling.
2. Dynamically decide when to invoke:
   o Resume Parsing Tool
   o Feature Extraction Tool
   o Skill Normalization Tool
3. Use structured output schema validation.
4. Prevent prompt injection from resume content.
5. Implement retry logic for malformed outputs.
6. Maintain internal reasoning state.
7. Log token usage and latency.
8. Persist intermediate reasoning metadata.
   Deterministic-only logic without LLM reasoning will result in rejection.
   Calling an LLM once and directly mapping output to DB without orchestration is not
   acceptable.

Resume Parsing(Tool-Based via Agent)

Resume parsing must be implemented as a callable tool.
The LLM agent must:
• Decide when resume parsing is required.
• Retrieve file securely from S3.
• Invoke Resume Parsing Tool.
• Extract structured schema:
{
experienceYears: number,
skills: string[],
normalizedSkills: string[],
location: string,
education: string[],

keywords: string[]
}
Note
Resume content must never be directly injected into LLM system prompts without sanitization.
Prompt injection mitigation is mandatory.
Must support:
• PDF
• PPTX
Must use backend S3 retrieval.

Feature Vector
{
experienceYears: number,
skills: string[],
location: string,
skillMatchScore: number,
experienceMatchScore: number,
locationMatchScore: number
}

Matching & Scoring Engine (Invoked as Tool)
The deterministic matching logic must be implemented as a separate tool callable by the LLM
agent.
The agent must:
• Pass structured feature vector to matching tool.
• Receive scoring breakdown.
• Use this result in reasoning step.
• Produce explanation based on scoring + reasoning.
Hardcoded controller-level scoring without agent invocation is not acceptable
