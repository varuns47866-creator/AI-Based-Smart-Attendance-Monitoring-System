# AI-Based-Smart-Attendance-Monitoring-System

This is a production-grade, modular, real-world operational platform for tracking classroom, office, or controlled zone attendance using computer vision and facial recognition.

---

## 1. System Architecture

The application is structured into clearly bounded layers:
1. **Frontend**: React + TypeScript + Vite + Tailwind CSS v4, driving a responsive SaaS panel with WebSocket notifications.
2. **Backend**: FastAPI + SQLAlchemy + SQLite (Dev) / PostgreSQL (Prod) providing asynchronous DB actions, role-based authorization (RBAC), and append-only audits.
3. **Computer Vision Pipeline**:
   - **Ingestion**: Background stream workers capture video frames via OpenCV from RTSP, USB, or test video feeds (with a simulated driver fallback).
   - **Detection**: Haar Cascade/pluggable detectors extract face bounding boxes and keypoints.
   - **Quality**: Filters poor frames on blur (Laplacian variance) and size thresholds.
   - **Liveness**: Analyzes motion variance (rejecting static paper/screen spoofs) and texture distributions.
   - **biometrics**: Generates 128D embeddings using ONNX MobileFaceNet.
   - **Matching**: Calculates Cosine Similarity against enrolled profiles, confirming identity over N consecutive frames.
   - **Policy Engine**: Computes student timelines and labels attendance status (`PRESENT`, `LATE`, `PARTIAL`, `ABSENT`).

---

## 2. Directory Layout

```text
ai-attendance-system/
├── backend/
│   ├── app/
│   │   ├── api/          # REST endpoints and WebSocket routers
│   │   ├── core/         # Config, security, and WS event broker
│   │   ├── database/     # SQLAlchemy models and connection sessions
│   │   ├── services/     # Audits, enrollments, policies, alerts services
│   │   ├── ai/           # Detector, tracker, liveness, embedding generator
│   │   ├── workers/      # Ingestion thread controllers
│   │   └── main.py       # Application bootstrap
│   └── requirements.txt
├── frontend/             # Vite + TS React Client code
├── docker-compose.yml    # Orchestration manifest
└── README.md
```

---

## 3. Local Quick Start

### Backend Installation

1. Create and activate a virtual environment:
   ```bash
   cd backend
   python -m venv venv
   # On Windows PowerShell:
   .\venv\Scripts\Activate.ps1
   # On Linux/macOS:
   source venv/bin/activate
   ```
2. Install packages:
   ```bash
   pip install --upgrade pip
   pip install -r requirements.txt
   ```
3. Initialize schemas and seed database data:
   ```bash
   python app/scripts/init_db.py
   ```
4. Run server:
   ```bash
   uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
   ```

### Frontend Client Execution

1. Navigate to the client directory and run install:
   ```bash
   cd frontend
   npm install
   ```
2. Start Vite dev server:
   ```bash
   npm run dev
   ```
   *The proxy is preconfigured to route all `/api` and `/ws` traffic to `http://127.0.0.1:8000`.*

---

## 4. Production Docker Deployment

Deploy the entire stack (including Postgres with `pgvector` extension support):
```bash
docker-compose up --build
```
- Frontend will be exposed on: `http://localhost` (Port 80).
- Backend REST documentation will be exposed on: `http://localhost:8000/docs`.

---

## 5. Main API Endpoints

### Authentication & Profiles
- `POST /api/v1/auth/login` - Authenticate and retrieve JWT token pair.
- `POST /api/v1/auth/refresh` - Submit refresh token for rotation.
- `GET /api/v1/users/me` - Read current profile details.

### Biometric Enrollment
- `POST /api/v1/people` - Register new individual.
- `POST /api/v1/people/{id}/enrollment/start` - Initialize enrollment session.
- `POST /api/v1/people/enrollment/{profile_id}/sample` - Upload binary face crop for orientation check (FRONTAL, LEFT, RIGHT).
- `POST /api/v1/people/enrollment/{profile_id}/complete` - Verify sample count and finalize biometric status.

### Hardware & Logical Mapping
- `POST /api/v1/zones` - Create classroom/gate zones.
- `POST /api/v1/cameras` - Register USB/RTSP sources (launches background streams).
- `POST /api/v1/cameras/{id}/test` - Execute connection handshakes.

### Registers & Audits
- `GET /api/v1/attendance` - Query registers with date/course filters.
- `POST /api/v1/attendance/{id}/correction` - Administrative manual corrections (requires audit reason).
- `GET /api/v1/attendance/export/csv` - Download spreadsheet export.
- `POST /api/v1/unknown-events/{id}/review` - Map unknown detections to registered face profiles.
- `GET /api/v1/audit-logs` - Query action audit trails.
- `GET /api/v1/alerts` - Query real-time liveness/spoofing alerts.
