# Limdocs

**Turn course materials into AI-generated practice quizzes and track what you need to study next.**

Limdocs is an adaptive learning platform for students: create course spaces, upload lecture PDFs and slides, extract text automatically, generate exam-style question sets with explanations, practice in the app, and review performance by topic over time.

| | |
|---|---|
| **Frontend** | React 19, Vite, React Router, AWS Amplify Auth, AWS Amplify Hosting (GitHub-connected deploys) |
| **Backend** | Python 3.9 on AWS Lambda (SAM), API Gateway, Cognito |
| **AI / OCR** | Amazon Textract, OpenAI (`gpt-4.1-mini`) |
| **Data** | DynamoDB, S3 |

---

## What you can do today

- **Sign up and sign in** with AWS Cognito (email/username), including account confirmation
- **Switch Hebrew ↔ English** with persisted preference and full RTL/LTR layout
- **Create and manage courses** (private or public visibility) from the dashboard
- **Upload materials** via S3 pre-signed URLs (PDF, PNG, JPEG)
- **Process documents** asynchronously with Textract, then extract bilingual sub-topics with OpenAI
- **Generate question sets** from one or more ready documents (async worker, status polling in the UI)
- **Take quizzes**, submit attempts, and review correct answers with explanations
- **Browse attempt history** and reopen past submissions in read-only review mode
- **View weakness analytics** per topic and difficulty on the course **Analyzed Weaknesses** tab (fed by quiz submissions)
- **Delete** documents, attempts, question sets, or entire courses (cascading S3 + DynamoDB cleanup)

Some dashboard chrome (global search, Documents / Analytics nav items) is presentational only and does not yet route to separate pages.

---

## Architecture

Serverless, event-driven design on AWS:

```
Browser (React SPA)
    → AWS Amplify Hosting (connected to GitHub for frontend CI/CD deploys)
    → Cognito (auth) + API Gateway (REST, Cognito authorizer)
        → Lambda handlers (courses, uploads, quizzes, attempts, progress)
    → S3 raw uploads (pre-signed PUT) → S3 event → process_document Lambda
        → Textract (async) → processed text in S3 → DynamoDB document status
        → OpenAI topic extraction → status READY
    → generate-quiz API → async worker Lambda → question_sets + questions tables
    → submit attempt → grades answers → attempts + attempt_answers + user_progress matrix
```

### AWS resources (SAM stack)

| Service | Role |
|---------|------|
| **Cognito User Pool** | Authentication for the SPA |
| **AWS Amplify Hosting** | Frontend hosting and CI/CD deployment from GitHub |
| **API Gateway** | REST API (`/prod` stage), default Cognito authorizer |
| **Lambda** | Business logic, Textract orchestration, quiz worker |
| **DynamoDB** | `users`, `courses`, `documents`, `question_sets`, `questions`, `attempts`, `attempt_answers`, `user_progress` |
| **S3** | Raw uploads (`uploads/` prefix) and processed text outputs |
| **Textract** | OCR for uploaded PDFs/images |
| **OpenAI** | Sub-topic extraction after OCR; quiz question generation |

### Document processing lifecycle

Typical `processing_status` values on a document:

| Status | Meaning |
|--------|---------|
| `UPLOADED` | Metadata recorded; file in S3 |
| `PROCESSING` | Textract job in progress |
| `READY` | Text extracted, topics available, eligible for quiz generation |
| `GENERATING` | Quiz worker running for this document |
| `FAILED` / `ERROR` | Processing or generation failed (`failure_reason` when set) |

The course page polls materials while any document is not in a terminal state (`READY`, `FAILED`, `ERROR`).

### Deletion semantics

Deletes follow **S3 first, then DynamoDB** for documents. Deleting a course cascades through documents (raw + processed buckets), question sets, questions, attempts, and attempt answers before removing the course row.

---

## Repository layout

```
Limdocs/
├── frontend/          # React + Vite SPA
├── backend/
│   ├── template.yaml  # SAM / CloudFormation
│   └── src/           # Lambda handlers
├── docs/
│   ├── design.md      # Product & architecture design (text)
│   └── progress.log.md
└── package.json       # npm workspaces (runs frontend scripts from root)
```

---

## Local development

### Prerequisites

- Node.js 18+
- Python 3.9+ (matches Lambda runtime in `backend/template.yaml`)
- [AWS CLI](https://aws.amazon.com/cli/) configured
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)

### Frontend

From the repo root:

```bash
npm install
npm run dev
```

Or from `frontend/`:

```bash
cd frontend
npm install
npm run dev
```

Copy `frontend/.env.example` to `frontend/.env.local` and set values from SAM stack outputs after deploy:

```bash
VITE_COGNITO_USER_POOL_ID=<UserPoolId>
VITE_COGNITO_USER_POOL_CLIENT_ID=<UserPoolClientId>
VITE_API_URL=https://<api-id>.execute-api.<region>.amazonaws.com/prod
```

The app uses the Cognito **ID token** as `Authorization: Bearer` for API Gateway.

**Routes:** `/login`, `/register`, `/home`, `/course/:courseId`

### Backend (SAM)

```bash
cd backend
sam build            # Windows: sam build --use-container
sam validate
sam deploy --guided
```

During deploy you will be prompted for **`OpenAIApiKey`** (`NoEcho` parameter). Quiz generation and topic extraction depend on it.

`sam build` must bundle Python dependencies from `backend/src/requirements.txt` (including `openai`). Do not deploy only `.py` sources without building, or workers will fail with `ModuleNotFoundError: no module named 'openai'`.

The stack is configured for **AWS Learner Lab** style accounts (`LabRole` IAM role). Adjust `Role` in `template.yaml` for other environments.

**Key outputs:** `ApiUrl`, `UserPoolClientId`, bucket names, table names (see `Outputs` in `template.yaml`).

### Main API surface

All routes require Cognito auth unless noted. Owner checks apply on course-scoped operations.

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/users` | Sync user profile after signup |
| `POST` | `/courses` | Create course |
| `GET` | `/users/{userId}/courses` | List caller's courses |
| `DELETE` | `/courses/{courseId}` | Delete course and all related data |
| `POST` | `/courses/{courseId}/upload-url` | Pre-signed upload + document row |
| `GET` | `/courses/{courseId}/materials` | List course documents |
| `DELETE` | `/courses/{courseId}/documents/{documentId}` | Delete document |
| `POST` | `/courses/{courseId}/generate-quiz` | Start async question generation (`202`) |
| `GET` | `/courses/{courseId}/question-sets` | List question sets |
| `GET` | `/courses/{courseId}/question-sets/{setId}` | Question set detail |
| `DELETE` | `/courses/{courseId}/question-sets/{setId}` | Delete question set |
| `POST` | `/courses/{courseId}/question-sets/{setId}/attempts` | Submit quiz attempt |
| `GET` | `/courses/{courseId}/attempts` | List attempts |
| `GET` | `/courses/{courseId}/attempts/{attemptId}/answers` | Attempt review payload |
| `DELETE` | `/courses/{courseId}/attempts/{attemptId}` | Delete attempt |
| `GET` | `/courses/{courseId}/progress` | Topic/difficulty mastery matrix |

S3 uploads trigger `process_document` automatically (no HTTP route).

---

## Technology stack

| Layer | Choices |
|-------|---------|
| **UI** | React 19, Vite 8, CSS (logical properties for RTL/LTR), shared design tokens in `frontend/src/index.css` |
| **Client auth** | `aws-amplify` v6 |
| **Frontend hosting** | AWS Amplify Hosting (GitHub-connected build/deploy) |
| **HTTP** | `axios` service modules under `frontend/src/services/` |
| **IaC** | AWS SAM (`backend/template.yaml`) |
| **Compute** | AWS Lambda (Python 3.9) |
| **Auth** | Amazon Cognito User Pools |
| **API** | Amazon API Gateway (REST) |
| **Storage** | DynamoDB, S3 |
| **OCR** | Amazon Textract (async document text detection) |
| **LLM** | OpenAI API via official Python SDK |

---

## Documentation

- [Design document](docs/design.md) — product goals, AWS module mapping, data model notes
- [Progress log](docs/progress.log.md) — chronological engineering milestones

---

## Authors

- Yarden Vaknin
- Nadav Masliah

*Cloud Computing Workshop with AWS — The Academic College of Tel Aviv-Yafo.*
