# Cloud Workspace

> A lightweight cloud workspace manager for defining, inspecting, launching, and estimating the cost of containerized development environments.

Cloud Workspace is a Flask web application built around a simple idea: **describe a development environment once, then manage that environment through a browser-based workspace dashboard**.

A workspace is represented by a Docker Compose YAML file and can optionally include a Conda `environment.yml`. The application stores workspace metadata, lets authenticated users inspect services and resource requirements, can hand execution off to a remote Docker runner, and provides a side-by-side cost estimate across AWS, Azure, Google Cloud, and DigitalOcean.

---

## What it does

Cloud Workspace combines four pieces into one web application:

* **Workspace management** — create, inspect, run, stop, refresh, and delete workspaces.
* **Environment definition** — upload a Docker Compose file and optionally a Conda environment file.
* **Remote execution** — send a workspace definition to an external runner through authenticated HTTP endpoints.
* **Cloud cost analysis** — parse Compose resource requirements and estimate monthly infrastructure costs across multiple providers.

The application is intentionally split conceptually into a **web control plane** and an optional **remote execution layer**.

```text
┌──────────────────────┐
│      Browser UI      │
│ Login / Dashboard    │
│ Workspace / Costs    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Flask Application  │
│ Auth + DB + Uploads  │
│ Compose Analysis     │
│ Workspace Lifecycle  │
└───────┬───────┬──────┘
        │       │
        │       └──────────────────────┐
        │                              │
        ▼                              ▼
┌───────────────┐             ┌──────────────────┐
│ SQLite /      │             │ Optional Remote  │
│ PostgreSQL    │             │ Docker Runner    │
└───────────────┘             │ /run-docker      │
                              │ /stop-docker      │
                              └─────────┬────────┘
                                        │
                                        ▼
                              ┌──────────────────┐
                              │ Docker / Cloud   │
                              │ Worker Host      │
                              └──────────────────┘
```

---

## Core workflow

### 1. Create an account

Users can register and sign in through the built-in Flask-Login authentication flow.

Passwords are stored as Werkzeug password hashes rather than plaintext.

### 2. Create a workspace

From the dashboard, create a workspace with:

* a workspace name
* a Docker Compose `.yml` / `.yaml` file
* an optional Conda `environment.yml`

Uploaded files are renamed with generated UUID prefixes before being stored, reducing filename collisions and avoiding direct trust of user-supplied filenames.

### 3. Inspect the workspace

The dashboard reads the Compose definition and exposes the services declared in the file.

For each service, Cloud Workspace can identify:

* service name
* container image
* build configuration
* requested CPU and memory limits when declared

### 4. Run the workspace

When a remote runner is configured, the application sends the uploaded Compose YAML and optional environment file to:

```text
POST {RUNNER_URL}/run-docker
```

The request includes:

```text
X-Runner-Token: {RUNNER_TOKEN}
```

A successful response updates the workspace status to `running`.

### 5. Stop the workspace

The web application can ask the remote runner to stop a workspace through:

```text
POST {RUNNER_URL}/stop-docker
```

The runner is given the workspace project identifier so it can map the request to the corresponding Docker workload.

### 6. Compare infrastructure cost

Cloud Workspace parses the Compose file, extracts resource requirements, and produces estimated monthly costs for:

| Provider     | Example instance families |
| ------------ | ------------------------- |
| AWS          | T3                        |
| Azure        | B-series                  |
| Google Cloud | E2                        |
| DigitalOcean | Basic Droplets            |

The UI shows estimated compute cost, storage cost, total monthly cost, instance capacity, and expandable pros/cons.

> **Important:** these are application-level estimates based on the pricing constants currently defined in `app.py`. They are not live provider quotes.

---

## Features

### Authentication

* User registration
* Login / logout
* Session-based authentication with Flask-Login
* Per-user workspace ownership
* Authorization checks before workspace operations

### Workspace management

* Create workspaces
* Upload Compose files
* Upload optional Conda environment definitions
* View workspace status
* Run / stop through the remote runner
* Refresh workspace status
* Delete workspaces and associated files

### Compose analysis

The application uses PyYAML to inspect Compose definitions without requiring Docker just to display their structure.

It can derive:

* total CPU requirement
* total memory requirement
* number of services
* per-service CPU / memory
* a simple storage estimate

### Cloud cost comparison

The current implementation contains estimates for:

* AWS
* Azure
* GCP
* DigitalOcean

For each provider, the application selects an available instance configuration that satisfies the parsed CPU and memory requirements, then estimates:

```text
monthly compute
+ estimated storage
= estimated monthly total
```

### Browser UI

The application uses server-rendered Jinja templates with Tailwind utility classes and Font Awesome icons.

The UI includes:

* login / registration pages
* responsive workspace dashboard
* workspace creation form
* workspace status indicators
* delete confirmation modal
* provider cost comparison cards
* expandable provider pros / cons
* responsive desktop table and mobile-friendly workspace cards

---

## Tech stack

### Backend

* **Python**
* **Flask**
* **Flask-Login**
* **Flask-SQLAlchemy**
* **Werkzeug**
* **PyYAML**
* **Requests**
* **Gunicorn**

### Data layer

Development defaults to:

* SQLite

Production can use:

* PostgreSQL through `DATABASE_URL`

### Frontend

* Jinja2 templates
* Tailwind CSS utilities
* Font Awesome
* Small inline JavaScript for interactions and transitions

### Deployment

The repository includes a Gunicorn `Procfile` and a Python runtime declaration, making it suitable for platforms such as Render.

---

## Project structure

```text
Cloud-Workspace/
│
├── app.py
├── requirements.txt
├── environment.yml
├── runtime.txt
├── Procfile
│
├── templates/
│   ├── base.html
│   ├── login.html
│   ├── dashboard.html
│   ├── workspace.html
│   └── cost_comparison.html
│
└── sample-workspace/
    ├── docker.yml
    ├── environment.yml
    ├── test2.yml
    └── text.yml
```

### `app.py`

The main Flask application.

It contains:

* configuration
* SQLAlchemy models
* authentication routes
* workspace routes
* file upload handling
* Compose parsing
* remote runner integration
* cost estimation logic

### `templates/`

Jinja2 server-rendered pages for authentication, workspace management, dashboard views, and cost analysis.

### `sample-workspace/`

Small example workspace definitions that can be used to test the upload and analysis flow.

---

## Data model

The application currently revolves around two primary database models.

### User

Stores:

```text
id
username
password_hash
```

A user can own multiple workspaces.

### Workspace

Stores:

```text
id
name
yaml_filename
env_filename
created_at
status
user_id
```

This keeps the database focused on **workspace metadata**, while the uploaded files live in the configured upload directory.

---

## Configuration

Cloud Workspace reads configuration from environment variables.

### Required / recommended

```env
SECRET_KEY=replace-me
```

### Database

```env
DATABASE_URL=postgresql://user:password@host:5432/database
```

When `DATABASE_URL` is not provided, the application falls back to a local SQLite database.

### Uploads

```env
UPLOAD_FOLDER=/tmp/uploads
MAX_CONTENT_LENGTH=5242880
```

The default upload size limit is **5 MB**.

### Remote runner

To enable actual remote workspace execution:

```env
RUNNER_URL=https://your-runner.example.com
RUNNER_TOKEN=your-shared-secret
```

The Flask application expects the runner to expose:

```text
POST /run-docker
POST /stop-docker
```

The runner implementation itself is **not included in this repository**.

---

## Local development

### 1. Clone the repository

```bash
git clone https://github.com/MayenkJoshi37/Cloud-Workspace.git
cd Cloud-Workspace
```

### 2. Create a virtual environment

```bash
python -m venv .venv
```

Activate it on Windows:

```powershell
.venv\Scripts\activate
```

Activate it on macOS / Linux:

```bash
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure environment variables

At minimum, set a production-safe secret key:

```bash
SECRET_KEY=change-this
```

For remote execution, also set:

```text
RUNNER_URL
RUNNER_TOKEN
```

### 5. Start the application

```bash
python app.py
```

The Flask development server binds to:

```text
0.0.0.0:$PORT
```

and defaults to port `5000` when `PORT` is not supplied.

Open:

```text
http://localhost:5000
```

---

## Using the sample workspace

The repository includes a small example based around code-server:

```text
sample-workspace/
├── docker.yml
└── environment.yml
```

The sample Compose file defines a `codeserver` service with:

* a code-server image
* password-based authentication
* port `8443`
* a mounted project directory

The sample environment file contains Python / data-science dependencies such as:

* NumPy
* pandas

This makes the sample useful for testing the basic **workspace definition → upload → inspection** flow.

---

## Deployment

The repository contains:

```text
Procfile
runtime.txt
requirements.txt
```

The Procfile starts Gunicorn with four workers:

```bash
gunicorn -w 4 -b 0.0.0.0:$PORT app:app
```

The current runtime declaration is Python 3.11.9.

### Production architecture

A practical deployment looks like:

```text
                Internet
                    │
                    ▼
        ┌──────────────────────┐
        │ Flask / Gunicorn     │
        │ Cloud Workspace UI   │
        └──────────┬───────────┘
                   │
          HTTPS + runner token
                   │
                   ▼
        ┌──────────────────────┐
        │ Remote Runner        │
        │ Docker host / EC2    │
        └──────────┬───────────┘
                   │
                   ▼
              Containers
```

This distinction matters because the Flask web service is **not itself the complete cloud container runtime**.

---

## Important implementation notes

### Docker execution is externalized

The current `app.py` is written with cloud-hosted web deployment in mind. The main `Run` path sends workspace files to the configured remote runner instead of assuming that the Flask process itself has access to a Docker daemon.

There is still legacy/local Docker-related code in the repository, particularly around workspace deletion and Compose build helpers. Treat those paths as local-runtime support rather than evidence that the deployed web process can always control Docker directly.

### Upload storage is ephemeral by default

The application defaults uploads to:

```text
/tmp/uploads
```

That is convenient for a development or ephemeral deployment, but it should **not** be treated as durable object storage.

For a persistent production deployment, move uploaded workspace definitions to durable storage such as object storage or a persistent volume.

### Database initialization

The current application calls:

```python
db.create_all()
```

during application startup.

That is convenient for a small prototype, but a larger production deployment should use a migration system such as Flask-Migrate / Alembic so schema changes can be tracked safely.

### Cost estimates are static

The provider pricing table is embedded in `app.py`.

That means the cost comparison is an **estimation feature**, not a live pricing API.

Prices, instance availability, regions, network costs, discounts, reserved pricing, spot pricing, and service-specific charges are not dynamically queried.

### Security hardening

Before exposing this application to untrusted users, production deployments should additionally consider:

* a strong `SECRET_KEY`
* HTTPS
* CSRF protection
* stricter upload validation
* secure cookie configuration
* rate limiting
* secret rotation
* durable and isolated workspace storage
* stronger remote-runner authentication
* authorization and audit logging
* avoiding credentials inside sample Compose files

---

## Example workspace definition

A minimal workspace can look like:

```yaml
services:
  app:
    image: python:3.11-slim
    ports:
      - "8000:8000"
    command: python -m http.server 8000
```

Upload the file from the **Create Workspace** page, optionally add an `environment.yml`, and the application can inspect the service definition and include it in the workspace dashboard.

---

## Why the project is interesting

Cloud Workspace sits at the intersection of several useful engineering concepts:

* web application development
* authentication and authorization
* containerized environments
* infrastructure abstraction
* YAML-based configuration
* remote execution
* cloud resource estimation
* multi-provider comparison
* server-rendered UI

The interesting architectural problem is not just “run Docker from Flask”.

It is the separation between:

```text
What the user wants
        ↓
Workspace specification
        ↓
Control plane
        ↓
Execution backend
        ↓
Actual infrastructure
```

That separation makes it possible to move the execution layer independently of the user-facing application.

---

## Current limitations

Cloud Workspace is a prototype-oriented implementation and has several deliberate limitations:

* No live cloud-provider pricing API
* No built-in remote runner implementation
* No persistent object storage layer
* No full container orchestration system
* No streaming container logs
* No terminal/session proxy
* No workspace networking management
* No automated cleanup scheduler
* No infrastructure-as-code generation
* No formal database migration workflow
* No production-grade CSRF / rate-limiting layer

These are natural next steps if the project evolves toward a production multi-tenant platform.

---

## Potential next steps

A natural evolution of the project would be:

1. **Dedicated runner service**

   * Isolate Docker execution in a separate service.
   * Add job IDs, execution states, logs, and health checks.

2. **Persistent workspace storage**

   * Store Compose and environment definitions in object storage.
   * Keep ephemeral compute separate from durable workspace metadata.

3. **Live infrastructure telemetry**

   * CPU / RAM / container state
   * logs
   * health checks
   * execution history

4. **Real cloud pricing**

   * Replace hard-coded estimates with provider pricing APIs.
   * Add region selection and usage profiles.

5. **Workspace lifecycle automation**

   * idle shutdown
   * TTLs
   * scheduled cleanup
   * automatic resource reclamation

6. **Stronger production security**

   * CSRF protection
   * secrets manager integration
   * isolated execution
   * audit logs
   * stricter tenant boundaries

---

## License

No explicit license is currently defined in the repository.

Until a license is added, the code should be treated as **all rights reserved by default** rather than automatically assumed to be open-source licensed.
