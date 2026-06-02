# Clostridium Tracker — Clinical Patient Management App

> A web application for internal medicine teams to track *Clostridioides difficile* (C. difficile) infection cases across initial diagnosis and follow-up visits — structured around clinical severity scoring, recurrence-risk assessment, treatment recording, and antibiotic-stewardship review.

![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)
![Vite](https://img.shields.io/badge/Vite-7-646CFF?logo=vite&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-backend-3FCF8E?logo=supabase&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-database-4169E1?logo=postgresql&logoColor=white)
![Vercel](https://img.shields.io/badge/Vercel-deployment-000000?logo=vercel&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-ES2022-F7DF1E?logo=javascript&logoColor=black)

---

## Problem

Internal medicine teams managing *C. difficile* infection need to follow each patient across multiple touchpoints: the initial diagnostic visit (severity, vulnerability, recurrence risk, treatment) and one or more follow-up visits (recurrence, persistence of symptoms, new antibiotic exposure). Doing this on paper or in scattered spreadsheets makes it hard to link episodes, track recurrences, and review antibiotic use consistently.

Clostridium Tracker provides a single structured tool for capturing this workflow. It was designed with and for a clinical team, mirroring the questions and scoring scales they use in practice, so that data is recorded uniformly and episodes stay linked over time.

## Live Demo

[https://clostridium-tracker.vercel.app/](https://clostridium-tracker.vercel.app/)

## Features

All features below correspond to the actual application code.

**Patient identification (`PatientLookup`).** Look up a patient by their clinical history number (NHC). If the patient does not yet exist, a short form creates a new record with date of birth and sex.

**Multi-step initial survey wizard (`SurveyWizard`).** A five-step guided form for a principal (diagnosis/treatment) record, with a progress indicator and per-step validation:

1. *Diagnosis & severity* — whether testing was indicated, patient origin (procedencia), clinical data, and a vulnerability question.
2. *Recurrence risk (GEIH-CD)* — a scoring guide table, manual entry of the total GEIH-CD points with an auto-suggested risk level (Bajo / Intermedio / Alto) that the clinician can confirm or override, plus dynamic tables for previous episodes and current-episode treatment (each with a "No aplica" option), and additional measures (stop PPI, stop concomitant antibiotics).
3. *Antibiotic analysis* — a table of antibiotics received in the prior 3 months and stewardship questions (was any antibiotic unnecessary, was a lower-risk alternative available, was there room to shorten duration) with conditional follow-up fields.
4. *Recommended treatment* — see Treatment display below, plus a free-text field for the rationale when the chosen treatment differs from the recommendation.

**Clinical data & severity calculation (`ClinicalData`).** Captures leukocytes, daily stool count, and severity criteria (renal impairment, hypotension/shock, paralytic ileus, megacolon). It computes a suggested severity (Leve / Grave / Fulminante) from these inputs and lets the clinician confirm or correct the final severity, flagging when the manual choice differs from the calculated one.

**Treatment display (`TreatmentDisplay`).** Looks up a recommended treatment from a reference guideline table based on episode type (first vs. recurrence), severity, vulnerability, and recurrence-risk score, and displays the matching recommendation.

**Follow-up form (`FollowUpForm`).** Records a follow-up visit linked to a specific prior diagnosis for the same patient. It loads the patient's existing principal records to link against, captures visit date/type (telephone or in-person) and timing (10–15 days, 8 weeks, or other), and asks the three follow-up questions (recurrence, altered bowel rhythm, new antibiotics this period — with a detail table). If recurrence is confirmed, the user can open a new principal form pre-linked as a recurrence episode.

**Episode linking & recurrence chains.** Follow-ups link back to their initial diagnosis, and a new recurrence episode links to both its originating follow-up and the initial diagnosis, so the full timeline for a patient can be reconstructed.

**Role-based / credential authentication (`Auth`).** A custom username + password sign-in backed by a `usuarios` table, with account request (self-registration with a generated temporary password), forced password change on first login (`must_change_password`), and password recovery via the registered email/phone. Passwords are SHA-256 hashed client-side before being sent. The signed-in user's username is stored on every record for traceability. Sessions are held in browser storage and cleared on logout and on tab/window close.

**Human-readable clinical views.** SQL views translate the stored data into a per-patient clinical timeline in plain language (no raw TRUE/FALSE/NULL), and into relationship views linking follow-ups to diagnoses and recurrence chains.

> Note: authentication is implemented as a custom credential layer against the `usuarios` table through the Supabase client — not Supabase's built-in (GoTrue) Auth service.

## Architecture

```
┌─────────────┐     HTTPS      ┌──────────────────────────┐
│   Browser   │ ─────────────► │  React + Vite frontend   │
│  (clinician)│ ◄───────────── │  (SPA, deployed on Vercel)│
└─────────────┘                └────────────┬─────────────┘
                                             │  @supabase/supabase-js
                                             ▼
                            ┌──────────────────────────────────────┐
                            │              Supabase                  │
                            │                                        │
                            │  Custom credential auth                │
                            │   └─ usuarios table (SHA-256 hashes)   │
                            │                                        │
                            │  PostgreSQL                            │
                            │   ├─ pacientes        (patients)       │
                            │   ├─ encuestas        (episodes/forms) │
                            │   ├─ usuarios         (app users)      │
                            │   ├─ ref_guia_tratamiento (guideline)  │
                            │   └─ views:                            │
                            │       vw_encuestas_episodios           │
                            │       vw_seguimiento_diagnostico       │
                            │       vw_cadena_recidiva               │
                            │       vw_medicos_resumen_humano        │
                            └──────────────────────────────────────┘
```

The frontend is a single-page React application talking directly to Supabase via the JavaScript client. Authentication, patient lookup, survey submission, and follow-up linking are all performed as queries against PostgreSQL tables.

## Data Model

The schema centres on patients and a single episodes table that stores both principal records and follow-ups, distinguished by a `registro_tipo` field inside a JSON column. Relationships between episodes are stored as references inside that JSON (`diagnostico_inicial_id`, `seguimiento_origen_id`) and surfaced through SQL views.

**`pacientes`** — patients. Key columns: `numero_historia_clinica` (clinical history number, primary key), `fecha_nacimiento`, `sexo` (Hombre / Mujer).

**`encuestas`** — the central table holding every recorded form (both initial/principal episodes and follow-up visits). Key columns include: `id` (UUID), `created_at`, `fecha_visita`, `paciente_nhc` (foreign key → `pacientes.numero_historia_clinica`); diagnostic fields `indicacion_pruebas`, `procedencia`; clinical fields `leucocitos`, `deposiciones`, `creatinina`, `deterioro_renal`, `hipotension_shock`, `ileo_paralitico`, `megacolon`; severity `gravedad_calculada`, `gravedad_episodio`; vulnerability `es_vulnerable_calculado`, `paciente_vulnerable_manual` and the reasons `es_oncologico`, `es_neutropenico`, `es_trasplante`, `es_eii`, `es_antibiotico_prolongado`; recurrence risk `riesgo_recurrencia_puntos`, `nivel_riesgo_geih`; JSON tables `episodios_previos`, `tratamiento_actual`, `antibioticos_previos`, `acortar_duracion_ejemplos`; stewardship fields `antibiotico_innecesario`, `opcion_menor_riesgo`, `opcion_menor_riesgo_ejemplos`, `margen_acortar_duracion`; measures `suspension_ibp`, `suspension_antibiotico`; `motivo_tratamiento_medico`, `tratamiento_medico_real`; a full `datos_clinicos` JSON snapshot (which also carries `registro_tipo`, `tipo_registro_principal`, `diagnostico_inicial_id`, `seguimiento_origen_id`, and the follow-up sub-objects); and `usuario` (foreign key → `usuarios.username`). Row Level Security is enabled on this table.

**`usuarios`** — application users. Columns: `id` (UUID), `username` (unique), `password_hash`, `email_or_phone`, `must_change_password`, `created_at`.

**`ref_guia_tratamiento`** — reference lookup table of treatment guidelines, queried by the treatment display. Columns referenced in code: `tipo_episodio` (Primer / Recurrencia), `gravedad`, `es_vulnerable`, `condicion_riesgo` (`< 3` / `>= 3` / `No`), `tratamiento_recomendado`.

**Views** (read-only, defined in the SQL files):

- `vw_encuestas_episodios` — unified list of all episodes (principal + follow-up) with the key linking fields extracted from `datos_clinicos`.
- `vw_seguimiento_diagnostico` — each follow-up joined to its initial diagnosis.
- `vw_cadena_recidiva` — recurrence chain: a new principal-by-recurrence episode joined to its originating follow-up and initial diagnosis.
- `vw_medicos_resumen_humano` — a per-patient clinical timeline rendered in plain human-readable language (no raw TRUE/FALSE/NULL).

> No real patient data is stored in this repository. The schema and views describe structure only.

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 7, React Router DOM 7, plain JavaScript (ES modules) |
| Backend / API | Supabase (`@supabase/supabase-js`) accessed directly from the SPA |
| Database | PostgreSQL (managed by Supabase) with SQL migrations and views; Row Level Security enabled on `encuestas` |
| Auth | Custom username/password layer on a `usuarios` table, SHA-256 client-side hashing (not Supabase GoTrue Auth) |
| Deployment | Vercel (SPA), configured via `vercel.json` |
| Tooling | ESLint 9, Vite dev server |

## Repository Structure

```
clostridium-patient-tracker/
├── README.md
├── package.json
├── vercel.json
├── .env.example                       # variable NAMES only — no secrets
├── DB_VISUALIZACION.md                # how to load the SQL views in Supabase
├── src/
│   ├── main.jsx
│   ├── App.jsx                        # session handling + top-level routing
│   └── components/
│       ├── Auth.jsx                   # login / register / recover / change password
│       ├── PatientLookup.jsx          # find or create a patient by NHC
│       ├── SurveyWizard.jsx           # 5-step initial diagnosis/treatment form
│       ├── ClinicalData.jsx           # clinical inputs + severity calculation
│       ├── TreatmentDisplay.jsx       # guideline-based treatment recommendation
│       └── FollowUpForm.jsx           # follow-up visit linked to a diagnosis
├── sql/
│   ├── vistas_vinculos_encuestas.sql
│   └── vw_medicos_resumen_humano.sql
└── supabase/
    └── migrations/                    # schema + views (encuestas, usuarios, views)
```

## Local Setup

**1. Install dependencies**

```bash
git clone https://github.com/mtapiacosta/clostridium-patient-tracker
cd clostridium-patient-tracker
npm install
```

**2. Configure environment variables**

Copy the example file and fill in your own Supabase project values. The required variable names (from `.env.example`) are:

```
VITE_SUPABASE_URL
VITE_SUPABASE_ANON_KEY
```

```bash
cp .env.example .env   # then edit .env with your values
```

The real `.env` file is never committed (see the security note below).

**3. Set up the database (Supabase)**

Create a Supabase project, then run the SQL in `supabase/migrations/` (and the view scripts in `sql/`) from the Supabase SQL Editor to create the tables and views. See `DB_VISUALIZACION.md` for loading the human-readable views.

**4. Run the app**

```bash
npm run dev        # start the Vite dev server
npm run build      # production build
npm run preview    # preview the production build
```

## Important Note on Medical Data

This application handles medical data. The `.env` file containing Supabase credentials is excluded from the repository and must be supplied locally. **No real patient data is included in this repository** — only the application code and the database schema/views, which describe structure only. Anyone deploying this tool is responsible for ensuring compliance with the applicable data-protection and medical-data regulations in their jurisdiction.
