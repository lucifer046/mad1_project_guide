## PHASE 0: SETUP & ENVIRONMENT (Day 1–2)

> **Goal**: Establish the local development environment, project folder structure, dependencies, and version control foundations for the Examination Management Portal (EMP).

---

## 🧠 Big Picture: What is the Project?

The **Examination Management Portal (EMP)** is a web-based management portal designed to coordinate examinations, scheduling slots, student registrations, examiner evaluations, and results.

### Key Rules & Constraints:
* **Tech Stack**: Python (Flask) + Jinja2 + Bootstrap 5 + SQLite (Flask-SQLAlchemy ORM).
* **Strict Rule**: **No JavaScript for core features** — All interactions (filtering, search, forms, actions) must function using pure HTML GET/POST forms.
* **Strict Rule**: Database schema must be created **programmatically by code** (`db.create_all()`), never with manual GUI tools.
* **Strict Rule**: Default administrator account must be **seeded programmatically** via Python on application startup.
* **Strict Rule**: Bootstrap is the only permitted CSS styling framework.

---

## 🛠️ Step 0.1 — Install Required Software

Install the foundational toolchain:

1. **Python 3.10+** — [python.org/downloads](https://www.python.org/downloads/) (Remember to check **"Add Python to PATH"** on Windows).
2. **VS Code** — [code.visualstudio.com](https://code.visualstudio.com/) (Recommended extensions: *Python*, *Jinja*, *SQLite Viewer*).
3. **Git** — [git-scm.com/downloads](https://git-scm.com/downloads).
4. **GitHub Account** — [github.com](https://github.com).

```bash
# Verify Python & Git installation
python --version   # Must be 3.8+ (recommended 3.10+)
git --version
```

---

## 📂 Step 0.2 — Create Project Folder Structure

Create the project directory tree manually or via command line:

```bash
mkdir emp && cd emp
```

Your project directory will look like this:

```
emp/
├── app.py                  ← Main Flask application factory & entry point
├── config.py               ← App configuration (Secret key, SQLite DB URI)
├── extensions.py           ← Shared extensions (SQLAlchemy db instance)
├── models.py               ← All SQLAlchemy database models
├── decorators.py           ← Custom role-based access decorators
├── utils.py                ← Helper functions (datetime formatting, logging)
├── requirements.txt        ← Python dependencies manifest
├── README.md               ← Project summary
├── .gitignore              ← Files excluded from Git tracking
├── seed_demo.py            ← Optional demo data seeder
│
├── routes/                 ← Modular Flask Blueprints
│   ├── __init__.py
│   ├── auth.py             ← Login, logout, student & examiner registration
│   ├── admin.py            ← Admin management CRUD & controls
│   ├── examiner.py         ← Examiner slots & student evaluations
│   ├── student.py          ← Student exam browsing, booking, & history
│   └── api.py              ← Optional JSON REST API
│
├── static/
│   └── css/
│       └── style.css       ← Custom CSS styling
│
└── templates/              ← Jinja2 HTML templates
    ├── base.html           ← Global layout with navigation & flash messages
    ├── home.html           ← Public landing page
    ├── error.html          ← Friendly error message page
    ├── auth/               ← Login & registration forms
    ├── admin/              ← 14 Admin dashboard & CRUD templates
    ├── examiner/           ← 6 Examiner workflow templates
    └── student/            ← 7 Student booking & results templates
```

---

## 🐍 Step 0.3 — Create Virtual Environment

A virtual environment ensures your project dependencies don't interfere with your system Python packages.

```bash
# In the emp/ project folder:
python -m venv venv

# Activate Virtual Environment:
# On Windows (cmd/PowerShell):
venv\Scripts\activate

# On Linux/WSL/macOS:
source venv/bin/activate

# Notice prompt changes to (venv)
```

---

## 📦 Step 0.4 — Create `requirements.txt`

Create `requirements.txt` in the root folder with the required dependencies:

```text
Flask>=3.0
Flask-SQLAlchemy>=3.1
Flask-Login>=0.6
```

Install the dependencies:

```bash
pip install -r requirements.txt
```

---

## 🙈 Step 0.5 — Create `.gitignore`

Ensure temporary files, databases, and virtual environments are never committed to Git:

```gitignore
venv/
__pycache__/
*.pyc
instance/
*.db
.env
```

---

## 📄 Step 0.6 — Create `README.md`

Create the project README:

```markdown
# Examination Management Portal (EMP)
Flask + SQLite web app where Admin, Examiners and Students manage exam slots, bookings and evaluations.

## Run
pip install -r requirements.txt
python app.py
Open http://127.0.0.1:5000

Admin login: admin@emp.com / admin123
```

---

## 📚 Learning Plan & Resources

Follow these recommended video tutorials while working on this milestone:

### Hindi Tutorials
* **Python Basics**: [CodeWithHarry Python One-Shot](https://www.youtube.com/watch?v=UrsmFxEIp5k)
* **HTML & CSS Basics**: [Thapa Technical HTML/CSS](https://www.youtube.com/watch?v=BSwa_Egnk58)
* **Git & GitHub**: [Apna College Git & GitHub Tutorial](https://www.youtube.com/watch?v=apGV9Kg7ics)
* **Flask Fundamentals**: [Harshit Vashisth Flask Playlist](https://www.youtube.com/playlist?list=PLwgFb6VsUj_nXD_Nss5jD_7_pnbxb_g2R)

### English Tutorials
* **Corey Schafer Flask Series**: [YouTube Playlist](https://www.youtube.com/playlist?list=PL-osiE80TeTs4UjLw5MM6OjgVECeNe8H3)
* **Bootstrap 5 Crash Course**: [Traversy Media Bootstrap 5](https://www.youtube.com/watch?v=4sosXZsdy-s)
