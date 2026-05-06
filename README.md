# MedTrace AI — Radiology Dashboard

An AI-assisted radiology workflow system that combines medical image segmentation (MedSAM2) with vision-language report generation (MedGemma / Qwen VL) to produce draft radiology reports from DICOM studies, ready for physician review.

---

## Architecture

```
┌──────────────┐      ┌────────────────────────┐      ┌─────────────────────┐
│  Radiologist │────▶│     Frontend SPA        │────▶│   FastAPI Backend   │
│  (Browser)   │◀────│  React 19 + TypeScript  │◀────│   Python + Pydantic │
└──────────────┘      │  Vite + TailwindCSS    │      └────────┬────────────┘
                      │                        │               │
                      │  ┌─StudyPanel──────┐   │     ┌─────────┴───────────┐
                      │  │ Upload + List   │   │     │                     │
                      │  └─────────────────┘   │     ▼                     ▼
                      │  ┌─ViewerWorkspace─┐   │   ┌──────────┐  ┌──────────────┐
                      │  │ DICOM + ROI     │   │   │ MedSAM2  │  │  MedGemma    │
                      │  │ Seg Overlay     │   │   │ Service  │  │  Service     │
                      │  └─────────────────┘   │   └────┬─────┘  └──┬─────┬─────┘
                      │  ┌─DecisionPanel───┐   │        │           │     │
                      │  │ Accept / Reject │   │        ▼           ▼     ▼
                      │  │ Confidence      │   │   ┌─────────┐ ┌──────┐ ┌───────┐
                      │  └─────────────────┘   │   │HTTP/    │ │Nebius│ │Hugging│
                      └────────────────────────┘   │Mock     │ │QwenVL│ │Face   │
                                                   └─────────┘ └──────┘ └───────┘
```

---

## Workflow

```
1. Upload          Radiologist uploads a DICOM study file
                      │
2. Preview           Backend converts DICOM → PNG (windowing + rescale)
                      │
3. Draw ROI          Radiologist draws a Region of Interest on the image
                      │
4. Segment           ROI coords → MedSAM2 Service → Segmentation mask overlay
                      │
5. Generate Report   Segmented image + prompt → MedGemma/Qwen VL → Draft report
                      │
6. Review            Radiologist reviews: Accept Draft or Needs Correction
                      │
7. Feedback          (Optional) Doctor provides correction feedback → regenerate
```

---

## Project Structure

```
MedTrace-AI-radiology/
├── CLAUDE.md                      # Project guidance for AI coding agents
├── architecture-diagram.html      # Interactive SVG architecture diagram
├── README.md                      # This file
├── .gitignore
│
├── frontend/                      # React 19 SPA
│   ├── index.html                 # Vite entry point
│   ├── package.json               # Dependencies (React 19, Vite 7, TailwindCSS, Lucide)
│   ├── vite.config.ts             # Vite configuration
│   ├── tailwind.config.js         # Custom shadows (active-glow, panel-soft)
│   ├── postcss.config.js
│   ├── tsconfig.json              # TypeScript root config
│   ├── tsconfig.app.json          # App source config (ES2022, strict)
│   ├── tsconfig.node.json         # Vite config TS settings
│   └── src/
│       ├── main.tsx               # React root mount
│       ├── App.tsx                # Entire SPA (1167 lines, single-file)
│       │   ├── Types              # ReviewDecision, Study, RoiBox, Segmentation, etc.
│       │   ├── API functions      # postStudyFile, requestSegmentation, requestReport
│       │   ├── App component      # State management, keyboard shortcuts, event handlers
│       │   ├── StudyPanel         # Left sidebar: upload + study list
│       │   ├── ViewerWorkspace    # Center: DICOM viewer, ROI drawing, toolbar
│       │   ├── DecisionPanel      # Right sidebar: report display, accept/reject
│       │   ├── FeedbackModal      # Doctor correction feedback overlay
│       │   └── Helper components   # Overlays, status dots, action cards
│       ├── styles.css             # Tailwind directives + dark theme + CSS medical-scan art
│       └── vite-env.d.ts          # Vite client types
│
└── backend/                       # FastAPI Python server
    ├── README.md                  # Detailed API contract & MCP integration docs
    ├── app.py                     # Main FastAPI app (routes, DICOM processing)
    │   ├── Routes                 # /health, /llm/status, /studies, /segmentations, /reports
    │   ├── DICOM processing       # Rescale, windowing, percentile fallback, PNG output
    │   └── Pydantic models        # RoiPrompt, SegmentationRequest/Response, etc.
    ├── requirements.txt           # fastapi, uvicorn, pydicom, pillow, httpx, numpy
    └── model_adapters/
        ├── __init__.py            # Exports MedSAM2Service, MedGemmaService
        ├── medsam2.py             # MedSAM2 segmentation adapter (3 modes)
        │   ├── HTTP endpoint      # POST to external MedSAM2 server /segment
        │   ├── Local adapter      # Dynamic import of MEDSAM2_ADAPTER_MODULE
        │   └── Mock               # Deterministic fake segmentation
        └── medgemma.py            # MedGemma report generation adapter (4 modes)
            ├── Nebius Qwen VL     # OpenAI-compatible chat/completions API
            ├── HTTP endpoint      # Forward to external /generate-report server
            ├── Local HuggingFace  # transformers pipeline (image-text-to-text)
            └── Mock               # Deterministic fake report from ROI count
```

---

## API Endpoints

| Method   | Path                                    | Description                              |
|----------|-----------------------------------------|------------------------------------------|
| GET      | `/health`                               | Health check                             |
| GET      | `/llm/status`                           | LLM model availability status            |
| POST     | `/studies`                              | Upload DICOM file (multipart)            |
| GET      | `/sample-studies/pre-liver`             | Get pre-loaded sample study              |
| POST     | `/studies/{id}/segmentations/medsam2`   | Run MedSAM2 segmentation on ROI          |
| POST     | `/studies/{id}/reports/medgemma`        | Generate report via MedGemma             |
| POST     | `/studies/{id}/reports/qwen-vl`         | Generate report via Nebius Qwen VL       |

### Key Request Schemas

**Segmentation Request:**
```json
{
  "roi_prompts": [
    { "x_min": 0.2, "y_min": 0.3, "x_max": 0.7, "y_max": 0.8 }
  ]
}
```
> ROI coordinates are normalized 0–1; the backend converts to pixel coordinates.

**Report Response:**
```json
{
  "study_id": "...",
  "report": {
    "findings": "...",
    "impression": "...",
    "recommendation": "...",
    "confidence": 0.85
  }
}
```

---

## Model Adapter Modes

### MedSAM2 (Segmentation)

| Mode          | Env Variable          | Description                            |
|---------------|-----------------------|----------------------------------------|
| HTTP endpoint | `MEDSAM2_ENDPOINT`    | POST ROI to external MedSAM2 server    |
| Local adapter | `MEDSAM2_ADAPTER_MODULE` | Dynamic Python module import + call |
| Mock          | *(default)*           | Returns deterministic fake mask        |

### MedGemma (Report Generation)

| Mode          | Env Variable          | Description                            |
|---------------|-----------------------|----------------------------------------|
| Nebius QwenVL | `NEBIUS_API_KEY`      | Cloud API, OpenAI-compatible endpoint  |
| HTTP endpoint | `MEDGEMMA_ENDPOINT`   | Forward to external report server      |
| Local HF      | `MEDGEMMA_MODEL_ID`   | HuggingFace transformers pipeline      |
| Mock          | *(default)*           | Deterministic report from ROI count    |

---

## Environment Variables

| Variable               | Required | Description                                |
|------------------------|----------|--------------------------------------------|
| `NEBIUS_API_KEY`       | No       | API key for Nebius Qwen VL cloud service   |
| `NEBIUS_BASE_URL`      | No       | Nebius API base URL                        |
| `NEBIUS_QWEN_VL_MODEL` | No       | Qwen VL model identifier on Nebius         |
| `MEDSAM2_ENDPOINT`     | No       | URL of external MedSAM2 segmentation server|
| `MEDSAM2_ADAPTER_MODULE`| No      | Python module path for local MedSAM2       |
| `MEDGEMMA_ENDPOINT`    | No       | URL of external MedGemma report server     |
| `MEDGEMMA_MODEL_ID`    | No       | HuggingFace model ID for local MedGemma    |
| `VITE_API_BASE_URL`    | No       | Override backend URL (default: localhost:8000)|

---

## Quick Start

### Prerequisites

- Python 3.10+
- Node.js 18+

### Backend

```bash
cd backend
python3 -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt

# Optional: enable real AI models
cp ../.env.example .env       # Add your API keys

uvicorn app:app --reload --port 8000
```

### Frontend

```bash
cd frontend
npm install
npm run dev                    # Starts on http://localhost:5173
```

Open `http://localhost:5173` — the app auto-connects to the backend at `localhost:8000`.

### With AI Models (Optional)

For real segmentation and report generation, configure at least one model adapter:

```bash
# Option A: Nebius Cloud (easiest, no GPU needed)
NEBIUS_API_KEY=your-key-here

# Option B: Local MedGemma (requires GPU + torch)
pip install torch transformers accelerate
MEDGEMMA_MODEL_ID=google/medgemma-4b-it

# Option C: External model servers
MEDSAM2_ENDPOINT=http://localhost:5001
MEDGEMMA_ENDPOINT=http://localhost:5002
```

Without any model configuration, the system runs in **mock mode** with deterministic fake outputs — useful for UI development and testing.

---

## Frontend Layout

```
┌──────────────────────────────────────────────────────────┐
│  MedTrace AI - Radiology Workflow Dashboard              │
├────────────┬──────────────────────────┬──────────────────┤
│            │  [Seg] [👁] [↩] [↪] [✕] │                  │
│  Studies   │  ┌──────────────────────┐│  Draft Report    │
│            │  │                      ││                  │
│  ▸ Study 1 │  │   DICOM Viewport     ││  Confidence: 85% │
│  ▸ Study 2 │  │   + ROI Drawing      ││                  │
│  ▸ Study 3 │  │   + Seg Overlay      ││  Findings: ...   │
│            │  │                      ││  Impression: ... │
│  [Upload]  │  └──────────────────────┘│  Recs: ...       │
│            │  Brightness [━━━━]       │                  │
│            │  Zoom      [━━━━]       │  [Accept] [Fix]  │
├────────────┴──────────────────────────┴──────────────────┤
│  Ctrl+Z Undo  |  Ctrl+Y Redo  |  ROI: drag on viewport  │
└──────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer      | Technology                                             |
|------------|--------------------------------------------------------|
| Frontend   | React 19, TypeScript, Vite 7, TailwindCSS 3, Lucide   |
| Backend    | FastAPI, Uvicorn, Pydantic v2, httpx, pydicom, Pillow  |
| AI - Seg   | MedSAM2 (HTTP / local adapter / mock)                  |
| AI - Report| MedGemma, Qwen VL via Nebius (HTTP / HuggingFace / mock)|
| Storage    | Local filesystem (DICOM + PNG previews)                |

---

## License

MIT
