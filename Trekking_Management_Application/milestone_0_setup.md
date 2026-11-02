## PHASE 0: SETUP (Day 1-2)

### Step 0.1 — Install Required Software

```bash
# 1. Install Python (if not installed)
# Download from: https://www.python.org/downloads/

# 2. Check Python version
python --version  # Should be 3.8+

# 3. Install Flask and dependencies
pip install flask flask-sqlalchemy flask-login

# 4. Install VS Code (recommended editor)
# Download from: https://code.visualstudio.com/
```

### Step 0.2 — Create Your Project Folder Structure

Create this folder structure **manually first**, then we'll fill files one by one:

```
trekking_app/
│
├── app.py                  ← Main Flask application file
├── config.py               ← Configuration (secret key, DB path)
├── models.py               ← All database models (tables)
├── requirements.txt        ← List of all pip packages
│
├── static/
│   ├── css/
│   │   └── style.css       ← Your custom CSS
│   └── images/             ← Any images you use
│
└── templates/
    ├── base.html           ← Common layout (navbar, footer)
    ├── index.html          ← Home page
    │
    ├── auth/
    │   ├── login.html
    │   └── register.html
    │
    ├── admin/
    │   ├── dashboard.html
    │   ├── manage_treks.html
    │   ├── manage_users.html
    │   ├── manage_staff.html
    │   └── assign_staff.html
    │
    ├── staff/
    │   ├── dashboard.html
    │   └── manage_trek.html
    │
    └── user/
        ├── dashboard.html
        ├── browse_treks.html
        ├── booking_history.html
        └── profile.html
```

### Step 0.3 — Create requirements.txt

```
flask
flask-sqlalchemy
flask-login
werkzeug
```

---