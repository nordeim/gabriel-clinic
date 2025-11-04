### Executive summary
This document is a meticulous, actionable master plan to transform Osler Health’s site by merging your strategic, emotionally resonant vision with the rigorous, implementation-ready transformation plan I produced earlier. It converts strategy into an end-to-end program: discovery, validated design, component-driven engineering, accessibility & privacy guardrails, analytics-driven rollout, and iterative optimisation. Each phase includes clear goals, deliverables, success criteria, test cases, owners, and risks — ready for immediate handoff to product, design, and engineering.

---

### Phase overview (high level)
1. Phase 0 — Technical Discovery & Governance (week 0–1)  
2. Phase 1 — IA, Content Audit, Accessibility & Analytics Contracts (week 1–2)  
3. Phase 2 — Design System, Wireframes & Prototypes (week 3–4)  
4. Phase 3 — Frontend Build, Booking Integration & CMS (week 5–9)  
5. Phase 4 — QA, Accessibility Remediation & Security (week 10–11)  
6. Phase 5 — Beta Launch, Monitoring, Staff Training (week 12)  
7. Phase 6 — Iteration, A/B Testing & Launch Optimisation (weeks 13–18)

Estimated total: 12–18 weeks depending on backend complexities and resource allocation.

---

### Phase 0 — Technical Discovery & Governance (Week 0–1)
Goal: Remove implementation unknowns and define governance.
Deliverables:
- PMS integration questionnaire and capabilities matrix (endpoints, auth, availability model, rate limiting, realtime vs batch, webhook support).
- Legal & compliance map: PDPA requirements, data residency, retention policies, and required documentation.
- Canonical clinician & clinic data schema (fields, types, ownership).
- Asset inventory template (images, videos, documents) and current size/bandwidth profile.
Acceptance criteria:
- PMS matrix completed and signed off by clinic IT/ops.
- Compliance owner assigned and PDPA constraints documented.
- Clinician schema signed by operations and clinicians.
Key tasks:
- Technical interview with PMS vendor or operations.
- Inventory export of clinicians, clinics, and existing content.
Owners: Product Lead, Clinic Ops, External PMS contact, Security/Legal.

Risks & mitigations:
- Risk: PMS offers no realtime slots -> Mitigation: design booking UI to show "next available" and gracefully handle last-second conflicts with optimistic reservations + background reconciliation.

---

### Phase 1 — IA, Content Audit, Accessibility & Analytics Contracts (Week 1–2)
Goal: Validate content, define structure, produce acceptance contracts for accessibility and analytics.
Deliverables:
- Content inventory with required enrichments (bios, credentials, languages, fees, durations, service tags, FAQs).
- Finalized information architecture (nav, hub taxonomy, service categories, clinic structure).
- Accessibility acceptance checklist (page & component level) — measurable pass/fail criteria.
- Analytics event spec and funnel definitions (booking + registration + profile views) with event names, properties, and dashboard owners.
Acceptance criteria:
- Content inventory >95% complete and mapped to new schema.
- IA signed off by stakeholders.
- Accessibility checklist defined and included in Definition of Done.
- Analytics spec reviewed and owners assigned.
Key tasks:
- Run automated accessibility and SEO baseline scans.
- Stakeholder workshops to align on IA and priority services.
Owners: Content Lead, UX Architect, Accessibility Lead, Analytics Engineer.

Example accessibility checks (component-level):
- Booking modal: focus trap test; aria-modal present; tab order functional; keyboard-only booking complete.
- Date picker: keyboard navigable; screen reader announcements for date selection.
- Typeahead search: aria-activedescendant, live region announcements.

Analytics key events (examples):
- booking_start, booking_select_clinic, booking_select_service, booking_select_time, booking_complete.
- registration_start, registration_complete, registration_upload_document.

---

### Phase 2 — Design System, Wireframes & Prototypes (Week 3–4)
Goal: Create pixel-consistent, accessible design system and validate flows with prototypes and usability tests.
Deliverables:
- Tokenized design system (colors, typography, spacing, motion tokens) and component library in Figma.
- Core components: header, sticky mobile action bar, service card, clinician card, clinic card, booking modal, multi-step form, file upload, toasts, skeleton loaders.
- Annotated low-fidelity and high-fidelity wireframes for: Home, Service Hub, Service Detail, Clinic Detail, Doctor Profile, Booking modal/full flow, New Patient Registration, Patient Hub (basic).
- Clickable prototype for booking and registration flows (desktop & mobile).
- Usability test script and 5 moderated remote tests (with target personas).
Acceptance criteria:
- Design system includes contrast tokens meeting WCAG AA; components include accessible states and focus styles.
- Prototype passes 5 core tasks with ≥80% success rate in moderated tests (book appointment, complete registration, find clinic hours).
- All interaction specs annotated with aria attributes and keyboard behaviour.
Key tasks:
- Create component usage rules and content patterns (microcopy, error states).
- Produce sample content for clinician bios and service pages.
Owners: Design Lead, UX Researcher, Content Lead.

Test cases (example):
- Task: Book a same-day appointment with Dr X. Success if user completes booking and receives confirmation screen within 3 steps.
- Task: Start new-patient registration, upload ID, save and resume. Success if saved progress is recoverable.

---

### Phase 3 — Frontend Build, Booking Integration & CMS (Week 5–9)
Goal: Implement production-ready UI components, integrate booking flows with PMS fallback strategies, and migrate content to headless CMS.
Deliverables:
- Next.js frontend with SSR/SSG where appropriate; component library documented in Storybook.
- Headless CMS (Sanity/Contentful) configured with clinician & service schema; content migrated and validated.
- Booking integration module:
  - Primary: PMS API live integration (availability, booking create, cancel, modify).
  - Fallback: provisional reservation via queue with reconciliation; admin notification flow for conflicts.
- Authentication scaffold: token-based sessions + OTP for critical flows; patient portal skeleton (authenticated pages gated).
- Image and video pipeline: CDN, responsive srcsets, AVIF/WebP generation, LQIP placeholders.
- Analytics event wiring per spec.
Acceptance criteria:
- End-to-end booking from site to PMS and back confirmed with test bookings in staging (or simulated if PMS unavailable).
- Booking flows handle race conditions gracefully (user receives conflict message and alternative options).
- CMS content mapping completeness ≥98% with visual validation on staging.
- Lighthouse mobile score ≥80 (goal 90+ post-optimisation).
Key tasks:
- Implement server-side caching and incremental static regeneration for top pages.
- Implement form validation, autosave, file upload with size/format checks, virus scanning integration if available.
Owners: Engineering Lead, Backend Engineer, Frontend Engineer, DevOps.

Security & privacy checks during build:
- TLS everywhere, input sanitisation, secure storage of PHI, data encryption at rest (as required by PDPA), role-based access controls for CMS.

---

### Phase 4 — QA, Accessibility Remediation & Security (Week 10–11)
Goal: Ensure functional correctness, accessibility compliance, and security posture.
Deliverables:
- Automated test suites (unit + integration) for critical components, and E2E booking/registration tests.
- Accessibility audit (automated + manual) with remediation report.
- Security review: OWASP checklist, penetration testing summary, HIPAA vs PDPA alignment documentation.
- Performance optimisation pass (image compression, critical CSS, caching headers).
Acceptance criteria:
- No critical accessibility or security findings outstanding.
- All critical E2E tests pass in CI; booking/reg flows tested across desktop and mobile emulators.
- Performance regressions resolved; Lighthouse mobile target met or a documented plan to reach it.
Key tasks:
- Manual keyboard and screen reader testing for 10 user scenarios.
- Run penetration test and resolve high risk issues.
Owners: QA Lead, Accessibility Lead, Security Lead.

---

### Phase 5 — Beta Launch, Monitoring & Staff Training (Week 12)
Goal: Release to a controlled audience, monitor, and prepare staff to operate new flows.
Deliverables:
- Staged rollout plan (percentage rollout or clinic-by-clinic), feature flags for booking new UI.
- Monitoring dashboards (conversion funnel, errors, performance, uptime) and alerting.
- Training materials and runbooks for clinic staff and receptionists: booking reconciliation, admin overrides, how to handle conflicts.
- Post-launch checklist and rollback plan.
Acceptance criteria:
- Beta users can book and register with no major issues; support SLAs established for first 72 hours.
- Staff training completed and a support channel established.
Key tasks:
- Enable monitoring: Sentry (errors), GA4 funnels, custom dashboards.
- Communicate change to patients via site banner and clinic channels.
Owners: Product Ops, Support Lead, Analytics Owner.

---

### Phase 6 — Iteration, A/B Testing & Launch Optimisation (Weeks 13–18)
Goal: Improve conversion, usability, and scalability through measured experiments and iterative fixes.
Deliverables:
- A/B tests (prioritised) with hypothesis, sample size, and target metrics:
  - Hero CTA: single primary CTA vs dual CTA.
  - Booking modal: compact modal vs full-page flow.
  - Mobile sticky bar vs header CTAs.
- Ongoing accessibility regressions monitoring and monthly audits.
- Roadmap items for advanced features (voice search MVP, AI symptom checker, telemedicine expansion).
Acceptance criteria:
- Statistically significant gains for prioritized experiments or clear learnings to guide next steps.
- KPI improvements aligned with goals (see KPIs below).
Key tasks:
- Run experiments for minimum viable periods with traffic allocation, measure and decide.
Owners: Product Lead, Growth/Analytics, UX Research.

---

### KPIs and success metrics (operational)
- Booking conversion rate (CTA click → confirmed booking) +30% vs baseline.  
- Booking abandonment reduction −40% vs baseline.  
- New patient registration completion +25% vs baseline.  
- Time-to-book for returning users <90s.  
- WCAG AA compliance on core user flows (automated + manual checks).  
- Lighthouse performance mobile ≥90 (goal) or operational plan documented.  
- Mobile conversion uplift +35% vs baseline.

---

### Component-level acceptance checklist (sample)
Booking modal
- Opens within 250ms of CTA click on mobile/desktop.  
- Focus lands on the first interactive control; focus returns to CTA after close.  
- Keyboard navigation completes full booking without mouse.  
- ARIA attributes: role=dialog, aria-modal, aria-labelledby, aria-describedby.  
- Edge case: concurrent booking conflict handled with user-facing conflict resolution message and alternate slot suggestions.

New Patient Registration form
- Autosave every 15–30s with explicit success toast.  
- Resume token valid for configurable TTL (e.g., 7 days).  
- File upload: max 10MB, allowed types jpg/png/pdf, client-side validation + server-side virus scan.  
- Explicit consent checkbox with link to PDPA policy and retention period.

Clinician profile
- Must include: photo, credentials, languages, consultation types, telemedicine availability, next available slots.  
- “Book with Dr” opens booking modal with clinician preselected.

---

### Data & migration plan (practical)
- Stage 1: Export raw clinician and service data from current site/CMS.  
- Stage 2: Normalize to canonical schema in CSV/JSON; flag missing fields for operations to complete.  
- Stage 3: Import into headless CMS with validation scripts; automate backups.  
- Stage 4: Manual visual verification on staging; QA sign-off before publishing.

---

### Governance, roles & RACI (recommended)
- Product Lead: overall owner (R)  
- Design Lead: UI/UX, design system (A)  
- Engineering Lead: build & integration (A)  
- Accessibility Lead: accessibility tests & sign-off (R)  
- Security/Legal: compliance & privacy (C)  
- Clinic Ops: clinician data owner, PMS liaison (C)  
- QA Lead: validation & release gating (R)  
- Analytics Owner: instrumentation & dashboards (R)

(R = Responsible, A = Accountable, C = Consulted, I = Informed)

---

### Risks, mitigations and contingencies
- PMS integration is delayed → use provisional booking queue and communication plan; parallel work to integrate once APIs available.  
- Incomplete clinician data → enforce mandatory fields and schedule clinician data collection sprint with deadlines.  
- Accessibility regressions post-launch → schedule weekly remediation sprints and maintain QA gates in CI.  
- Staff adoption friction → dedicated training, shadowing sessions, and one-week hypercare support.

---

### Immediate next deliverable options (pick one)
1. Technical Discovery Pack: full PMS questionnaire, PDPA checklist, clinician schema template.  
2. Component Accessibility Contract: detailed component-level tests and ARIA attributes for each interactive element.  
3. Analytics Event Spec & Dashboard Mock: naming conventions, event payloads, funnel definitions, sample dashboards.  
4. Annotated Wireframes: low-fidelity + high-fidelity annotated wireframes for Home, Booking modal, Service Detail (desktop & mobile) ready for Figma handoff.

I will produce the chosen deliverable immediately with complete acceptance criteria, test cases, and an implementation-ready checklist. Select which one you want first and I’ll generate it.
