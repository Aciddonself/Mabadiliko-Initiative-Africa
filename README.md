# Mabadiliko Initiative Africa (MIA) — website & web app

**Skilled Hands. Digital Minds. Shared Futures.**

A production-quality, mobile-first, multilingual (EN / Kiswahili / Français / العربية with RTL) website and
web app for a Ugandan NGO that gives underprivileged Ugandans and refugees practical, job-ready skills:
international language education, tailoring and garment-making, and digital skills.

Built with **Next.js (App Router) + TypeScript + Tailwind CSS v4 + PostgreSQL via Drizzle ORM**.

---

## 1. Proposed sitemap (as built)

| Route | Purpose |
| --- | --- |
| `/` | Home: hero + Enroll/Donate/Partner CTAs, live impact counters from the DB, program highlights, SDG strip, testimonials, partners, transparency, latest news |
| `/about` | Mission, vision, values, story & timeline, team/Board, legal registration, safeguarding summary |
| `/programs` | All four programs with duration, schedule, mode, fees, outcomes, SDGs, upcoming cohorts |
| `/programs/languages` | International Language Education (8 languages, CEFR levels, exam prep) |
| `/programs/tailoring` | Tailoring & garment-making + start-up kit & mentorship |
| `/programs/computer-office` | Computer literacy & Microsoft Office + internet safety + mobile-money literacy |
| `/programs/ai-basics` | AI basics & practical prompt-writing (incl. ethics, privacy, verification) |
| `/certification` | Language Certification Hub: pathways per language, CEFR level guide, exam dates & centres, registration FAQ, editable **Accreditation & Partners** register |
| `/enroll` | Enrollment / application form — **works with a phone number only, no email required** |
| `/portal` | Student portal MVP: sign in with phone/email + access code, my courses, schedule, materials, progress tracker, PDF certificate download, offline/PWA guidance |
| `/donate` | One-off & monthly giving, suggested amounts with impact statements, card (Stripe) / PayPal / MTN MoMo / Airtel Money placeholders, fund transparency |
| `/get-involved` | Volunteer, trainer/teacher application, partnership inquiry, Sponsor a Learner |
| `/impact` | SDG dashboard (SDG 1, 4, 5, 8, 10, 16, 17), how each program contributes, methodology, annual reports, stories of change, gallery |
| `/news`, `/news/[slug]` | CMS-driven news / blog (DB-backed, editable in admin) |
| `/contact` | Contact form, WhatsApp click-to-chat, map embed, direct lines, social links, complaints |
| `/privacy` | Privacy & data protection (Uganda Data Protection and Privacy Act, 2019), refugee-data safeguards |
| `/safeguarding` | Safeguarding policy, code of conduct, reporting route, investigation steps |
| `/admin`, `/admin/login` | Role-based admin dashboard (Admin / Staff / Trainer) |
| `/api/health` | Health probe |
| `/api/certificate?serial=…` | Generates the MIA Certificate of Completion as a real PDF (no dependency) |
| `/api/admin/export?entity=…` | CSV/Excel export (applications, students, cohorts, donations, volunteers, posts) |
| `/sitemap.xml`, `/robots.txt`, `/manifest.webmanifest`, `/sw.js` | SEO + PWA/offline shell |

## 2. Where people sign in, and where each form goes

### Staff, trainer and administrator sign-in

**URL: `/admin/login`** — email + password only.

- Deliberately **not** linked from the main navigation, so it is not discoverable by search engines. Reach it from
  the footer ("Staff sign in"), the mobile menu ("Admin"), or by typing the URL.
- `/admin` is server-guarded: with no valid session it returns a **307** to `/admin/login`. The dashboard is never
  rendered for an anonymous visitor, and `/api/admin/export` returns **401** without a session.
- On success a random token is written to the `sessions` table and sent as an **httpOnly** cookie named
  `mia_session` (`SameSite=Lax`, `Secure` in production, 14-day expiry). Sign out is the "Sign out" button on the
  dashboard, which deletes the row and the cookie, then redirects to `/admin/login`.
- Passwords are scrypt hashes with a per-user random salt (`hashPassword`/`verifyPassword` in `src/lib/auth.ts`)
  and compared with `timingSafeEqual`.
- A user whose role is `student` is rejected here even with a correct password — students cannot reach `/admin`.
- Section visibility comes from `ROLE_ACCESS` / `canAccess` in `src/lib/auth.ts`.

### Student sign-in

**URL: `/portal`** — phone number *or* email, plus a per-learner access code (format `MIA-0000`). No password.

- The learner signs in with whatever they used to apply, plus the code their trainer gives them
  (demo learner: `+256750000004` / `MIA-2481`).
- On success the learner's numeric `students.id` is stored in an httpOnly cookie named `mia_student`
  (8-hour lifetime, no `Secure` flag difference from staff sessions).
- An access code is issued automatically when an admin uses **"Admit → create student"** on the Applications tab,
  and the code is recorded in the application's admin notes.
- Signing in does not send anything anywhere — the code is checked against the `students.access_code` column.
  If a learner cannot sign in, staff re-issue a code; there is no self-service reset.

### Where each form is submitted

There is **no public REST endpoint for form submissions**. All four public forms call a `"use server"` action
directly from `src/app/actions.ts` using `useActionState`, so the browser posts back to the page's own URL and
Next.js runs the function on the server. Each action calls `ensureSeeded()` and then `notify()`.

| Form | Page | Component | Server action | Table written | Shows up in admin |
| --- | --- | --- | --- | --- | --- |
| Enrolment / application | `/enroll`, `/programs/[slug]` | `EnrollmentForm` | `submitApplication` | `applications` | **Applications** tab (new reference `MIA-YYYY-NNNN`) |
| Donation pledge | `/donate` | `DonationForm` | `submitDonation` | `donations` | **Donations** tab (reference `MIA-D-100NN`) |
| Get involved (volunteer / trainer / partner / sponsor) | `/get-involved` | `GetInvolvedForm` | `submitGetInvolved` | `volunteers` | **Volunteers, trainers & partners** tab |
| Contact / complaints | `/contact` | `ContactForm` | `submitContact` | `contact_messages` | **Overview** tab ("Latest contact messages") |

Validation is server-side and returns per-field errors; a submission never half-writes because it is a single
`db.insert` after validation. `revalidatePath("/admin")` makes new rows appear in the dashboard immediately.

Enrolment and contact forms also store the visitor's `locale` so replies can be sent in their own language.
Enrolment accepts a **phone number only** — email is optional, and email is the only optional-on-every-form field.

### Contact numbers

The published phone and WhatsApp numbers live in one place — `site` in `src/lib/site.ts`:

| Field | Value |
| --- | --- |
| `site.phone` | `+256 753 149 567` |
| `site.whatsapp` | `256753149567` (digits only, used by `whatsappLink()`) |
| `site.whatsappNumbers` | `["+256753149567", "+256736370117"]` |

`whatsappLink(message, number?)` builds a `https://wa.me/…` deep link, defaulting to `site.whatsapp`. Every click-to-chat
button iterates `site.whatsappNumbers`, so both lines are published on `/contact`, in the footer, and in the floating
WhatsApp button. Change the numbers there and the whole site updates.

### Other endpoints

| Endpoint | Purpose |
| --- | --- |
| `POST /api/mia-test/*` | **Does not exist** — that was a temporary test harness and has been deleted. |
| `GET /api/health` | Health probe, `{"ok":true}` |
| `GET /api/certificate?serial=…` | Renders the MIA Certificate of Completion PDF |
| `GET /api/admin/export?entity=…` | CSV export, requires a staff session (401 otherwise) |

## 3. Database schema (Drizzle, `src/db/schema.ts`)

| Table | Notes |
| --- | --- |
| `users` | admin / trainer / staff / student, scrypt password hash, optional `access_code` |
| `sessions` | server-side session tokens for staff sign-in (httpOnly cookie) |
| `programs` | program catalogue (slug, category, duration, fees, SDGs) |
| `cohorts` | scheduled classes: dates, schedule, mode, location, capacity, enrolled, status |
| `applications` | enrollment submissions. `email` nullable, refugee/asylum status + accessibility **optional**, consent + consent name, review status, admin notes |
| `students` | admitted learners, access code for the portal |
| `enrollments` | student ↔ cohort, progress %, attendance %, grade, certificate serial, completion date |
| `materials` | downloadable class materials with file size (low-bandwidth UX) |
| `donations` | amount, frequency (once/monthly), method (stripe/paypal/mtn-momo/airtel-money), designation, status, anonymity |
| `volunteers` | volunteer / trainer / partner / sponsor submissions incl. sponsorship amount |
| `contact_messages` | contact form, incl. confidential safeguarding subject |
| `posts` | CMS news/blog (slug, excerpt, body, category, author, status, publishedAt) |
| `testimonials` | consented quotes (`consent` flag) |
| `impact_counters` | admin-editable live counters (key, label, value, detail) |
| `reports` | annual reports & policy PDFs |
| `exam_sessions` | official exam sittings: language, level, awarding body, centre, city, dates, fee |
| `partners` | editable Accreditation & Partners register |
| `audit_log` | who did what in the admin dashboard |

**Data protection by design:** minimal collection, optional refugee/accessibility fields, no immigration document
numbers, no biometrics, nullable email everywhere, `consent` and `consent_name` stored with each application.

## 4. Setup

```bash
# 1. install
npm install

# 2. environment
cp .env.example .env   # or create .env
# DATABASE_URL=postgresql://user:pass@host:5432/db

# 3. push the schema (no migration files needed)
npx drizzle-kit push

# 4. run
npm run dev        # http://localhost:3000
npm run build && npm run start
```

Seed content (programs, cohorts, materials, users, applications, students, donations, volunteers, posts,
testimonials, counters, reports, exam sessions, partners) is inserted automatically on first request by
`ensureSeeded()` in `src/db/seed.ts`. It is idempotent: it skips if `programs` already has rows.

### Environment variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | yes | PostgreSQL connection string |
| `NOTIFY_WEBHOOK_URL` | no | Webhook for transactional notifications (Zapier/Make/Resend) — see below |
| `RESEND_API_KEY` / `SENDGRID_API_KEY` | no | Live transactional email |
| `STRIPE_SECRET_KEY`, `PAYPAL_CLIENT_SECRET`, `MTN_MOMO_SUBSCRIPTION_KEY`, `AIRTEL_CLIENT_SECRET` | no | Live payments |

### Demo accounts (seeded)

| Role | Email | Password |
| --- | --- | --- |
| Admin | `admin@mabadiliko-africa.org` | `MIA-admin-2026` |
| Staff (intake) | `staff@mabadiliko-africa.org` | `MIA-staff-2026` |
| Trainer | `trainer@mabadiliko-africa.org` | `MIA-trainer-2026` |
| Learner (portal) | phone `+256750000004` | access code `MIA-2481` |

Role-based access is enforced in `src/lib/auth.ts` (`ROLE_ACCESS`) and in the dashboard itself.
Student records are NOT readable through the admin login path — learners sign in at `/portal`.

## 5. Transactional email / notifications

`notify()` in `src/app/actions.ts` fires on every application, donation, get-involved and contact submission.
Today it logs to the server console and POSTs to `NOTIFY_WEBHOOK_URL` if set. To send real email:

```ts
// inside notify(), replace the console.info line with:
await fetch("https://api.resend.com/emails", {
  method: "POST",
  headers: {
    authorization: `Bearer ${process.env.RESEND_API_KEY}`,
    "content-type": "application/json",
  },
  body: JSON.stringify({
    from: "MIA <intake@mabadiliko-africa.org>",
    to: ["intake@mabadiliko-africa.org"],
    subject,
    text: JSON.stringify(body, null, 2),
  }),
});
```

For learners without email, keep the WhatsApp route: send the application reference by WhatsApp/SMS via a
provider such as Twilio, Africa's Talking or 360dialog.

## 6. Deploying

**Vercel (recommended)**

1. Push the repo to GitHub and import it in Vercel.
2. Add `DATABASE_URL` (Neon / Supabase / RDS — use a pooled connection string) and any secrets.
3. Build command `npm run build`, output is detected automatically.
4. Run `npx drizzle-kit push` once against the production database (locally, with the prod URL).
5. Add your domain and update `site.domain` in `src/lib/site.ts` so metadata/OG/sitemap use the live URL.

**Netlify** — install `@netlify/plugin-nextjs`, build command `npm run build`, publish directory `.next`.

**Docker / VPS** — `npm run build && npm run start` behind a reverse proxy with HTTPS (required for the
secure session cookies and the service worker).

## 7. Accessibility, performance and context notes

- WCAG 2.1 AA target: visible 3px focus rings, 44px+ tap targets, semantic landmarks, `aria-current`,
  `role="progressbar"`, native `<details>` disclosures (zero JS), meaningful `alt` text, skip link.
- Low bandwidth: system font stack (no webfont download), no JS framework beyond React on the pages that need
  it, `loading="lazy"` + `decoding="async"` on every non-hero image, compressed images served at ~900px,
  file sizes shown before download, and a ~1 KB service worker caching the last-viewed pages.
- Arabic: `dir="rtl"` is applied to `<html>` by the i18n provider, and layout uses logical properties
  (`ms-`/`me-`, `start-`/`end-`) so the whole site mirrors.
- Tone: no pity imagery. Placeholder photography is accompanied by written image briefs that staff can hand to
  a photographer.

## 8. Recommended next steps before launch

1. **Legal review** of `/privacy`, `/safeguarding`, the consent wording and the complaints procedure by a
   Uganda-qualified adviser; register with the Personal Data Protection Office.
2. **Real names, photos and biographies** on `/about` (team & Board) and real NGO registration/permit numbers.
3. **Payment gateway keys** — Stripe + PayPal for international givers, MTN MoMo Collections API and Airtel
   Money API for Uganda; add webhook handlers and reconcile `donations.status`.
4. **Exam-centre agreements** — replace the placeholder Accreditation & Partners register with signed
   agreements, and confirm the live exam calendar before publishing dates.
5. **Real photography** with written consent forms (briefs are supplied on `/` and `/impact`).
6. **Auth hardening** — replace the seeded demo accounts, enforce 2FA for admins, add rate limiting on the
   login endpoints, and move sessions to short-lived rotating tokens.
7. **Monitoring & backups** — error tracking, uptime probe on `/api/health`, nightly encrypted DB backups with
   access logging (refugee data requires demonstrable care).
8. **Content translation pass** — extend `src/lib/i18n.ts` beyond nav/UI/forms into program pages, and have
   native Kiswahili, French and Arabic speakers review the copy.

---

“Mabadiliko” is Swahili for **transformation**.
