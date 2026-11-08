<div align="center">

# GNTI | Vocalis

<p><strong>Enterprise-Grade AI Voice & Audio Platform with Zero-Shot Voice Cloning & Serverless GPU Inference</strong></p>

<p>A flagship production foundation engineered by <a href="https://gnti.tech">Galaxy Nova Technologies & Innovations (GNTI)</a>.</p>

<p>
  <a href="https://gnti.tech"><img src="https://img.shields.io/badge/GNTI-gnti.tech-000000?style=for-the-badge" alt="GNTI" /></a>&nbsp;
  <a href="https://github.com/GNTI-X/vocalis"><img src="https://img.shields.io/badge/Repository-GNTI--X%2Fvocalis-24292e?style=for-the-badge&logo=github" alt="GitHub Repo" /></a>&nbsp;
  <a href="./LICENSE"><img src="https://img.shields.io/badge/License-MIT-3b82f6?style=for-the-badge" alt="MIT License" /></a>&nbsp;
  <a href="https://gnti.tech/projects/vocalis"><img src="https://img.shields.io/badge/Status-Production%20Ready-10b981?style=for-the-badge" alt="Production Ready" /></a>
</p>

</div>

---

## 📌 1. System Overview

**Vocalis** is an open-source, enterprise-grade AI voice synthesis and zero-shot voice cloning platform designed as a scalable foundation for voice applications. It empowers organizations to synthesize studio-quality speech with fine-grained expressive control and clone target voices from just a 10-second reference audio sample—without fine-tuning or model retraining.

Vocalis couples a reactive Next.js 16 / React 19 interface with an asynchronous serverless GPU inference pipeline powered by Chatterbox TTS and Modal. Audio assets are stored in Cloudflare R2, metered character-by-character through Polar billing, and delivered with sub-second latency and interactive waveform analysis via WaveSurfer.js.

---

## 🏛 2. Architecture & System Flow

### System Architecture Diagram

```mermaid
graph TB
    subgraph Client ["Client Layer (Next.js 16 / React 19)"]
        UI[Audio Studio UI]
        Wave[WaveSurfer.js Visualizer]
        ClerkClient[Clerk Auth & Org Switcher]
    end

    subgraph API ["Application Server (Next.js & tRPC)"]
        TRPC[tRPC API Router]
        AuthGuard[Clerk Auth & Multi-Tenant Guard]
        BillingService[Polar Metering & Credit Guard]
    end

    subgraph DB ["Data & Storage Layer"]
        Postgres[(PostgreSQL via Prisma)]
        R2[(Cloudflare R2 Object Storage)]
    end

    subgraph Inference ["AI Inference Infrastructure (Modal)"]
        ModalAPI[Modal Serverless GPU Runner]
        Chatterbox[Chatterbox TTS / FastSpeech Engine]
    end

    UI -->|1. Generate Speech / Clone Voice| TRPC
    TRPC -->|2. Verify Session & Permissions| AuthGuard
    TRPC -->|3. Validate Character Credits| BillingService
    TRPC -->|4. Dispatch Synthesis Request| ModalAPI
    ModalAPI -->|5. Run Audio Generation| Chatterbox
    Chatterbox -->|6. Upload Output Audio WAV/MP3| R2
    TRPC -->|7. Persist Generation Metadata| Postgres
    TRPC -->|8. Record Usage & Deduct Credits| BillingService
    R2 -->|9. Presigned Audio Stream| Wave
```

### End-to-End Workflow

1. **Authentication & Multi-Tenant Routing**:
   - The user authenticates via Clerk. Requests carry organizational metadata (`orgId`), enforcing strict multi-tenant boundary checks across voice models and generation histories.
2. **Request Dispatch & Credit Metering**:
   - Client sends text synthesis parameters (text payload, selected voice ID, speed, expression, flow) via type-safe tRPC.
   - The server verifies available character credits against Polar before committing compute resources.
3. **Serverless GPU Inference**:
   - The server calls a dedicated Modal runner hosting the Chatterbox TTS model.
   - For custom voice cloning, a 10-second WAV reference previously uploaded to Cloudflare R2 is passed directly to the inference container for zero-shot speaker embedding extraction.
4. **Storage & Delivery**:
   - Synthesized audio is uploaded with unique UUID keys directly into Cloudflare R2.
   - Presigned S3 URLs are generated and returned to the client.
5. **Interactive Playback & Analytics**:
   - The client renders the audio stream with real-time waveform visualization, scrub controls, playback velocity modifications, and one-click master downloads.

---

## 🛠 3. Categorized Tech Stack

| Domain | Technology | Purpose |
|:---|:---|:---|
| **Frontend Framework** | `Next.js 16` (App Router) & `React 19` | High-performance server/client rendered UI with optimistic updates |
| **Styling & Components** | `Tailwind CSS`, `Radix UI`, `Lucide Icons` | Modern responsive interface with accessible design primitives |
| **Audio Visualization** | `WaveSurfer.js` | Interactive audio waveform rendering, audio scrubbing, and time markers |
| **API Layer** | `tRPC v11` & `Zod` | End-to-end type safety between frontend mutations and backend procedures |
| **Database & ORM** | `PostgreSQL` via `Prisma ORM` | Relational persistence for voices, audio generation logs, and settings |
| **AI & Inference** | `Chatterbox TTS`, `Modal (GPU Serverless)` | Low-latency voice cloning and text-to-speech inference |
| **Object Storage** | `Cloudflare R2` (AWS S3-compatible SDK) | Zero-egress cost storage for audio samples and generated speech |
| **Authentication** | `Clerk` | Multi-tenant organization switching, user lifecycle, and role management |
| **Billing & Metering** | `Polar SDK` | Usage-based character tracking, checkout sessions, and credit subscriptions |

---

## 🗄 4. Domain Data Model & Entity Relationships

The PostgreSQL database (managed via Prisma) enforces strict multi-tenant isolation:

- **`Voice`**:
  - `id` (CUID, Primary Key)
  - `orgId` (Nullable string: `null` for built-in system voices, populated for custom team-cloned voices)
  - `name`, `description`, `category` (Enum: `AUDIOBOOK`, `CONVERSATIONAL`, `GENERAL`, `NARRATIVE`, `CHARACTERS`, etc.)
  - `language` (e.g. `en-US`, `es-ES`, `fr-FR`)
  - `variant` (`SYSTEM` vs `CUSTOM`)
  - `r2ObjectKey` (Audio reference path for cloned embeddings)
  - `generations` (One-to-Many relation with `Generation`)
- **`Generation`**:
  - `id` (CUID, Primary Key)
  - `orgId`, `userId`
  - `voiceId` (Foreign Key referencing `Voice`)
  - `text` (Original input script)
  - `characterCount` (Characters consumed for billing audits)
  - `audioUrl` / `r2Key` (Cloudflare R2 object identifier)
  - `settings` (JSON snapshot of temperature, speed, expression at time of run)
  - `createdAt` (Timestamp)

---

## 🔑 5. Environment Variables Reference

| Variable | Required | Description | Example / Source |
|:---|:---:|:---|:---|
| `DATABASE_URL` | **Yes** | PostgreSQL connection string | `postgresql://user:pass@host:5432/vocalis?sslmode=require` |
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | **Yes** | Clerk publishable frontend key | `pk_test_...` from Clerk Dashboard |
| `CLERK_SECRET_KEY` | **Yes** | Clerk private API key | `sk_test_...` from Clerk Dashboard |
| `MODAL_API_URL` | **Yes** | Endpoint to Modal serverless inference | `https://your-org--chatterbox-tts.modal.run` |
| `MODAL_TOKEN_ID` | Optional | Modal auth token identifier | Generated via `modal token new` |
| `MODAL_TOKEN_SECRET` | Optional | Modal auth secret | Generated via `modal token new` |
| `CLOUDFLARE_R2_ACCOUNT_ID` | **Yes** | Cloudflare account identifier | Cloudflare dashboard -> R2 |
| `CLOUDFLARE_R2_ACCESS_KEY_ID` | **Yes** | S3-compatible R2 API token key | Cloudflare R2 API tokens |
| `CLOUDFLARE_R2_SECRET_ACCESS_KEY` | **Yes** | S3-compatible R2 secret key | Cloudflare R2 API tokens |
| `CLOUDFLARE_R2_BUCKET_NAME` | **Yes** | Storage bucket for audio files | `vocalis-audio` |
| `POLAR_ACCESS_TOKEN` | Optional | Polar billing API token | `polar_at_...` from Polar Dashboard |
| `NEXT_PUBLIC_APP_URL` | **Yes** | Base application domain | `http://localhost:3000` |

---

## 🚀 6. Local Development & Deployment

### 1. Prerequisites
- **Node.js**: v18.18.0 or v20+
- **Package Manager**: `npm`, `pnpm`, or `bun`
- **PostgreSQL**: Running locally or via Neon/Supabase

### 2. Installation
```bash
git clone https://github.com/GNTI-X/vocalis.git
cd vocalis
npm install
```

### 3. Database Initialization
```bash
# Push Prisma schema to your PostgreSQL database
npx prisma db push

# (Optional) Seed default system voices
npx prisma db seed
```

### 4. Running the Development Server
```bash
npm run dev
```
Navigate to [http://localhost:3000](http://localhost:3000) to access the Vocalis studio.

### 5. Production Build
```bash
npm run build
npm run start
```

---

## 🏢 About GNTI

Galaxy Nova Technologies & Innovations (GNTI) builds and deploys scalable software across multi-tenant SaaS, real-time communication, AI automation, and media streaming infrastructure.

- **Website**: [gnti.tech](https://gnti.tech)
- **Official Specification**: [gnti.tech/projects/vocalis](https://gnti.tech/projects/vocalis)
- **Inquiries**: [contact@gnti.tech](mailto:contact@gnti.tech)

---

## 📄 License

Distributed under the **MIT License**. See [LICENSE](./LICENSE) for details.

Copyright &copy; 2026 [Galaxy Nova Technologies & Innovations (GNTI)](https://gnti.tech). All rights reserved.
