# Job Board Platform (Unique Features) — Project Spec

> Stack: **Next.js + NestJS + Prisma + PostgreSQL + pgvector + Redis/BullMQ + Socket.IO**  
> Differentiated job board: **AI resume–job matching scores**, skill assessments, employer **ATS pipeline**, salary insights, **anonymous / bias-reduced applications**, referrals, and interview scheduling.

---

## 1. Overview & Goals

Not a generic “post job / apply” clone. Unique pillars:
1. Embedding-based match score + explainable skill gaps.
2. Structured ATS kanban per job with scorecards.
3. Optional anonymous mode stripping demographic signals until onsite stage.
4. Timed skill assessments (auto-graded + manual).
5. Referral marketplace with bounty escrow.
6. Interview scheduling with timezone-safe slots + calendar sync stubs.
7. Salary insights from consented offer/outcome data (privacy-preserving aggregates).

---

## 2. Roles & Permissions

| Role | Permissions |
|------|-------------|
| `PLATFORM_ADMIN` | Moderate jobs, users, payouts, feature flags |
| `EMPLOYER_ADMIN` | Company, billing, members, all jobs |
| `EMPLOYER_RECRUITER` | Jobs assigned, ATS, schedule interviews |
| `EMPLOYER_HIRING_MANAGER` | Scorecards, feedback, limited candidate PII |
| `CANDIDATE` | Profile, resume, apply, assessments, referrals |
| `REFERRER` | (often same as candidate) submit referrals |

Company membership: `CompanyMember(role)`.

---

## 3. Feature Modules

1. Auth & companies  
2. Candidate profile, resume parse, skills ontology  
3. Job posting & moderation  
4. Search & recommendations  
5. AI matching & explanations  
6. Applications + anonymous mode  
7. ATS pipeline & scorecards  
8. Skill assessments  
9. Interview scheduling  
10. Referrals & bounties  
11. Salary insights  
12. Messaging / email notifications  
13. Employer billing (optional slots / seats)  

---

## 4. Advanced Business Logic

### 4.1 Resume–job matching
- On resume upload / job publish: chunk text → embedding (`text-embedding` provider adapter) → store `vector(1536)` in pgvector.
- Match score:
```
semantic = cosine(resumeEmb, jobEmb)           // 0..1
skillCoverage = |skills∩required| / |required|
experienceFit = f(years, level)
total = w1*semantic + w2*skillCoverage + w3*experienceFit
```
- Persist `MatchScore(jobId, candidateId, total, breakdown Json)`.
- Explanation: missing skills, weak semantic sections; never fabricate.
- Recompute async on profile change; cache top matches.

### 4.2 Anonymous applications
- When job.`anonymousEnabled` or candidate opts in: application presents `AnonymousPacket` — skills, experience years bands, assessment scores, portfolio links; hide name, photo, gender, age, university names (configurable redaction list).
- Reveal PII only on transition to stage `ONSITE` or `OFFER` with candidate consent event logged.
- Bias reduction metric: admin dashboard compares progression rates (aggregate only).

### 4.3 ATS pipeline
- Default stages: `APPLIED → SCREEN → ASSESSMENT → INTERVIEW → ONSITE → OFFER → HIRED` + `REJECTED`/`WITHDRAWN`.
- Custom stages per company with ordering; templates.
- Transitions enforce required scorecards / assessment pass thresholds.
- SLA timers: stage age alerts for recruiters.

### 4.4 Skill assessments
- Question bank: MCQ, short text, coding (link to external runner stub).
- Timed attempt; anti-cheat basics: tab blur count, paste detect flags (signal only).
- Auto-grade MCQ; manual rubric for text.
- Attach assessment to job stage gate: cannot move to INTERVIEW unless `score >= threshold` or waiver.

### 4.5 Interview scheduling
- Hiring team publishes availability windows; candidate picks slot; create `Interview` with idempotent calendar event ids.
- Conflict detection across interviewer calendars.
- Reschedule keeps audit; no-show marking.

### 4.6 Referrals
- Job sets `referralBountyMinor`; referrer submits candidate email/link.
- States: `SUBMITTED → ACCEPTED_BY_CANDIDATE → IN_PROCESS → HIRED → BOUNTY_PAYABLE → PAID | INELIGIBLE`.
- Duplicate detection: same candidate+job referral unique; employer employees ineligible.

### 4.7 Salary insights
- Collect voluntary `CompensationReport` on hire/offer (band, level, location, remote).
- Publish only aggregates with k-anonymity (`count >= 5` per bucket).
- API returns percentiles p25/p50/p75.

### 4.8 Edge cases
- Embedding provider outage → queue recompute; show “score pending”.  
- Candidate applies twice → idempotent unique(jobId,candidateId).  
- Employer views anonymous app → PII fields null in serializer.  
- Assessment time expiry mid-submit → accept if within grace 5s else reject.  
- Concurrent stage moves → version column optimistic lock.

---

## 5. State Machines

**Job**: `DRAFT → IN_REVIEW → PUBLISHED → PAUSED → CLOSED`  
**Application**: stage-based + `REJECTED`/`WITHDRAWN`  
**AssessmentAttempt**: `STARTED → SUBMITTED → GRADED | EXPIRED`  
**Interview**: `PROPOSED → CONFIRMED → COMPLETED | CANCELLED | NO_SHOW`  
**Referral**: see §4.6  
**RevealRequest**: `REQUESTED → APPROVED | DENIED`

---

## 6. Prisma Schema Draft (core)

```prisma
model User {
  id String @id @default(cuid())
  email String @unique
  passwordHash String
  role String
  fullName String
}
model Company {
  id String @id @default(cuid())
  name String
  slug String @unique
  status String
}
model CompanyMember {
  id String @id @default(cuid())
  companyId String
  userId String
  role String
  @@unique([companyId, userId])
}
model CandidateProfile {
  userId String @id
  headline String?
  yearsExperience Int?
  skills String[]
  location String?
  willingAnonymous Boolean @default(true)
  resumeText String?
  // embedding via Unsupported vector or separate table
}
model ResumeEmbedding {
  candidateId String @id
  embedding Unsupported("vector(1536)")
  updatedAt DateTime @updatedAt
}
model Job {
  id String @id @default(cuid())
  companyId String
  title String
  slug String
  description String
  requiredSkills String[]
  level String?
  location String?
  remoteType String?
  salaryMinMinor Int?
  salaryMaxMinor Int?
  currency String @default("BDT")
  status String
  anonymousEnabled Boolean @default(false)
  referralBountyMinor Int @default(0)
  publishedAt DateTime?
  @@index([companyId, status])
}
model JobEmbedding {
  jobId String @id
  embedding Unsupported("vector(1536)")
}
model MatchScore {
  id String @id @default(cuid())
  jobId String
  candidateId String
  total Float
  breakdown Json
  updatedAt DateTime @updatedAt
  @@unique([jobId, candidateId])
  @@index([jobId, total])
}
model Application {
  id String @id @default(cuid())
  jobId String
  candidateId String
  stage String
  isAnonymous Boolean @default(false)
  version Int @default(0)
  matchScore Float?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  @@unique([jobId, candidateId])
}
model StageEvent {
  id String @id @default(cuid())
  applicationId String
  fromStage String?
  toStage String
  byUserId String?
  at DateTime @default(now())
  meta Json?
}
model Scorecard {
  id String @id @default(cuid())
  applicationId String
  interviewerId String
  stage String
  ratings Json
  recommendation String
  @@unique([applicationId, interviewerId, stage])
}
model Assessment {
  id String @id @default(cuid())
  companyId String?
  title String
  durationMinutes Int
  passScore Int
}
model AssessmentQuestion {
  id String @id @default(cuid())
  assessmentId String
  type String
  prompt String
  options Json?
  answerKey Json?
  points Int @default(1)
}
model JobAssessment {
  jobId String
  assessmentId String
  gateStage String
  @@id([jobId, assessmentId])
}
model AssessmentAttempt {
  id String @id @default(cuid())
  assessmentId String
  candidateId String
  applicationId String?
  status String
  score Int?
  startedAt DateTime @default(now())
  submittedAt DateTime?
  signals Json?
}
model Interview {
  id String @id @default(cuid())
  applicationId String
  startsAt DateTime
  endsAt DateTime
  timezone String
  status String
  meetingUrl String?
  interviewerIds String[]
}
model Referral {
  id String @id @default(cuid())
  jobId String
  referrerId String
  candidateEmail String
  candidateId String?
  status String
  bountyMinor Int
  @@unique([jobId, candidateEmail])
}
model CompensationReport {
  id String @id @default(cuid())
  companyId String?
  jobLevel String
  location String
  remoteType String?
  baseMinor Int
  bonusMinor Int @default(0)
  currency String
  observedAt DateTime @default(now())
}
model OutboxEvent {
  id String @id @default(cuid())
  type String
  payload Json
  processedAt DateTime?
  createdAt DateTime @default(now())
}
```

---

## 7. REST API

- Candidate profile + resume upload  
- `GET /jobs/search`, `GET /jobs/:id/match` (auth)  
- `POST /jobs/:id/applications` (anonymous flag)  
- Employer ATS: `GET /employer/jobs/:id/pipeline`, `POST /applications/:id/transition`  
- `POST /applications/:id/scorecards`  
- Assessments CRUD + `POST /attempts`, `POST /attempts/:id/submit`  
- `POST /interviews`, availability endpoints  
- Referrals CRUD  
- `GET /insights/salary?level&location`  
- Admin moderation  

---

## 8. Real-time

- Recruiter pipeline board updates  
- Candidate interview reminders  
- Assessment timer sync (optional)

---

## 9. Jobs (BullMQ)

- Embedding compute / recompute  
- Match score batch for new jobs  
- Stage SLA reminders  
- Interview reminders  
- Referral bounty payout  
- Salary aggregate refresh materialized view  

---

## 10. Security & Privacy

- Strict serializers for anonymous applications.  
- Audit every PII reveal.  
- Encrypt resumes at rest; signed download URLs.  
- Rate-limit apply & assessment start.  
- GDPR/BD-style data export/delete for candidates (soft delete + anonymize applications).

---

## 11. Caching

- Job detail cache; search results short TTL.  
- Materialized top match lists in Redis per candidate.

---

## 12. Testing

- Unique application constraint.  
- Anonymous serializer strips fields (unit snapshot).  
- Match score weights fixtures.  
- Assessment expiry boundary.  
- Interview overlap for same interviewer rejected.  
- Salary API returns 404/empty when bucket &lt; 5.  
- Optimistic lock on transition conflict.

---

## 13. Folder Structure

```
job-board-platform/
  Project_spec.md
  frontend/  # (candidate) (employer) (admin)
  backend/
    src/modules/{jobs,matching,applications,ats,assessments,interviews,referrals,insights,embeddings}
    prisma/migrations/...._pgvector.sql
```

---

## 14. Env vars

`DATABASE_URL`, `REDIS_URL`, `JWT_*`, `APP_URL`, `CORS_ORIGINS`, `S3_*`, `SMTP_URL`, `SMS_*`, `EMBEDDING_PROVIDER`, `EMBEDDING_API_KEY`, `EMBEDDING_MODEL`, `MATCH_WEIGHT_SEMANTIC`, `MATCH_WEIGHT_SKILLS`, `MATCH_WEIGHT_EXPERIENCE`, `ASSESSMENT_SUBMIT_GRACE_MS`, `SALARY_MIN_BUCKET_SIZE`, `STRIPE_*`

---

## 15. Milestones

1. Auth, companies, jobs CRUD, apply  
2. Resume parse + embeddings + match scores  
3. ATS board + scorecards + anonymous mode  
4. Assessments  
5. Interviews  
6. Referrals + bounty  
7. Salary insights + polish  

---

## 16. Acceptance Criteria

- [ ] Applying twice to same job returns idempotent same application.  
- [ ] Anonymous application API hides name/photo/school until reveal stage.  
- [ ] Match endpoint returns breakdown with missing skills.  
- [ ] Assessment auto-expires and cannot submit after grace.  
- [ ] Pipeline transition blocked without required assessment pass.  
- [ ] Referral duplicate email+job rejected.  
- [ ] Salary insights withheld for small buckets.  
- [ ] pgvector extension migration present; similarity query works in integration test.  
- [ ] Seed: 2 companies, 10 jobs, 20 candidates with embeddings mocked.

---

## 17. Agent Notes

- `EmbeddingProvider` interface with `FakeEmbeddingProvider` for tests (hash→vector).  
- Keep ATS transition in one service with policy hooks.  
- Frontend employer board: optimistic UI + version conflict toast.  
- Document model fairness limitations in README (important for portfolio narrative).
