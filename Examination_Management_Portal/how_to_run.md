## HOW TO RUN THE APP

Follow these step-by-step instructions to run the Examination Management Portal locally.

---

### Step 1 — Navigate into Project Folder
```bash
cd emp
```

---

### Step 2 — Activate Virtual Environment
```bash
# On Linux / macOS / WSL:
source venv/bin/activate

# On Windows (cmd or PowerShell):
venv\Scripts\activate
```

---

### Step 3 — Install Dependencies
```bash
pip install -r requirements.txt
```

---

### Step 4 — (Optional) Seed Demo Data
To test with pre-populated courses, exams, examiners, and students:
```bash
python seed_demo.py
```

---

### Step 5 — Start Flask Development Server
```bash
python app.py
```

The terminal will display:
```
 * Running on http://127.0.0.1:5000
 * Press CTRL+C to quit
```

---

### Step 6 — Access the Portal in Browser
Open your browser and navigate to:
```
http://127.0.0.1:5000
```

---

### Default Credentials:

| Role | Email | Password | Notes |
|---|---|---|---|
| **Admin** | `admin@emp.com` | `admin123` | Created automatically by `app.py` on startup |
| **Examiner** | `examiner@emp.com` | `password` | Populated by `seed_demo.py` (Approved) |
| **Student** | `student@emp.com` | `password` | Populated by `seed_demo.py` (Active) |
