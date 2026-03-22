# BookSolid — Technical Delivery Specification

## Product Overview
BookSolid is a managed AI scheduling and no-show prevention service for appointment-based small businesses. Jarred delivers the outcome (no-shows eliminated, calendar full); the client provides access to their calendar and phone/email.

---

## Architecture

### Delivery Model
- **Type:** Managed service (not SaaS self-serve at launch)
- **Deployment:** OpenClaw-powered automation layer connecting client's calendar, Twilio (SMS), SendGrid (email), Stripe (deposits)
- **Client interface:** Dashboard web app (read-only for client) + Slack/SMS alerts for anomalies

### Core Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Calendar Integration | Google Calendar API, Microsoft Graph API, CalDAV | Pull/push appointment data |
| SMS Reminders | Twilio Programmable SMS | Outbound reminders + 2-way confirm |
| Email Reminders | SendGrid | HTML email reminders + confirm links |
| Voice Reminders | Twilio Voice + TTS | Optional call-based reminders |
| Booking Page | Custom hosted URL (booksolid.ai/[client-slug]) | Client's branded online booking |
| Waitlist Engine | Custom (OpenClaw) | Auto-notify + first-responder booking |
| Deposit Collection | Stripe | Tokenized card capture at booking |
| Dashboard | Simple web UI | Client-facing metrics |
| Notification Layer | Twilio + email | Ops alerts to client and to ops team |

---

## Reminder Sequence Logic

### Standard Sequence (configurable)
```
Booking confirmed → Immediate SMS + Email confirmation
T-48 hours → SMS + Email reminder (confirm or cancel)
T-24 hours → If not confirmed: SMS + Email #2 (more urgent tone)
T-2 hours → Final SMS reminder
T-1 hour → If no confirmation AND high-value: Voice call
```

### Confirmation Logic
- Client taps "Confirm" link or replies "YES" → marked confirmed, no further messages
- Client replies "CANCEL" → triggers waitlist notification + reschedule offer
- Client ignores all → flagged in dashboard, sent to no-show management flow post-appointment

### Waitlist Auto-Fill
```
On cancellation:
1. Mark slot as open
2. Query waitlist by priority (first-in-first-out + time preference match)
3. Send SMS to top 3: "A slot opened [DATE TIME] — reply YES to claim it"
4. First YES wins slot → booking confirmed → remainder notified "slot filled"
5. If no waitlist match in 30 min → alert client ops (Slack/SMS)
```

### No-Show Management
```
If appointment time passes with no confirmation + no cancel:
1. Log as no-show event
2. If deposit collected → release/charge per policy
3. If no deposit: send "We missed you" text with reschedule link
4. Flag account as no-show risk (score +1)
5. After 2 no-shows: auto-require deposit for future bookings
6. Alert business owner of repeat pattern
```

---

## Integrations Supported at Launch

| Platform | Integration Method | Priority |
|----------|-------------------|---------|
| Google Calendar | OAuth + Calendar API | Day 1 |
| Microsoft Outlook | Microsoft Graph OAuth | Day 1 |
| Calendly | Webhook + API | Day 1 |
| Acuity Scheduling | API + Webhooks | Day 1 |
| Jane App | API | Week 2 |
| Mindbody | API | Week 2 |
| Square Appointments | API | Week 3 |
| Vagaro | API | Week 4 |
| EHR systems (generic) | HL7 FHIR / custom | Phase 2 |

---

## Data Flow

```
Client calendar → BookSolid sync layer (every 5 min)
    ↓
Appointment queue (SQLite → Postgres at scale)
    ↓
Reminder scheduler (cron, checks every 15 min)
    ↓
Twilio SMS/Voice + SendGrid Email
    ↓
Confirmation webhook → update appointment status
    ↓
Waitlist engine (if cancel) OR dashboard update (if confirm)
    ↓
Nightly report: no-show rate, slots filled, revenue saved
```

---

## Pricing Tiers — Technical Limits

| Plan | Appts/Mo | Staff Calendars | SMS Volume | Voice Calls | Monthly Cost |
|------|---------|----------------|-----------|------------|-------------|
| Starter | 150 | 1 | 600 | No | $299 |
| Professional | Unlimited | 2 | Unlimited | Yes (2hr slot) | $499 |
| Business | Unlimited | Unlimited | Unlimited | Yes | $899 |

### Unit Economics (Professional Plan)
- Twilio SMS: ~$0.0079/msg × 3 msgs × 200 appts = ~$4.74/mo
- SendGrid: ~$0.0010/email × 3 emails × 200 appts = ~$0.60/mo
- Hosting: ~$15/mo (shared infra)
- Labor (onboarding + monitoring): ~$30/mo equivalent
- **Total COGS: ~$50/client/month**
- **Gross margin on Professional: ~90%**

---

## Onboarding Sequence

### Day 0 (Setup call — 30 min)
1. Connect calendar (OAuth)
2. Set reminder timing preferences
3. Set business hours / blackout windows
4. Configure no-show policy (deposit Y/N, fee amount, threshold)
5. Connect Stripe for deposit collection (if enabled)
6. Test: send test reminder to owner's phone
7. Configure waitlist settings

### Day 1–3
- System runs in shadow mode (no outbound) — owner reviews proposed reminders
- Owner approves → go live

### Day 7
- First week review: show stats, no-shows prevented, revenue protected
- Fine-tune reminder wording if needed

### Day 30
- Monthly ROI review: show exact $ recovered, no-show rate change, slot fill rate

---

## SLA Commitments

| Metric | Target | Monitoring |
|--------|--------|-----------|
| Reminder delivery (SMS) | 99.5% within 5 min of scheduled time | Twilio status webhook |
| Reminder delivery (email) | 99.0% within 10 min | SendGrid events |
| Waitlist notification | < 5 min after cancellation | Internal log |
| Dashboard uptime | 99.9% | UptimeRobot |
| Setup completion | ≤ 48 hours after payment | Internal checklist |
| Support response | < 4 business hours | Slack/email |

---

## Security + Privacy

- All calendar OAuth tokens stored encrypted at rest (AES-256)
- Client data isolated per-tenant (separate DB namespaces)
- Client contact data (phone/email) never sold or shared
- CCPA compliant (California businesses)
- Opt-out mechanism on all reminder messages ("Reply STOP to unsubscribe")
- Stripe handles all payment data (PCI-DSS compliant, no card data on BookSolid servers)

---

## Build Phases

| Phase | Timeline | Deliverables |
|-------|---------|-------------|
| MVP | Weeks 1–2 | Google Calendar + Twilio SMS reminders + basic dashboard |
| Phase 2 | Weeks 3–4 | Waitlist engine + deposit via Stripe + email reminders |
| Phase 3 | Month 2 | Outlook/Calendly/Acuity integrations + voice call option |
| Phase 4 | Month 3 | Mindbody/Jane + multi-location + white-label portal |
| Scale | Month 4+ | Self-serve onboarding + automated provisioning |

---

## First Client Delivery Checklist

- [ ] Calendar connected and syncing
- [ ] Reminder sequence tested end-to-end (test appointment created → reminder sent → confirmation captured)
- [ ] Waitlist flow tested
- [ ] Deposit flow tested (if enabled)
- [ ] Dashboard showing live data
- [ ] Client trained on dashboard (15-min walk-through)
- [ ] Support channel established (Slack or email)
- [ ] SLA document sent
- [ ] First invoice processed
