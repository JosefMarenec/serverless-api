# Library Management API

Serverless Library Management API built with AWS Lambda, API Gateway, and PostgreSQL (RDS). Full CRUD for books, members, and loans — with overdue fee calculation, availability tracking via SQL triggers, and Terraform-provisioned infrastructure.

---

## Architecture

```
Client → API Gateway → Lambda (Python 3.11) → RDS PostgreSQL
                                ↑
                       Secrets Manager (credentials)
                       CloudWatch (logs + alarms)
```

## Endpoints

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/health` | DB connectivity check |
| GET | `/books` | List books (filter by genre, author, status, search) |
| POST | `/books` | Add a new book |
| GET | `/books/{id}` | Get book by ID |
| PUT | `/books/{id}` | Update book details |
| DELETE | `/books/{id}` | Retire a book (soft delete) |
| GET | `/members` | List members (filter by status, search) |
| POST | `/members` | Register a new member |
| GET | `/members/{id}` | Get member by ID |
| PUT | `/members/{id}` | Update member details |
| GET | `/members/{id}/loans` | List all loans for a member |
| GET | `/loans` | List loans (filter by status, member, overdue) |
| POST | `/loans` | Check out a book |
| GET | `/loans/{id}` | Get loan by ID |
| PUT | `/loans/{id}` | Return a book / update loan status |

## SQL Features

- **Triggers** — auto-sync `available_copies` on loan create/return; auto-update `updated_at`
- **Indexes** — partial indexes for available books, overdue loans, active members
- **Views** — `v_active_loans`, `v_overdue_loans`, `v_popular_books`, `v_member_summary`
- **ENUMs** — typed statuses for books, members, loans, and reservations
- **Generated columns** — computed totals stored at DB level

## Project Structure

```
├── api/
│   ├── handlers/
│   │   ├── books.py       # CRUD /books
│   │   ├── members.py     # CRUD /members + /members/{id}/loans
│   │   ├── loans.py       # Checkout, return, manage /loans
│   │   └── health.py      # GET /health
│   └── utils/
│       ├── db.py          # Connection pooling + Secrets Manager
│       ├── response.py    # Standardised JSON responses
│       └── pagination.py  # Query param helpers
├── migrations/
│   ├── V1__create_api_tables.sql
│   ├── V2__add_indexes.sql
│   └── V3__add_views.sql
├── terraform/
│   ├── main.tf
│   ├── lambda.tf          # 4 Lambda functions
│   ├── api_gateway.tf     # REST API + routes
│   ├── iam.tf             # Least-privilege IAM roles
│   ├── variables.tf
│   └── outputs.tf
└── tests/                 # 42 unit tests, fully mocked
```

## Getting Started

### Prerequisites
- Python 3.11+
- Terraform 1.6+
- AWS CLI configured
- PostgreSQL (RDS or local)

### Install dependencies
```bash
make install
```

### Run tests
```bash
make test
```

### Run migrations
```bash
export RDS_HOST=your-host DB_NAME=library DB_USER=postgres
make migrate
```

### Deploy to AWS
```bash
cp terraform/terraform.tfvars.example terraform/terraform.tfvars
# Fill in your RDS host, subnet IDs, etc.
make deploy-dev
```

## Checkout Flow

```
POST /loans
{
  "book_id": "<uuid>",
  "member_id": "<uuid>",
  "due_date": "2026-04-14T00:00:00Z"  // optional
}
```

Validates:
- Member is active and under their loan limit
- Book has available copies
- Decrements `available_copies` atomically via SQL trigger

## Return Flow

```
PUT /loans/{id}
{ "action": "return" }
```

Automatically calculates overdue fee at **$0.25/day** past the due date.
