# UNAI Recruitment Management System

An applicant tracking workflow for member recruitment. Applicants read the recruitment information and submit an application online; administrators review applicants and move them through a fixed status workflow.

> Technical Case 03 — Recruitment Management System (Frontend & Backend Developer selection)

- **Live demo:** `https://beamish-bubblegum-761c36.netlify.app/public/recruitment.html`
- **Repository:** `(https://zahrajannah945-dotcom.github.io/recruitment-/public/recruitment.html)`

---

## 1. Project Explanation

### Problem
The organization needs a simple way to collect recruitment applications and track each applicant from first submission to final decision, instead of handling forms and spreadsheets manually.

### Users
| User | What they can do |
|---|---|
| Applicant (public) | Read recruitment info, submit the application form |
| Recruitment admin | Log in, view/search/filter applicants, read details, update status, export CSV |

### Status Workflow
```
Pending ──► Interview ──► Accepted
   │            │
   └──► Rejected ◄┘
```

| From | Allowed next status |
|---|---|
| pending | interview, rejected |
| interview | accepted, rejected |
| accepted | none (final) |
| rejected | none (final) |

The rules are enforced **on the server** (`PATCH /api/applicants/[id]`). A request that tries to skip a step (e.g. pending → accepted) is rejected with HTTP 400, so the workflow cannot be bypassed from the browser.

### Required Data
Name, email, contact number, division preference, motivation, application status — all stored in the `applicants` table.

### Features
**Implemented**
- Recruitment information page (divisions, timeline, call to action)
- Application form with client- and server-side validation
- Confirmation page after submission
- Admin login (JWT in an httpOnly cookie, bcrypt-hashed passwords)
- Route protection for admin pages via middleware, and per-request auth checks on admin APIs
- Admin dashboard: applicant table, status filter, name/email search, summary statistics
- Applicant detail page with status-update buttons that only show valid next steps
- CSV export of all applicants

**Not implemented / limitations** (see also section 6)
- Real CV file upload — the database has a `cv_url` column but the form does not upload files yet
- Applicants cannot check their own status
- Division filter exists in the API but not in the dashboard UI
- No duplicate-application check per email, no pagination

---

## 2. Tech Stack and Decisions

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js 14 (App Router) | Pages and API routes in one repo; deploys to Vercel with no extra config |
| Database | MySQL | Required stack for this submission |
| ORM | Prisma | Typed schema, migrations, and safe parameterized queries |
| Auth | JWT + bcryptjs, httpOnly cookie | Simple, no third-party service; cookie is not readable by client JS |
| Styling | Tailwind CSS | Fast, consistent, responsive UI |
| Hosting | Vercel + hosted MySQL | Vercel has no database, so MySQL is hosted externally |

Key decisions worth explaining:
1. **Status transitions validated in the backend**, not trusted from the UI.
2. **Prisma client singleton** (`src/lib/prisma.js`) to avoid exhausting database connections during hot reload and on serverless.
3. **Server-side validation duplicated from the client** because client validation is only a convenience.
4. **Middleware uses `jose`** (Edge-compatible) while API routes use `jsonwebtoken`; both verify the same signed token.

---

## 3. Project Structure

```
prisma/
  schema.prisma          Database schema (Applicant, Admin)
  seed.js                Creates the first admin account
src/
  middleware.js          Protects /admin/dashboard and /admin/applicants
  lib/
    prisma.js            Prisma client singleton
    auth.js              JWT sign/verify + cookie helpers
  app/
    page.js              Recruitment information
    apply/page.js        Application form
    apply/success/       Post-submit confirmation
    admin/login/         Admin login
    admin/dashboard/     Applicant list, filters, statistics
    admin/applicants/[id]/  Applicant detail + status update
    api/
      applicants/        POST (public) create, GET (admin) list
      applicants/[id]/   GET, PATCH status, DELETE (admin)
      auth/login, logout Session handling
      stats/             Counts per status and division
      export/            CSV download
```

---

## 4. Setup Instructions

### Prerequisites
- Node.js 18 or newer
- A MySQL database (local, or hosted such as Railway, Aiven, or PlanetScale)

### Steps

1. **Install dependencies**
   ```bash
   npm install
   ```

2. **Create the environment file**
   ```bash
   cp .env.example .env
   ```
   Fill in:
   ```
   DATABASE_URL="mysql://USER:PASSWORD@HOST:3306/recruitment_db"
   JWT_SECRET="a-long-random-string"
   SEED_ADMIN_EMAIL="admin@unai.org"
   SEED_ADMIN_PASSWORD="choose-a-strong-password"
   ```

3. **Create the database tables**
   ```bash
   npx prisma migrate dev --name init
   ```

4. **Generate the Prisma client** (also runs automatically on install)
   ```bash
   npx prisma generate
   ```

5. **Seed the first admin account**
   ```bash
   npm run seed
   ```

6. **Start the app**
   ```bash
   npm run dev
   ```
   - Public site: http://localhost:3000
   - Apply: http://localhost:3000/apply
   - Admin: http://localhost:3000/admin/login

### Quick test checklist
1. Submit an application at `/apply`.
2. Log in at `/admin/login`.
3. Open the applicant from the dashboard.
4. Move it Pending → Interview → Accepted.
5. Try to change a final-status applicant — no buttons appear, and the API returns 400 if called directly.
6. Click **Export CSV**.

---

## 5. Deployment (Vercel)

1. Push the project to GitHub.
2. Import the repository in Vercel.
3. Add environment variables in **Project Settings → Environment Variables**:
   - `DATABASE_URL` (your hosted MySQL)
   - `JWT_SECRET` (a long random string — never leave it as the default)
4. Deploy. `postinstall` and `build` both run `prisma generate`.
5. From your machine, apply the schema to the production database:
   ```bash
   DATABASE_URL="<production url>" npx prisma migrate deploy
   DATABASE_URL="<production url>" npm run seed
   ```
6. Open the live URL and test the full flow in production.

Serverless functions are short-lived, so use a MySQL host that handles many short connections well (connection pooling).

---

## 6. Known Limitations and Next Steps

- Add real CV upload (Vercel Blob or Cloudinary) and store the returned URL in `cv_url`.
- Add a public "check my status" page (requires a lookup endpoint by email).
- Add division filter, sorting, and pagination to the dashboard.
- Prevent duplicate applications from the same email.
- Add rate limiting on login and the public application endpoint.
- Add automated tests for the status-transition rules.

---

## 7. AI Usage Report

**Tools used:** Claude (Anthropic)  
| Area | AI contribution |
|---|---|---|
| Project scaffolding and folder structure | Generated initial structure | 
| Prisma schema | Drafted models and enum |
| API routes and status-transition logic | Drafted | 
| UI pages and Tailwind styling | Drafted |
| README | Drafted | Reviewed and filled in project details |

**Important prompts** (replace with your real ones):

## Halaman

PUBLIC:
1. index.html — Hero section ("Join Us, Create Impact"), About Recruitment, Recruitment Timeline, Available Divisions, Requirements, FAQ (pakai <details>/<summary> HTML native, tanpa JS), CTA, tombol "Apply Now →" ke apply.html
2. apply.html — Form dengan field: Full Name, Email, Contact Number, Division (select), Motivation (textarea, validasi minimal 50 karakter), Upload CV (input type=file). Gunakan atribut Netlify Forms: `<form name="application" method="POST" data-netlify="true">`. Setelah submit sukses, redirect ke success.html.
3. success.html — Konfirmasi visual, tampilkan status "Pending", tombol ke status.html
4. status.html — Form cek status berdasarkan email, hasil ditampilkan sebagai progress steps visual (Submitted → Under Review → Interview → Final Decision) menggunakan HTML list + CSS, bukan library eksternal

ADMIN (butuh koneksi ke API eksternal via js/api.js):
5. admin/login.html — Form login sederhana dan clean
6. admin/dashboard.html — Kartu statistik (total, pending, interview, accepted, rejected)
7. admin/applicants.html — Tabel pelamar dengan search dan filter status (pakai <select> dan <input>, logic filter di JS vanilla)
8. admin/applicant-detail.html — Detail pelamar + tombol ubah status (Pending→Interview→Accepted/Rejected), tombol yang muncul harus sesuai aturan transisi yang valid dari status saat ini

## Validasi Form (Client-side, di js/validate.js)
Tampilkan pesan error tepat di bawah field, bukan alert(). Contoh:
- "Email is required"
- "Please enter a valid email address"  
- "Motivation must be at least 50 characters"
Gunakan elemen <span class="error-message"> di bawah setiap input, toggle class "visible" lewat JS.

## State Handling di Halaman Admin
Setiap halaman yang memanggil API harus menangani 4 kondisi secara eksplisit:
- Loading: tampilkan elemen dengan class "loading-spinner" sebelum data datang
- Success: render data ke dalam DOM
- Error: tampilkan div dengan class "error-banner", pesan jelas
- Empty: tampilkan div dengan class "empty-state" kalau hasil filter kosong

## CSS Requirements
- Gunakan CSS Custom Properties (:root { --brand-color: ...; }) untuk design token: warna brand, warna status (pending/interview/accepted/rejected), spacing, border-radius — supaya konsisten di semua file HTML
- Mobile-first, gunakan @media (min-width: ...) bukan max-width
- Sidebar admin: flex/grid di desktop, berubah jadi hamburger menu (toggle class lewat JS) di bawah 768px
- Tabel pelamar: bungkus dengan <div style="overflow-x:auto"> agar scroll horizontal di mobile, tidak merusak layout
- Tambahkan transisi antar halaman: gunakan CSS @view-transition (navigation: auto) sebagai enhancement progresif, dengan fallback tanpa animasi di browser yang belum mendukung. Hormati prefers-reduced-motion.
- Status badge: warna berbeda per status (kuning pending, biru interview, hijau accepted, merah rejected), buat sebagai class CSS reusable (.status-badge.pending, dst.)

## netlify.toml
Buatkan juga file netlify.toml dengan konfigurasi redirect dasar (opsional pretty URLs) dan header keamanan dasar.

## Output yang saya butuhkan
1. Semua file HTML lengkap (bukan potongan kode)
2. File CSS lengkap dengan design token di :root
3. File JS vanilla (api.js, validate.js, auth-guard.js, page-transition.js)
4. netlify.toml
5. README.md berisi: cara buka di lokal, cara isi BASE_URL API, cara deploy ke Netlify (drag-drop atau lewat GitHub), dan penjelasan batasan (fitur mana yang butuh backend eksternal)
6. Jelaskan di akhir keputusan teknis yang kamu ambil, terutama kenapa fitur admin butuh API eksternal padahal websitenya HTML/CSS statis

**Verification:** I ran the app locally against MySQL, tested each status transition (valid and invalid), and checked that admin routes reject unauthenticated requests. 
