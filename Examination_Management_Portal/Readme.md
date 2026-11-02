# Examination Management Portal (EMP) — Complete Project Roadmap
### IITM BS | MAD-1 Project Guide | Flask + Jinja2 + SQLite + Bootstrap 5

---

Welcome to the comprehensive modular guide for the **Examination Management Portal (EMP)** project.
Each milestone contains both the **conceptual roadmap/architectural guidelines** and the **complete copy-paste production code** for that phase.

---

## 🗺️ Navigation Map (Milestones)

Follow the guides in chronological order:

1. **[Milestone 0: Setup & Environment](milestone_0_setup.md)**
   - Initial environment setup, Python packages, project directory tree, virtual environment, and learning plan.
2. **[Milestone 1: Design ER Diagram & Architecture](milestone_1_er_diagram.md)**
   - User roles, page map, database entities, column mappings, Mermaid ER diagram, and wireframes.
3. **[Milestone 2: Database Models & App Skeleton](milestone_2_db_models.md)**
   - Configuration (`config.py`), shared extensions (`extensions.py`), utilities (`utils.py`), all 9 SQLAlchemy models (`models.py`), app factory & programmatic admin seeder (`app.py`).
4. **[Milestone 3: Authentication & Role-Based Access](milestone_3_auth.md)**
   - Custom RBAC decorators (`decorators.py`), auth blueprint (`routes/auth.py`), master layout (`templates/base.html`), login and registration templates.
5. **[Milestone 4: Admin Dashboard & Management](milestone_4_admin.md)**
   - Admin routes (`routes/admin.py`), course CRUD, exam scheduling windows, examiner approval/assignment, rubrics, booking oversight, rescheduling, search, and audit history (all 16 admin templates).
6. **[Milestone 5: Examiner Dashboard & Slot Management](milestone_5_examiner.md)**
   - Examiner routes (`routes/examiner.py`), slot creation & validation, attendee rosters, rubric evaluations, IDOR protection, and all 6 examiner templates.
7. **[Milestone 6: Student Dashboard & Exam Booking](milestone_6_student.md)**
   - Student routes (`routes/student.py`), pure GET search/filter, the 6 critical booking rules, atomic capacity updates, cancellation, and student templates.
8. **[Milestone 7: Booking History & Evaluation Tracking](milestone_7_history.md)**
   - Student past booking history, published results view, admin audit trail, and full lifecycle verification.
9. **[Milestone 8: GitHub Setup & Collaboration](milestone_8_github.md)**
   - Initializing Git repo, private GitHub setup, adding collaborator `MADI-cs2003`, and commit workflows.
10. **[Milestone 9: Optional Features & APIs](milestone_9_optional.md)**
    - Demo data seeder (`seed_demo.py`), RESTful JSON API (`routes/api.py`), Chart.js analytics, and CSV export.
11. **[Milestone 10: Comprehensive Testing Checklist](milestone_10_testing.md)**
    - Exhaustive manual test checklists covering auth, admin, examiner, student, and edge cases.
12. **[Milestone 11: Final Submission & Viva Preparation](milestone_11_submission.md)**
    - Directory checksum calculation (`check.py`), clean ZIP creation, 5-page report guidelines, video demo structure, and viva voce live-coding scenarios.

---

## 🛠️ General Reference Docs

* **[Final Folder Structure](final_folder_structure.md)** — Complete ASCII tree of the final EMP application.
* **[How to Run the App](how_to_run.md)** — Step-by-step local execution instructions and default credentials.
* **[Common Errors and Fixes](common_errors.md)** — Quick solutions for frequent Flask, SQLite, and Jinja2 errors.
* **[Milestone Commit Messages](commit_messages.md)** — Standardized Git commit messages for every phase.
