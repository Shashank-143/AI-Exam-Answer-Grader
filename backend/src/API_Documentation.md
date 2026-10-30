# AI Exam Answer Grader API

Group 2's reusable NLP service for the AI University Platform. It grades a student's answer against an instructor's reference answer. Every answer is screened by **PromptGuard** before grading.

Built with **FastAPI**. Interactive docs are generated automatically once the server is running:

| URL | What it is |
|---|---|
| `/docs` | Swagger UI (try requests in the browser) |
| `/redoc` | ReDoc reference |
| `/openapi.json` | Raw OpenAPI schema |

---

## What happens at startup

Before the API accepts any request, the `lifespan` handler does the following:

1. It checks that `models/grader_models.joblib` exists. If the file is missing, startup fails with an error asking you to run `src/compare_models.py` and `src/improve_model.py` first.
2. It loads the model bundle with `joblib`. The bundle holds the trained classifiers and the name of the embedding model.
3. It builds a `TextEncoder` for that embedding model and wraps the bundle and encoder in a `V2_NLP_Pipeline`.
4. It keeps the bundle and pipeline in memory, so models load once and are not reloaded on each request.

When the server shuts down, this in-memory state is cleared.

---

## Endpoints

### 1. `GET /health`

This endpoint checks whether the service is up and the models are loaded.

**Internal process**
1. Looks up the loaded model bundle in memory.
2. If no bundle is loaded, returns `503`.
3. Otherwise, reports the embedding model name and the trained models found in the bundle.

**Success response — `200 OK`**
```json
{
  "status": "ok",
  "embedding_model": "all-MiniLM-L6-v2",
  "available_models": ["v3_model", "v2_model", "v1_clf", "baseline_clf"]
}
```

| Field | Meaning |
|---|---|
| `status` | Always `"ok"` when the service is healthy |
| `embedding_model` | Sentence-embedding model used by the pipeline |
| `available_models` | Trained models present in the bundle. Use this to confirm a model exists before selecting it in `/evaluate-answer` |

**Error response**
- `503 Service Unavailable`: `{"detail": "Model bundle not loaded."}`

---

### 2. `POST /evaluate-answer`

This endpoint grades one student answer.

**Query parameter**

| Name | Values | Default | Model used |
|---|---|---|---|
| `model` | `v3` | ✅ | Hybrid features (including `keypoint_min_coverage` and `negation_mismatch`) with tuned Logistic Regression. This is the best validated model. |
| | `v2` | | Week 4 hybrid features with tuned Logistic Regression |
| | `v1` | | MiniLM sentence-embedding cosine similarity with Logistic Regression |
| | `baseline` | | TF-IDF cosine similarity with Logistic Regression |

**Request body (JSON)**

| Field | Type | Required | Description |
|---|---|---|---|
| `question` | string | ✅ | The exam question, or a question ID such as `"Q4"` |
| `reference_answer` | string | ✅ | The instructor's model answer |
| `student_answer` | string | ✅ | The answer to be graded |
| `maximum_marks` | integer ≥ 1 | ✅ | Marks available for the question |
| `marking_rubric` | string | ❌ | Optional mark scheme. The API accepts it so the interface is complete, but it is **not currently used** in grading. |

**Example request**
```bash
curl -X POST "http://127.0.0.1:8000/evaluate-answer?model=v3" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "Q4 What does supervised learning use to train a model?",
    "reference_answer": "Supervised learning trains a model on labelled data, i.e. input-output pairs.",
    "student_answer": "It uses labelled examples where the correct output is known.",
    "maximum_marks": 5
  }'
```

#### Internal process

```
request ──► PromptGuard screening ──► decision
                                        ├─ BLOCK / HUMAN_REVIEW ──► return without grading
                                        ├─ SANITIZE ──► grade the cleaned text
                                        └─ ALLOW (anything else) ──► grade the original text
                                                              │
                                     selected model (v3/v2/v1/baseline)
                                                              │
                                              score + feedback + PromptGuard summary
```

1. **Build a request ID.** The API hashes the question text with SHA-256 and uses the first 16 hex characters to form `repo1-<hash>`, which identifies the screening call.
2. **Screen with PromptGuard.** The API sends only the `student_answer` to `POST {PROMPTGUARD_URL}/v1/analyze`, with `source_type: "exam_answer"`. PromptGuard looks for prompt-injection or manipulation attempts and returns a `recommended_action`.
3. **Act on PromptGuard's decision.**
   - **`BLOCK` or `HUMAN_REVIEW`:** The answer is **not graded**. The API returns `score: null` and `grading_status: "human_review"`, so a teacher must check it manually.
   - **`SANITIZE`:** PromptGuard's cleaned `sanitized_text` replaces the student answer for grading. If PromptGuard asks for sanitisation but sends no usable text, the API returns `502`.
   - **Any other action (for example `ALLOW`):** The original answer is graded.
4. **Grade.** The chosen model compares the student answer with the reference answer and produces a score out of `maximum_marks`, feedback, a confidence value, and any missing key points.
5. **Assemble the response.** The API sets `grading_status: "graded"` and attaches the PromptGuard summary so callers can see what screening found.

#### Success response — `200 OK` (graded)
```json
{
  "question": "Q4 What does supervised learning use to train a model?",
  "score": 4,
  "maximum_score": 5,
  "feedback": "Correct core idea; mention input-output pairs explicitly.",
  "confidence": 0.87,
  "missing_key_points": ["input-output pairs"],
  "model": "v3",
  "grading_status": "graded",
  "promptguard": {
    "attack_detected": false,
    "risk_level": "low",
    "recommended_action": "ALLOW",
    "attack_type": null,
    "sanitized_text": null,
    "request_id": "repo1-3f9a1c2b7d4e5f60"
  }
}
```
*The values above are illustrative. Real scores and feedback come from the model.*

#### Success response — `200 OK` (flagged for human review)
```json
{
  "question": "Q4 ...",
  "score": null,
  "maximum_score": 5,
  "feedback": "PromptGuard flagged this answer for human review; it was not automatically graded.",
  "confidence": null,
  "missing_key_points": [],
  "model": "PromptGuard",
  "grading_status": "human_review",
  "promptguard": {
    "attack_detected": true,
    "risk_level": "high",
    "recommended_action": "HUMAN_REVIEW",
    "attack_type": "prompt_injection",
    "sanitized_text": "...",
    "request_id": "repo1-..."
  }
}
```

#### Response fields

| Field | Type | Meaning |
|---|---|---|
| `question` | string | The question that was sent in the request |
| `score` | int \| null | Marks awarded. This is `null` when the answer was not graded. |
| `maximum_score` | int | Marks available, taken from `maximum_marks` |
| `feedback` | string \| null | Explanation for the student or teacher |
| `confidence` | float \| null | The model's confidence in the score |
| `missing_key_points` | list[str] | Points from the reference answer that the student did not cover. Defaults to `[]`. |
| `model` | string | Which grader produced the result, or `"PromptGuard"` if the answer was flagged |
| `grading_status` | `"graded"` \| `"human_review"` | Whether the answer was auto-graded |
| `promptguard` | object | Summary of the screening: `attack_detected`, `risk_level`, `recommended_action`, `attack_type`, `sanitized_text`, `request_id` |

#### Error responses

| Code | When it happens |
|---|---|
| `422` | The request body is invalid: a required field is missing, `maximum_marks` is less than 1, or `model` has an unknown value |
| `500` | `PROMPTGUARD_TIMEOUT_SECONDS` is not a number or is ≤ 0 |
| `502` | PromptGuard is unreachable, rejected the request, returned invalid or unexpected JSON, or asked for sanitisation without returning text |
| `504` | PromptGuard did not reply within the timeout |

> **Important:** Grading depends on PromptGuard. If PromptGuard is not running, every `/evaluate-answer` call fails with `502`.

---

## Configuration

| Environment variable | Default | Purpose |
|---|---|---|
| `PROMPTGUARD_URL` | `http://127.0.0.1:8001` | Base URL of the PromptGuard API |
| `PROMPTGUARD_TIMEOUT_SECONDS` | `5` | Timeout for each screening call, in seconds |

**CORS:** The API accepts `GET` and `POST` from `localhost` and `127.0.0.1` on ports `5500` and `8080`. Add your frontend's origin in `api.py` if it runs elsewhere.

## Running locally

```bash
# 1. Train the models once (this creates models/grader_models.joblib)
python src/compare_models.py
python src/improve_model.py

# 2. Start PromptGuard on port 8001 (see its own repository)

# 3. Start this API
uvicorn api:app --reload --port 8000
```

Then open `http://127.0.0.1:8000/docs` to test the endpoints.
