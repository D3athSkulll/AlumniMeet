
# IIITM Alumni Meet and Network

An alumni preservation website for the IIITM community, built to support Alumni Meet events and make alumni connections easier. This presents an interactive alumni directory, missing-alumni search, profile update workflow, and restricted volunteer review portal.

The alumni network is backed by MongoDB and currently contains **4,500+ alumni records**. The directory lets users find people by name, academic program, entry year, company, country, city, role, specialization, club involvement, roll number, and nature of employment.

## Website Demonstration

A complete walkthrough is available in [assets/Video.mp4](./assets/Video.mp4). It demonstrates the website, the alumni directory, and the missing-alumni search functionality.

https://github.com/user-attachments/assets/e766f9d2-e270-458d-b0f0-333ca61b75d1



## Key Features

- Responsive public landing page with light and dark themes
- Alumni Meet information, timeline, event messages, gallery, and distinguished alumni
- Campus visit, donation, contact, and volunteer information
- Alumni directory backed by 4,500+ MongoDB records
- Basic and advanced alumni search
- Pagination for large result sets
- Missing-alumni search for incomplete or deceased job records
- Alumni profile update request submission
- Volunteer-only review workflow for approving or rejecting update requests
- Client-side route support for static hosting platforms

## Alumni Directory

The directory is available at `/directory` and is the project's main alumni network feature.

### Basic filters

- Name
- Year of entry
- Program name
- Current or last organization

### Advanced filters

- Roll number
- Country
- City
- Nature of job
- Specialization
- Last position
- College clubs

The specialization options are dynamically limited by the selected program. For example:

- `IPG` supports `MBA` and `MTECH`
- `MBA` supports `BA`, `IFS`, `ISM`, `ITES`, `NFS`, and `PSM`
- `MTECH` supports `AN`, `BI`, `CN`, `DC`, `ICS`, `IS`, `IT`, `SE`, `VLSI`, and `WNC`
- `PHD` supports `PIT`, `PITF`, `PMG`, `PMGF`, `PAM`, `PAMF`, and `PAP`

Nature-of-job options include academia, banking, career breaks, corporate roles, deceased status, entrepreneurship, freelance work, global roles, and government roles.

### Search behavior

Search values are sent to the backend through `api/search` as URL query parameters. Text fields are designed for partial and flexible matching rather than exact-only lookup. The backend can use MongoDB regular expressions for case-insensitive text searches across fields such as name, company, city, country, position, and club information.

Search requests support:

- Partial text matching with regular expressions
- Combinations of multiple filters
- Numeric year filtering
- Country normalization
- Program and specialization relationships
- Server-side pagination with a 20-record page size
- Sorted result output by year of entry
- Loading, timeout, empty-result, and API error states

Example directory request:

```text
GET /api/search?name=rahul&yearOfEntry=2020&programName=BCS&page=1&limit=20
```

Example missing-alumni request:

```text
GET /api/missing_alumni?yearOfEntry=2018&programName=MBA&specialization=ISM&page=1&limit=20
```

## Missing-Alumni Search

The missing-alumni feature is available at `/missing_alumni`. It helps the Alumni Cell identify records that need follow-up, including alumni whose job status is unknown or marked as deceased.

Filters include:

- Year of entry
- Program name
- Specialization

Results include name, roll number, entry year, program, specialization, organization, location, and LinkedIn profile when available. Paginated navigation is provided for every result set.

## Alumni Profile Update Workflow

Each directory row includes an edit action. An authenticated volunteer or administrator can review a submitted update before it changes the public record.

The update form supports basic, academic, professional, location, and social profile information. Submissions are sent to the backend as an update request containing:

- Roll number
- Existing alumni data
- Proposed new data

The request can be approved or rejected through the restricted `/volunteer-portal-secret` interface. Requests have pending, approved, and rejected states, optional reviewer notes, submission timestamps, and review timestamps.

## MongoDB and Backend Integration

This repository contains the React frontend. It communicates with a separately hosted MongoDB-backed API through `VITE_DATABASE_URL`.

The backend is expected to provide these endpoints:

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/api/search` | Search and paginate alumni records |
| `GET` | `/api/missing_alumni` | Search incomplete or deceased alumni records |
| `POST` | `/api/update-request` | Submit a profile update request |
| `GET` | `/api/volunteer/update-requests` | Fetch update requests for volunteers |
| `POST` | `/api/volunteer/update-requests/:id/approve` | Approve an update request |
| `POST` | `/api/volunteer/update-requests/:id/reject` | Reject an update request |

Typical MongoDB field usage includes:

- `name`
- `rollNumber`
- `department`
- `yearOfEntry`
- `yearOfGraduation`
- `programName`
- `specialization`
- `lastOrganization`
- `lastPosition`
- `natureOfJob`
- `country`
- `currentLocationIndia`
- `currentOverseasLocation`
- `collegeClubs`
- `email`
- `phone`
- `linkedIn`
- `twitter`
- `instagram`
- `facebook`
- `achievements`
- `photoLink`

For regex-based MongoDB queries, text values should be escaped and normalized by the backend before constructing expressions. Matching should be case-insensitive and avoid exposing unsupported regular-expression input to the database.

## Frontend Stack

- React 18
- TypeScript
- Vite 6
- React Router 7
- Tailwind CSS 3
- Framer Motion
- GSAP
- Swiper
- React Select
- Lucide and Tabler icons

## Project Structure

```text
AlumniMeet/
├── assets/
│   └── Video.mp4                 # Website and directory walkthrough
├── public/
│   └── _redirects                # Netlify SPA route fallback
├── src/
│   ├── assets/                   # Images, award portraits, and gallery files
│   ├── components/
│   │   ├── Navbar.tsx            # Main navigation
│   │   ├── Network.tsx           # Alumni directory and search
│   │   ├── MissingAlumni.tsx     # Missing-alumni search
│   │   ├── UpdateDetailsDialog.tsx
│   │   ├── VolunteerPortal.tsx   # Update-request review
│   │   ├── Gallery.tsx
│   │   ├── DistinguishedAlumni.tsx
│   │   └── ...                   # Landing page sections
│   ├── context/
│   │   └── ThemeContext.tsx      # Global light/dark theme state
│   ├── hooks/
│   │   ├── useLocationAPI.ts     # Country and city API lookup
│   │   └── useScrollAnimation.ts
│   ├── pages/
│   │   ├── AlumniMeet/           # Dedicated Alumni Meet page
│   │   ├── Directory/            # Directory route wrapper
│   │   └── MissingAlumni/        # Missing-alumni route wrapper
│   ├── App.tsx                   # Routes and main site layout
│   ├── main.tsx                  # React entry point
│   └── index.css
├── .env.example
├── netlify.toml
├── render.yaml
├── vercel.json
└── package.json
```

## Routes

| Route | Description |
| --- | --- |
| `/` | Main alumni website |
| `/alumnimeet` | Alumni Meet event page |
| `/directory` | Searchable alumni network |
| `/missing_alumni` | Missing/deceased alumni search |
| `/volunteer-portal-secret` | Restricted update-review portal |

## Local Setup

### Prerequisites

- Node.js 18 or newer
- npm
- A running MongoDB-backed backend API

### Install and run

```bash
npm install
cp .env.example .env
npm run dev
```

On Windows PowerShell, create the environment file with:

```powershell
Copy-Item .env.example .env
```

Set the backend URL in `.env`:

```env
VITE_DATABASE_URL=http://localhost:3001/
```

For a hosted backend, use the deployed API URL:

```env
VITE_DATABASE_URL=https://your-backend-name.onrender.com/
```

The application is then available at the URL printed by Vite, typically `http://localhost:5173`.

## Build and Quality Checks

```bash
npm run build
npm run lint
npm run preview
```

`npm run build` runs TypeScript project compilation and creates the production bundle in `dist`.

## Deployment

The repository includes deployment configuration for Render, Netlify, and Vercel.

- `render.yaml` configures a static frontend service
- `netlify.toml` and `public/_redirects` provide SPA route rewrites
- `vercel.json` rewrites application routes to `index.html`

Configure `VITE_DATABASE_URL` in the hosting platform's environment settings. Vite environment variables are compiled into the frontend build, so rebuild after changing the backend URL.

## Security Notes

- Never commit database credentials or API secrets
- Keep backend MongoDB connection strings server-side
- Validate and escape all regex input in the backend
- Use normalized case-insensitive searches
- Protect volunteer review endpoints with bearer authentication
- Do not store the volunteer secret in committed source files
