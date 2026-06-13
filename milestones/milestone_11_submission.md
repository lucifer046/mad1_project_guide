## PHASE 11: FINAL SUBMISSION (Day 34-38)

### Step 11.1 — Generate Checksum

```bash
pip install checksumdir

# Create check.py in folder ABOVE your project
# check.py
import checksumdir
hash = checksumdir.dirhash("trekking_app")  # Replace with your folder name
print("Directory Checksum:", hash)

python check.py  # Note down this hash
```

### Step 11.2 — Create ZIP File

```bash
# On Windows:
# Right-click your project folder → Send to → Compressed (zipped) folder

# On Mac/Linux:
zip -r trekking_app.zip trekking_app/

# Structure should be:
# trekking_app.zip
# └── trekking_app/
#   ├── app.py
#   ├── models.py
#   └── ...
```

### Step 11.3 — Write Project Report (3-5 Pages)

Your report must include:
1. **Student Details** — Name, Roll No, Email
2. **Problem Statement** — Brief summary of the project
3. **Your Approach** — How you structured the solution
4. **AI/LLM Declaration** — How much % AI was used (mandatory even if 0%)
5. **Frameworks Used** — Flask, Jinja2, SQLite, Bootstrap 5
6. **ER Diagram** — Screenshot from draw.io
7. **API Endpoints** — Table of your routes (if implemented)
8. **Google Drive Video Link** — 5-10 min demo video

### Step 11.4 — Record Demo Video (5-10 minutes)

Cover in this order:
1. Intro (30 sec) — "I built a Trekking Management App using Flask..."
2. Approach (30 sec) — "I started with ER design, then models, then auth..."
3. Demo (90 sec each):
   - Admin flow: login → add trek → approve staff → assign staff
   - Staff flow: login → view trek → update slots/status
   - User flow: register → login → browse → book → view history
4. Any extra features (30 sec)

Upload to Google Drive → Share → "Anyone with link" → Paste link in report.

---