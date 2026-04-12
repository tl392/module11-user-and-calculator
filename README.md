# Module 11 — Calculation Model, Factory Pattern & Validation

> **Project:** UserVault — FastAPI + SQLAlchemy + PostgreSQL  
> **Focus:** Defining the `Calculation` SQLAlchemy model, Pydantic schemas with validation, Factory design pattern, unit + integration tests, CI/CD pipeline maintenance.

---

## Table of Contents

1. [What Was Built](#1-what-was-built)
2. [Project Structure Changes](#2-project-structure-changes)
3. [How to Run Locally](#3-how-to-run-locally)
4. [Running the Tests](#4-running-the-tests)
5. [API Endpoints](#5-api-endpoints)
6. [How the Calculation Model Works](#6-how-the-calculation-model-works)
7. [How the Pydantic Schemas Work](#7-how-the-pydentic-schemas-work)
8. [Factory Pattern Explained](#8-factory-pattern-explained)
9. [CI/CD Pipeline](#9-cicd-pipeline)
10. [Docker Hub](#10-docker-hub)

---

## 1. What Was Built

| Requirement | Implementation |
|-------------|---------------|
| SQLAlchemy `Calculation` model | `app/models.py` — fields: `id`, `a`, `b`, `type`, `result`, `user_id` FK, timestamps |
| `OperationType` enum | `add`, `subtract`, `multiply`, `divide` |
| `user_id` foreign key | References `users.id` with `ondelete="CASCADE"` |
| Store or compute result | **Stored at write time** via the factory — fast reads, accurate audit trail |
| `CalculationCreate` schema | Validates `a`, `b`, `type`; rejects divide-by-zero at schema level |
| `CalculationRead` schema | Returns `id`, `a`, `b`, `type`, `result`, `user_id`, timestamps |
| `CalculationUpdate` schema | Partial update; re-validates divide-by-zero if type is divide |
| Factory pattern | `app/calculations/factory.py` — `Add`, `Subtract`, `Multiply`, `Divide` classes |
| Unit tests | 37 tests — hashing, factory correctness, schema validation |
| Integration tests | 36 tests — real Postgres, DB storage verification, error cases |
| CI/CD maintained | GitHub Actions runs all tests + pushes to Docker Hub on success |

---

## 2. Project Structure Changes

Files **added or modified** in Module 11:

```
module11-user-and-calculator/
│
├── app/
│   ├── __init__.py
│   ├── main.py                # FastAPI entry point
│   ├── database.py            # DB connection & session
│   ├── models/                # SQLAlchemy models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── calculator.py
│   ├── schemas/               # Pydantic schemas
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── calculator.py
│   ├── crud/                  # CRUD operations
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── calculator.py
│   └── api/                   # API routes
│       ├── __init__.py
│       ├── user.py
│       └── calculator.py
│
├── tests/
│   ├── __init__.py
│   ├── test_unit/             # Unit tests
│   ├── test_integration/      # Integration tests
│   └── e2e/                   # End-to-end tests (Playwright)
│
├── .github/
│   └── workflows/
│       └── ci.yml             # GitHub Actions CI pipeline
│
├── Dockerfile                 # Docker image definition
├── docker-compose.yml         # Multi-container setup
├── requirements.txt           # Python dependencies
├── README.md                  # Project documentation
├── .env                       # Environment variables
├── .gitignore
└── pytest.ini                 # Pytest configuration
```
---

## 3. How to Run Locally

### Option A — Docker Compose (easiest)

```bash
# 1. Clone the repo
git clone https://github.com/tl392/module11-user-and-calculator.git
cd fastapi-project

# 2. Wipe old containers and volumes
docker compose down -v

# 3. Build and start
docker compose up --build

# 4. Confirm startup in the logs:
#    db-1   | database system is ready to accept connections
#    app-1  | [startup] Tables created / verified OK.

# 5. Open the app
open http://localhost:8000
open http://localhost:8000/docs
```

### Option B — Local Python + virtual environment

```bash
# 1. Create and activate virtual environment
python -m venv venv
source venv/bin/activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Start PostgreSQL via Docker
docker run -d \
  --name uservault-db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=userdb \
  -p 5432:5432 \
  postgres:latest

# 4. Set the database URL
export DATABASE_URL="postgresql://postgres:postgres@localhost:5432/userdb"
# Windows PowerShell: $env:DATABASE_URL = "postgresql://postgres:postgres@localhost:5432/userdb"

# 5. Start the app
uvicorn main:app --reload

# 6. Open
open http://localhost:8000
```

---

## 4. Running the Tests

### Unit tests — no database required

Tests covered: bcrypt hashing, all four factory operations, divide-by-zero in factory, `CalculationCreate` schema validation, `CalculationUpdate` schema validation, `UserCreate` schema validation.

### Integration tests — requires PostgreSQL

Tests covered: DB stores correct result for all four operations, result is computed by factory (not hardcoded), invalid type rejected, divide-by-zero rejected, unknown user rejected, cascade delete removes calculations.


```bash
pytest -v
```

---
## 5. API Endpoints
 
All routes are defined in `main.py`. Each operation accepts two floats and returns the result.
 
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | HTML calculator UI (Jinja2 template) |
| `POST` | `/add` | Add `a + b` |
| `POST` | `/subtract` | Subtract `a - b` |
| `POST` | `/multiply` | Multiply `a × b` |
| `POST` | `/divide` | Divide `a ÷ b` (zero → 400) |
 
**Request body (`OperationRequest`):**
```json
{ "a": 10.0, "b": 4.0 }
```
 
**Success response (`OperationResponse`):**
```json
{ "result": 2.5 }
```
 
**Error response (`ErrorResponse`):**
```json
{ "error": "Cannot divide by zero" }
```
 
### curl examples
 
```bash
# Add
curl -X POST http://localhost:8000/add \
  -H "Content-Type: application/json" \
  -d '{"a": 15, "b": 7}'
# → {"result": 22.0}
 
# Multiply
curl -X POST http://localhost:8000/multiply \
  -H "Content-Type: application/json" \
  -d '{"a": 6, "b": 7}'
# → {"result": 42.0}
 
# Divide
curl -X POST http://localhost:8000/divide \
  -H "Content-Type: application/json" \
  -d '{"a": 10, "b": 4}'
# → {"result": 2.5}
 
# Divide by zero → 400
curl -X POST http://localhost:8000/divide \
  -H "Content-Type: application/json" \
  -d '{"a": 10, "b": 0}'
# → {"error": "Cannot divide by zero"}
 
# Invalid input → 400
curl -X POST http://localhost:8000/add \
  -H "Content-Type: application/json" \
  -d '{"a": "hello", "b": 5}'
# → {"error": "a: value is not a valid number"}
```
 
Try everything interactively at **http://localhost:8000/docs** (Swagger UI).
 
---
 
## 6. How the Calculation Model Works
 
**File:** `app/models/calculation.py`
 
This project uses **SQLAlchemy single-table polymorphic inheritance**. All four calculation types are stored in one database table. The `type` column (the discriminator) tells SQLAlchemy which Python subclass to return when a row is queried.
 
### Class hierarchy
 
```
Calculation  (SQLAlchemy Base — single table, __tablename__ = "calculations")
│   columns: id, user_id (FK), type (discriminator), inputs, created_at
│
├── Addition        →  get_result():  sum(inputs)
├── Subtraction     →  get_result():  inputs[0] - sum(inputs[1:])
├── Multiplication  →  get_result():  product of all inputs
└── Division        →  get_result():  inputs[0] ÷ inputs[1] ÷ ...
                                      raises ValueError if any divisor is 0
```
 
### Factory method
 
```python
# Correct subclass created automatically — no if/elif needed in calling code
calc = Calculation.create('addition', user_id=1, inputs=[3, 4, 5])
assert isinstance(calc, Addition)
assert calc.get_result() == 12
 
calc2 = Calculation.create('division', user_id=1, inputs=[100, 4])
assert isinstance(calc2, Division)
assert calc2.get_result() == 25.0
```
 
### Polymorphic queries
 
```python
# SQLAlchemy returns each row as the correct subclass
calculations = session.query(Calculation).all()
# → [<Addition>, <Division>, <Multiplication>, ...]
 
for c in calculations:
    print(type(c).__name__, "→", c.get_result())
```
 
### Key design decisions
 
- **Single table** — no joins needed; all calculations retrieved in one query
- **`type` discriminator** — set automatically by SQLAlchemy, never set manually
- **`get_result()` computes live** — no stored result column; always fresh and consistent
- **`user_id` FK** — every calculation belongs to a user; relationship enables `user.calculations`
 
---
 
## 7. How the Pydantic Schemas Work
 
**File:** `app/schemas/calculation.py`
 
### `CalculationType` enum
 
```python
class CalculationType(str, Enum):
    ADDITION       = "addition"
    SUBTRACTION    = "subtraction"
    MULTIPLICATION = "multiplication"
    DIVISION       = "division"
```
 
Pydantic automatically rejects any string not in this enum — no manual validation needed.
 
### `CalculationCreate`
 
Validates data before anything touches the database:
- `type` must be a valid `CalculationType`
- `inputs` must be a list with at least 2 numbers
- `user_id` required
- Division-by-zero checked with **LBYL** before any computation
 
```python
# Cross-field LBYL validator:
@model_validator(mode="after")
def check_division_by_zero(self):
    if self.type == CalculationType.DIVISION:
        if any(x == 0 for x in self.inputs[1:]):
            raise ValueError("Cannot divide by zero")
    return self
```
 
### `CalculationResponse`
 
Returned to clients — includes the computed result:
```json
{
  "id": 1,
  "user_id": 3,
  "type": "addition",
  "inputs": [10, 20, 30],
  "result": 60.0,
  "created_at": "2025-01-15T10:00:00Z"
}
```
 
### `CalculationUpdate`
 
All fields optional — supports partial updates. The division-by-zero guard runs again if `type` is `division`.
 
### LBYL vs EAFP — both used in this project
 
| Style | Where | Code |
|-------|-------|------|
| **LBYL** (Look Before You Leap) | Pydantic `CalculationCreate` validator | `if any(x == 0 for x in inputs[1:]): raise ValueError(...)` |
| **EAFP** (Easier to Ask Forgiveness) | `Division.get_result()` in the model | Attempt division, catch `ZeroDivisionError` internally |
 
---
 
## 8. Factory Pattern Explained
 
`Calculation.create()` is a class-level factory method. Callers never need to know which subclass to use:
 
```
Client code:
  Calculation.create('division', user_id=1, inputs=[20, 4])
          │
          ▼
  Factory resolves 'division' → Division class
          │
          ▼
  Returns: Division(user_id=1, inputs=[20, 4])
          │
          ▼
  .get_result() → 5.0
```
 
**Why the factory pattern?**
 
| Without Factory | With Factory |
|----------------|-------------|
| `if type == "addition": obj = Addition(...)` scattered everywhere | All creation in one place: `Calculation.create()` |
| Adding `Modulo` type means editing every `if/elif` chain | Adding `Modulo` = one new subclass + one line in the factory |
| Hard to test individual operation logic | Each subclass is independently unit-testable |
 
---
 
## 9. CI/CD Pipeline
 
**File:** `.github/workflows/test.yml`
 
The pipeline runs automatically on every push to `main`.
 
```
Push to main branch
         │
         ▼
   Job: test (all tests pass)
         │
         ▼
  Job: deploy (main branch only)
```
 
### Adding required GitHub secrets
 
1. GitHub repo → **Settings → Secrets and variables → Actions → New repository secret**
 
| Secret Name | Where to get it |
|-------------|----------------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub → Account Settings → Personal Access Tokens → Generate New Token (Read/Write/Delete) |
 
### Triggering the pipeline
 
```bash
git add .
git commit -m "describe your change"
git push origin main
# GitHub repo → Actions tab → watch the CI/CD run
```
 
A green ✅ on both `test` and `deploy` jobs means your image is live on Docker Hub.
 
---
 
## 10. Docker Hub
 
**Repository:** https://hub.docker.com/r/ltaravindh392/module11-user-and-calculator  
*(Update this URL to match your actual Docker Hub account)*
 
The pipeline pushes two tags per run:
- `latest` — always the newest build
- `<git-sha>` — pinned to the exact commit (for rollbacks)
 
**Pull and run the published image:**
 
```bash
# Pull
docker pull ltaravindh392/module11-user-and-calculator:latest
 
# Run (needs a Postgres instance)
docker run -d \
  -p 8000:8000 \
  -e DATABASE_URL="postgresql://postgres:postgres@host.docker.internal:5432/userdb" \
  ltaravindh392/module11-user-and-calculator:latest
 
open http://localhost:8000
```
 
> `host.docker.internal` works on Mac and Windows to reach the host machine.  
> On Linux, use `--network host` or replace with your machine's IP.
 
---