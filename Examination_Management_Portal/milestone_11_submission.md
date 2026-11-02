## PHASE 11: FINAL SUBMISSION & VIVA PREPARATION (Day 43–45)

---

## 🧮 Step 11.1 — Generate Directory Checksum

Course submission requires generating an MD5 or SHA256 directory checksum to prove code integrity.

```bash
# Install checksum utility
pip install checksumdir

# Create check.py in the folder ABOVE your project
# check.py:
import checksumdir
hash_val = checksumdir.dirhash("emp")
print("Project Directory Checksum:", hash_val)

python check.py
# Record this checksum hash for your submission portal form!
```

---

## 📦 Step 11.2 — Create Clean ZIP File

> ⚠️ **CRITICAL**: Do NOT include `venv/`, `__pycache__/`, `.db` files, or `.git/` in your submitted ZIP file. Submitting bloated virtual environments or pre-built database files may result in rejection.

```bash
# On Linux / macOS / WSL:
zip -r emp_submission.zip emp/ -x "*/venv/*" "*/__pycache__/*" "*.db" "*.pyc" "*/.git/*"

# On Windows:
# 1. Delete or move venv/ and *.db temporarily, or select only source folders:
#    app.py, config.py, models.py, extensions.py, decorators.py, utils.py,
#    requirements.txt, README.md, routes/, static/, templates/
# 2. Right-click -> Send to -> Compressed (zipped) folder.
```

---

## 📝 Step 11.3 — Project Report Guidelines (3–5 Pages)

Structure your PDF report using the following standard sections:

1. **Cover & Student Details**: Student Name, Roll Number, Degree Program, Project Title (Examination Management Portal).
2. **Problem Statement & Scope**: Brief synopsis of the exam management challenges and how your three-role web system solves them.
3. **Architecture & Tech Stack**: Description of Flask, SQLite, Jinja2, Bootstrap 5, and compliance with the No-JavaScript rule.
4. **AI / LLM Declaration**: Mandatory percentage declaration of AI assistance used.
5. **Entity-Relationship (ER) Diagram**: Clear visual screenshot of your database schema.
6. **Key Workflows & Business Rules**: Description of anti-overbooking mechanisms, window validations, IDOR prevention, and rubric grading.
7. **API Endpoints (if implemented)**: Table of `/api/v1/` routes.
8. **Screenshots & Video Link**: Publicly accessible Google Drive / YouTube link for your demo video.

---

## 🎥 Step 11.4 — Demo Video Walkthrough (5–10 Minutes)

Record a clean screen capture covering:
1. **Introduction (1 min)**: State name, roll number, project overview, and tech stack.
2. **Admin Walkthrough (2 min)**: Show course creation, exam scheduling with date windows, examiner approval, and rubric definition.
3. **Examiner Walkthrough (2 min)**: Show slot creation within window, attendance viewing, and rubric-based grading.
4. **Student Walkthrough (2 min)**: Show exam catalog filtering without JS, booking a slot, cancellation, and viewing results.
5. **Rule Demonstration (1-2 min)**: Demonstrate that double-booking and out-of-window bookings are strictly blocked.

---

## 🎯 Step 11.5 — Viva Voce Preparation

During the live viva, evaluators test your direct understanding of the code by asking you to make small, real-time code modifications:

### Common Live-Coding Requests:
1. *"Change the maximum allowed booking capacity for any slot to 10."*
   - Locate `routes/examiner.py` and modify `if capacity > 10: flash(...)`.
2. *"Add a new field `room_number` to the Slot model."*
   - Add `room_number = db.Column(db.String(50))` in `models.py`.
   - Update `slot_form.html` and `routes/examiner.py`.
3. *"Prevent students from cancelling less than 24 hours before the exam."*
   - In `routes/student.py` `booking_cancel`:
     ```python
     if slot.start_time - datetime.now() < timedelta(hours=24):
         flash('Cannot cancel within 24 hours of exam.', 'danger')
     ```
4. *"Explain why `db.session.commit()` is needed after updating an object."*
   - Answer: Changes to model instances occur only in Python's memory. `db.session.commit()` flushes the SQL statements to the SQLite transaction and commits them to the physical database file.
