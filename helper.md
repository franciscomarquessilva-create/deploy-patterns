Angola Digital Health Hub — Web App Build Instructions for GitHub Copilot
Save as .github/copilot-instructions.md at the repo root.

markdown# Copilot Instructions — Angola Digital Health Hub (Web App)

> Agent context file. Read this before generating any code, config, or
> infrastructure for this project. All decisions here are final unless
> the user explicitly overrides them in the current session.

---

## Deployment Pattern

This project uses the Cloudflare Tunnel + Traefik + Docker pattern
defined in DEPLOY.md. The summary:

```
Internet
   ↓
Cloudflare DNS (*.yourdomain.com)  ← wildcard CNAME → tunnel
   ↓
Cloudflare Tunnel (cloudflared)    ← outbound-only, no open ports
   ↓
Traefik Reverse Proxy              ← routes via Docker labels
   ↓
Next.js App Container              ← on the `proxy` Docker network
```

**TLS is handled by Cloudflare at the edge. Do not add cert-manager,
Let's Encrypt, or any TLS config inside the stack.**

**No ports are open on the host. Do not add host port mappings to
the app container.**

---

## Repository Structure

```
angola-health-hub/
├── app/                        # Next.js 14 App Router
│   ├── (public)/               # Unauthenticated routes
│   │   ├── page.tsx            # Home / marketing
│   │   ├── services/page.tsx
│   │   ├── model/page.tsx
│   │   └── contact/page.tsx
│   ├── (auth)/                 # Auth-gated routes
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── (portal)/               # Role-based portal
│   │   ├── layout.tsx          # Portal shell + role guard
│   │   ├── dashboard/page.tsx
│   │   ├── triage/
│   │   │   ├── page.tsx        # Start triage
│   │   │   └── [id]/page.tsx   # Active session
│   │   ├── consultation/
│   │   │   ├── page.tsx        # My consultations list
│   │   │   └── [id]/page.tsx   # Join / review session
│   │   ├── prescriptions/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   ├── physiotherapy/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   ├── health-profile/
│   │   │   └── page.tsx        # 360° patient profile
│   │   └── admin/
│   │       └── page.tsx        # Admin dashboard (role-gated)
│   ├── api/                    # Next.js API routes
│   │   ├── auth/[...nextauth]/route.ts
│   │   ├── trpc/[trpc]/route.ts
│   │   └── webhooks/
│   └── layout.tsx              # Root layout
├── components/
│   ├── ui/                     # shadcn/ui primitives
│   ├── layout/                 # TopNav, SiteFooter, PortalShell
│   ├── triage/                 # SymptomSelector, TriageOutcomeCard
│   ├── consultation/           # VideoFrame, SupervisionBanner
│   ├── prescription/           # PrescriptionCard, QRDisplay
│   ├── physiotherapy/          # ExercisePlan, SessionCard
│   ├── health-profile/         # ProfileSummary, VitalSignsChart
│   └── shared/                 # StatCard, ServiceCard, SectionLabel
├── server/
│   ├── db.ts                   # Prisma client singleton
│   ├── auth.ts                 # NextAuth config
│   ├── trpc.ts                 # tRPC init, context, middleware
│   ├── routers/                # tRPC routers (one file per domain)
│   │   ├── _app.ts             # Root router
│   │   ├── auth.ts
│   │   ├── patient.ts
│   │   ├── triage.ts
│   │   ├── consultation.ts
│   │   ├── prescription.ts
│   │   ├── physiotherapy.ts
│   │   ├── healthProfile.ts
│   │   └── admin.ts
│   └── services/               # Business logic, external integrations
│       ├── triage-engine.ts    # Calls Python FastAPI triage service
│       ├── video.ts            # 100ms.live or Daily.co session mgmt
│       ├── notifications.ts    # Twilio SMS
│       └── prescription-qr.ts # QR code generation
├── lib/
│   ├── trpc-client.ts          # tRPC React client
│   ├── supabase.ts             # Supabase browser client
│   ├── fhir/                   # HL7 FHIR R4 mappers
│   └── utils.ts
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── public/
├── docker-compose.yml          # App service — follows DEPLOY.md pattern
├── Dockerfile
├── .env.example
├── .env                        # Never commit — listed in .gitignore
├── next.config.ts
├── tailwind.config.ts
└── .github/
    ├── copilot-instructions.md ← this file
    └── workflows/
        └── deploy.yml
```

---

## Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router, TypeScript) |
| Styling | Tailwind CSS + shadcn/ui |
| Icons | lucide-react |
| API | tRPC (server + client in same repo) |
| Database | PostgreSQL via Supabase |
| ORM | Prisma |
| Auth | NextAuth.js v5 (Supabase adapter) |
| File storage | Supabase Storage |
| Video | 100ms.live SDK |
| Notifications | Twilio SMS |
| QR codes | `qrcode` npm package |
| Container | Docker (single container, Next.js standalone output) |
| Reverse proxy | Traefik (via Docker labels — see DEPLOY.md) |
| Tunnel | Cloudflare Tunnel (cloudflared) |

---

## Environment Variables — .env.example

```env
# App
NEXT_PUBLIC_APP_URL=https://health.yourdomain.com
NEXTAUTH_SECRET=
NEXTAUTH_URL=https://health.yourdomain.com

# Database
DATABASE_URL=postgresql://user:pass@host:5432/angola_health
DIRECT_URL=postgresql://user:pass@host:5432/angola_health

# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Video (100ms.live)
HMS_APP_ACCESS_KEY=
HMS_APP_SECRET=

# Triage engine (internal service URL)
TRIAGE_ENGINE_URL=http://triage-engine:8001

# SMS
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=
```

Never hardcode any of these values. Always read from `process.env`.

---

## Dockerfile

Use Next.js standalone output for a minimal production image.

```dockerfile
# Dockerfile
FROM node:20-alpine AS base

# --- deps ---
FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# --- builder ---
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN npm run build

# --- runner ---
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000
CMD ["node", "server.js"]
```

In `next.config.ts`, add:

```ts
const nextConfig = {
  output: 'standalone',
}
export default nextConfig
```

---

## docker-compose.yml — App Service

Follows the DEPLOY.md service pattern exactly. No host port mappings.
No TLS config. `proxy` network is external.

```yaml
services:

  angola-health-web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: angola-health-web
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.angola-health-web.rule=Host(`health.yourdomain.com`)"
      - "traefik.http.routers.angola-health-web.entrypoints=web"
      - "traefik.http.services.angola-health-web.loadbalancer.server.port=3000"

networks:
  proxy:
    external: true
```

**Label rules (from DEPLOY.md — never break these):**
- All three label name segments must be identical: `angola-health-web`
- `loadbalancer.server.port` must be `3000` — the port Next.js listens on
  inside the container, not a host-mapped port
- Never add `ports:` to this service — tunnel handles ingress

---

## GitHub Actions — .github/workflows/deploy.yml

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        run: |
          docker build -t angola-health-web:latest .

      - name: Copy image to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          source: "docker-compose.yml,.env.example"
          target: "/opt/angola-health"

      - name: Deploy on server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/angola-health

            # Pre-deploy validation (from DEPLOY.md)
            docker ps --filter "name=traefik" --filter "name=cloudflared" \
              --format "table {{.Names}}\t{{.Status}}"

            # Pull latest and redeploy
            docker compose pull
            docker compose up -d --build angola-health-web

            # Verify
            docker ps --filter "name=angola-health-web"
```

**Required GitHub Secrets:**
- `SERVER_HOST` — IP or hostname of the server
- `SERVER_USER` — SSH user
- `SERVER_SSH_KEY` — private key (Cloudflare Zero Trust SSH or direct)

---

## Pre-Deploy Validation (from DEPLOY.md)

Before deploying or modifying any container, run these checks:

```bash
# 1. Gateway is running
docker ps --filter "name=traefik" --filter "name=cloudflared" \
  --format "table {{.Names}}\t{{.Status}}"

# 2. proxy network exists
docker network ls --filter "name=proxy" --format "{{.Name}}"

# 3. No name collision
docker ps -a --filter "name=angola-health-web" \
  --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 4. Verify app is reachable after deploy
curl -I https://health.yourdomain.com
docker logs angola-health-web --tail 30
```

---

## Database Schema — prisma/schema.prisma

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

// ─── IDENTITY ────────────────────────────────────────────

model User {
  id           String     @id @default(cuid())
  email        String     @unique
  phone        String?
  passwordHash String?
  role         UserRole
  status       UserStatus @default(PENDING)
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt

  profile    UserProfile?
  patient    Patient?
  clinician  Clinician?
  student    MedicalStudent?
  auditLogs  AuditLog[]
}

enum UserRole {
  PATIENT
  MEDICAL_STUDENT
  SUPERVISING_CLINICIAN
  PHYSIOTHERAPIST
  PHARMACIST
  ADMINISTRATOR
}

enum UserStatus {
  PENDING
  ACTIVE
  SUSPENDED
}

model UserProfile {
  id          String    @id @default(cuid())
  userId      String    @unique
  firstName   String
  lastName    String
  dateOfBirth DateTime?
  gender      String?
  province    String?
  language    String    @default("pt")
  user        User      @relation(fields: [userId], references: [id])
}

// ─── PATIENT & HEALTH PROFILE ───────────────────────────

model Patient {
  id                String    @id @default(cuid())
  userId            String    @unique
  nationalId        String?   @unique
  bloodType         String?
  allergies         String[]
  chronicConditions String[]
  consentGiven      Boolean   @default(false)
  user              User      @relation(fields: [userId], references: [id])

  healthProfile  HealthProfile?
  triageSessions TriageSession[]
  consultations  Consultation[]
  prescriptions  Prescription[]
  physioSessions PhysioSession[]
  appointments   Appointment[]
}

model HealthProfile {
  id          String   @id @default(cuid())
  patientId   String   @unique
  summary     String?
  lastUpdated DateTime @updatedAt
  patient     Patient  @relation(fields: [patientId], references: [id])

  vitalSigns    VitalSign[]
  diagnoses     Diagnosis[]
  medications   Medication[]
  documents     MedicalDocument[]
}

model VitalSign {
  id              String        @id @default(cuid())
  healthProfileId String
  type            VitalSignType
  value           Float
  unit            String
  recordedAt      DateTime
  recordedBy      String
  healthProfile   HealthProfile @relation(fields: [healthProfileId], references: [id])
}

enum VitalSignType {
  BLOOD_PRESSURE_SYSTOLIC
  BLOOD_PRESSURE_DIASTOLIC
  HEART_RATE
  TEMPERATURE
  OXYGEN_SATURATION
  WEIGHT
  HEIGHT
  BLOOD_GLUCOSE
}

model Diagnosis {
  id              String        @id @default(cuid())
  healthProfileId String
  icd10Code       String
  description     String
  status          DiagnosisStatus
  diagnosedAt     DateTime
  diagnosedBy     String
  healthProfile   HealthProfile @relation(fields: [healthProfileId], references: [id])
}

enum DiagnosisStatus {
  ACTIVE
  RESOLVED
  CHRONIC
  SUSPECTED
}

model Medication {
  id              String        @id @default(cuid())
  healthProfileId String
  name            String
  dosage          String
  frequency       String
  startDate       DateTime
  endDate         DateTime?
  active          Boolean       @default(true)
  healthProfile   HealthProfile @relation(fields: [healthProfileId], references: [id])
}

model MedicalDocument {
  id              String        @id @default(cuid())
  healthProfileId String
  name            String
  type            String
  storageUrl      String
  uploadedAt      DateTime      @default(now())
  uploadedBy      String
  healthProfile   HealthProfile @relation(fields: [healthProfileId], references: [id])
}

// ─── CLINICIANS ──────────────────────────────────────────

model Clinician {
  id            String    @id @default(cuid())
  userId        String    @unique
  licenceNumber String    @unique
  speciality    String
  institution   String
  node          HubNode
  user          User      @relation(fields: [userId], references: [id])

  consultations      Consultation[]
  prescriptions      Prescription[]
  supervisionRecords SupervisionRecord[]
  supervisedStudents MedicalStudent[]
}

model MedicalStudent {
  id            String    @id @default(cuid())
  userId        String    @unique
  studentNumber String    @unique
  institution   String
  yearOfStudy   Int
  supervisorId  String
  node          HubNode
  user          User      @relation(fields: [userId], references: [id])
  supervisor    Clinician @relation(fields: [supervisorId], references: [id])

  consultations      Consultation[]
  supervisionRecords SupervisionRecord[]
}

enum HubNode {
  LUANDA
  LISBON
}

// ─── TRIAGE ──────────────────────────────────────────────

model TriageSession {
  id          String         @id @default(cuid())
  patientId   String
  startedAt   DateTime       @default(now())
  completedAt DateTime?
  symptoms    Json
  aiSuggestion Json?
  triageLevel TriageLevel?
  outcome     TriageOutcome?
  patient     Patient        @relation(fields: [patientId], references: [id])
  appointment Appointment?
}

enum TriageLevel {
  GREEN   // Self-care
  YELLOW  // Teleconsultation within 24h
  ORANGE  // Teleconsultation within 4h
  RED     // Urgent physical referral
}

enum TriageOutcome {
  SELF_CARE_ADVISED
  TELECONSULTATION_SCHEDULED
  PHYSICAL_REFERRAL_ISSUED
  EMERGENCY_DISPATCHED
}

// ─── CONSULTATIONS ───────────────────────────────────────

model Consultation {
  id               String             @id @default(cuid())
  patientId        String
  conductedById    String?
  supervisorId     String?
  scheduledAt      DateTime
  startedAt        DateTime?
  endedAt          DateTime?
  status           ConsultationStatus @default(SCHEDULED)
  videoSessionId   String?
  notes            String?
  clinicalSummary  String?
  followUpRequired Boolean            @default(false)
  patient          Patient            @relation(fields: [patientId], references: [id])
  student          MedicalStudent?    @relation(fields: [conductedById], references: [id])
  supervisor       Clinician?         @relation(fields: [supervisorId], references: [id])
  supervisionRecord SupervisionRecord?
  prescriptions    Prescription[]
  physioReferrals  PhysioReferral[]
}

enum ConsultationStatus {
  SCHEDULED
  IN_PROGRESS
  AWAITING_SUPERVISION
  COMPLETED
  CANCELLED
  NO_SHOW
}

model SupervisionRecord {
  id             String       @id @default(cuid())
  consultationId String       @unique
  studentId      String
  supervisorId   String
  submittedAt    DateTime     @default(now())
  reviewedAt     DateTime?
  approved       Boolean?
  feedback       String?
  coSignedAt     DateTime?
  consultation   Consultation  @relation(fields: [consultationId], references: [id])
  student        MedicalStudent @relation(fields: [studentId], references: [id])
  supervisor     Clinician     @relation(fields: [supervisorId], references: [id])
}

// ─── PRESCRIPTIONS ───────────────────────────────────────

model Prescription {
  id             String             @id @default(cuid())
  patientId      String
  prescriberId   String
  consultationId String?
  issuedAt       DateTime           @default(now())
  expiresAt      DateTime
  status         PrescriptionStatus @default(DRAFT)
  qrCodeUrl      String?
  items          PrescriptionItem[]
  dispensingLog  DispensingRecord[]
  patient        Patient            @relation(fields: [patientId], references: [id])
  prescriber     Clinician          @relation(fields: [prescriberId], references: [id])
  consultation   Consultation?      @relation(fields: [consultationId], references: [id])
}

enum PrescriptionStatus {
  DRAFT
  AWAITING_SUPERVISION
  ACTIVE
  PARTIALLY_DISPENSED
  FULLY_DISPENSED
  EXPIRED
  CANCELLED
}

model PrescriptionItem {
  id             String       @id @default(cuid())
  prescriptionId String
  medicationName String
  dosage         String
  frequency      String
  durationDays   Int
  instructions   String?
  quantity       Int
  prescription   Prescription @relation(fields: [prescriptionId], references: [id])
}

model DispensingRecord {
  id             String       @id @default(cuid())
  prescriptionId String
  pharmacistId   String
  pharmacyName   String
  dispensedAt    DateTime     @default(now())
  itemsDispensed Json
  prescription   Prescription @relation(fields: [prescriptionId], references: [id])
}

// ─── PHYSIOTHERAPY ───────────────────────────────────────

model PhysioReferral {
  id             String        @id @default(cuid())
  consultationId String
  patientId      String
  reason         String
  urgency        String
  status         String        @default("PENDING")
  consultation   Consultation  @relation(fields: [consultationId], references: [id])
  sessions       PhysioSession[]
}

model PhysioSession {
  id               String          @id @default(cuid())
  patientId        String
  physiotherapistId String
  referralId       String?
  scheduledAt      DateTime
  startedAt        DateTime?
  endedAt          DateTime?
  sessionType      PhysioSessionType
  videoSessionId   String?
  exercisePlan     Json?
  progressNotes    String?
  adherenceScore   Float?
  patient          Patient         @relation(fields: [patientId], references: [id])
}

enum PhysioSessionType {
  LIVE_VIDEO
  ASYNC_GUIDED
  FOLLOW_UP
}

// ─── APPOINTMENTS ────────────────────────────────────────

model Appointment {
  id              String            @id @default(cuid())
  patientId       String
  triageSessionId String?           @unique
  type            AppointmentType
  scheduledAt     DateTime
  status          AppointmentStatus @default(PENDING)
  patient         Patient           @relation(fields: [patientId], references: [id])
  triageSession   TriageSession?    @relation(fields: [triageSessionId], references: [id])
}

enum AppointmentType {
  TELECONSULTATION
  TELEPHYSIOTHERAPY
  PHYSICAL_REFERRAL
}

enum AppointmentStatus {
  PENDING
  CONFIRMED
  COMPLETED
  CANCELLED
  NO_SHOW
}

// ─── AUDIT ───────────────────────────────────────────────

model AuditLog {
  id         String   @id @default(cuid())
  userId     String
  action     String
  resource   String
  resourceId String
  metadata   Json?
  createdAt  DateTime @default(now())
  user       User     @relation(fields: [userId], references: [id])
}
```

---

## tRPC Setup — server/trpc.ts

```typescript
import { initTRPC, TRPCError } from '@trpc/server'
import { getServerSession } from 'next-auth'
import { authOptions } from './auth'
import { db } from './db'
import superjson from 'superjson'
import { UserRole } from '@prisma/client'

export const createContext = async () => {
  const session = await getServerSession(authOptions)
  return { db, session, user: session?.user ?? null }
}

const t = initTRPC.context().create({
  transformer: superjson,
})

export const router = t.router
export const publicProcedure = t.procedure

export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.user) throw new TRPCError({ code: 'UNAUTHORIZED' })
  return next({ ctx: { ...ctx, user: ctx.user } })
})

// Role guard factory
export const roleGuard = (...roles: UserRole[]) =>
  protectedProcedure.use(({ ctx, next }) => {
    if (!roles.includes(ctx.user.role as UserRole)) {
      throw new TRPCError({ code: 'FORBIDDEN' })
    }
    return next()
  })

export const adminProcedure      = roleGuard('ADMINISTRATOR')
export const clinicianProcedure  = roleGuard('SUPERVISING_CLINICIAN', 'ADMINISTRATOR')
export const studentProcedure    = roleGuard('MEDICAL_STUDENT', 'SUPERVISING_CLINICIAN', 'ADMINISTRATOR')
export const pharmacistProcedure = roleGuard('PHARMACIST', 'ADMINISTRATOR')
export const patientProcedure    = roleGuard('PATIENT')
```

---

## Supervision Enforcement Rule

Every action taken by a `MEDICAL_STUDENT` that writes to the patient
record must be held in `AWAITING_SUPERVISION` status until a
`SUPERVISING_CLINICIAN` explicitly co-signs it. This applies to:

- `consultation.complete` → status stays `AWAITING_SUPERVISION`
- `prescription.create` → status `DRAFT` → `AWAITING_SUPERVISION`
- `diagnosis.add` → flagged for review

Implement this check in every relevant tRPC mutation:

```typescript
if (ctx.user.role === 'MEDICAL_STUDENT') {
  // Do not commit as final — create supervision record and notify
  await ctx.db.supervisionRecord.create({ ... })
  // Return pending state to client
}
```

---

## Prescription State Machine

```
DRAFT → AWAITING_SUPERVISION → ACTIVE → PARTIALLY_DISPENSED → FULLY_DISPENSED
                                      ↘ EXPIRED
              ↘ CANCELLED (any state before ACTIVE)
```

Implement transitions as explicit tRPC mutations — never allow direct
status field updates from the client.

---

## Design Tokens — tailwind.config.ts

```ts
colors: {
  brand: {
    darkTeal:   '#024A63',
    teal:       '#065A82',
    midTeal:    '#0A7EA4',
    lightTeal:  '#D6EBF5',
    amber:      '#E8A838',
    amberLight: '#FFF3D6',
    charcoal:   '#1A2E3B',
    muted:      '#4A6B7A',
    mutedLight: '#8AAAB8',
  }
}
```

---

## Build Order for Copilot

Work in this sequence — each step unblocks the next:

1. **`prisma/schema.prisma`** + `prisma/seed.ts`
2. **`server/db.ts`** + **`server/auth.ts`** + **`server/trpc.ts`**
3. **`server/routers/_app.ts`** + all domain routers (skeleton first)
4. **`lib/trpc-client.ts`** + `app/api/trpc/[trpc]/route.ts`
5. **`Dockerfile`** + **`docker-compose.yml`** (app service)
6. **`app/(public)/`** — marketing pages
7. **`app/(auth)/`** — login + register
8. **`app/(portal)/`** — service flows in this order:
   - `triage/`
   - `consultation/`
   - `prescriptions/`
   - `physiotherapy/`
   - `health-profile/`
   - `admin/`
9. **`.github/workflows/deploy.yml`**

---

## Suggested Copilot Prompts Per Step

```
"Generate prisma/seed.ts with 2 patients, 1 medical student,
 1 supervising clinician (Luanda node), 1 pharmacist"

"Build the triage tRPC router following the schema and supervision
 rules in copilot-instructions.md"

"Scaffold the consultation/[id]/page.tsx — fetch consultation via tRPC,
 render VideoFrame if IN_PROGRESS, show SupervisionBanner if
 AWAITING_SUPERVISION"

"Generate docker-compose.yml for the Next.js app service using the
 Traefik label pattern in copilot-instructions.md — no host ports,
 proxy network external"

"Build the prescription state machine mutations in
 server/routers/prescription.ts — enforce supervision guard for
 MEDICAL_STUDENT role"
```
