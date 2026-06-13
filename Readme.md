# MAD-1 Trekking Management App — Complete Roadmap
### IITM BS | May 2026 Term | Flask + Jinja2 + SQLite

---

This is the primary index and index instructions for the MAD-1 Trekking Management App project guide. The roadmap is split into milestone-wise files for better readability and progression.

---

## 🗺️ Navigation Map (Milestones)

Please follow the guides in chronological order:

1. **[Milestone 0: Setup](milestones/milestone_0_setup.md)**
   - Initial environment setup, Python packages, project directory structure.
2. **[Milestone 1: Design ER Diagram](milestones/milestone_1_er_diagram.md)**
   - Database entities (User, Trek, Booking, StaffProfile), column mappings, and relationships.
3. **[Milestone 2: Database Models](milestones/milestone_2_db_models.md)**
   - Configuration setup, implementing SQLAlchemy models (`models.py`), basic `app.py`, testing DB creation.
4. **[Milestone 3: Authentication](milestones/milestone_3_auth.md)**
   - Templates for login/register, auth routes, access decorators (`login_required`, `admin_required`, `staff_required`).
5. **[Milestone 4: Admin Dashboard](milestones/milestone_4_admin.md)**
   - Admin routes, statistics, CRUD operations for trek management, staff approval/blacklisting.
6. **[Milestone 5: Trek Staff Dashboard](milestones/milestone_5_staff.md)**
   - Staff routes, slots & status updates, list of registered trekkers.
7. **[Milestone 6: User Dashboard & Booking](milestones/milestone_6_user.md)**
   - Browsing open treks, filters, booking logic, cancellation routes.
8. **[Milestone 7: Booking History & Status Tracking](milestones/milestone_7_history.md)**
   - Checking past bookings, status badges.
9. **[Milestone 8: GitHub Setup & Remote Push](milestones/milestone_8_github.md)**
   - Setting up repository, adding collaborators, commit workflows.
10. **[Milestone 9: Optional Features](milestones/milestone_9_optional.md)**
    - Flask-Login integration, Chart.js implementation, JSON API endpoints.
11. **[Milestone 10: Testing Checklist](milestones/milestone_10_testing.md)**
    - Detailed checkboxes to manually verify auth, admin, staff, and user flows.
12. **[Milestone 11: Final Submission](milestones/milestone_11_submission.md)**
    - Generating checksums, directory structuring, zip guidelines, report formatting, video recording rules.

---

## 🛠️ General Reference Docs
- **[Final Folder Structure](milestones/final_folder_structure.md)**
- **[How to Run the App](milestones/how_to_run.md)**
- **[Common Errors and Fixes](milestones/common_errors.md)**
- **[Milestone Commit Messages](milestones/commit_messages.md)**

---

## RESOURCES (Watch Before Starting Each Section)

### Hindi Playlist — Flask Full Course
- **[Corey Schafer - Flask Series (English, Best Quality)](https://www.youtube.com/playlist?list=PL-osiE80TeTs4UjLw5MM6OjgkjFeUxCYH)**
- **[Hindi Flask Tutorial by CodeWithHarry](https://www.youtube.com/watch?v=oA8brF3w5XQ)** — Full Flask in Hindi
- **[Python + Flask Hindi by Apna College](https://www.youtube.com/watch?v=Z1RJmh_OqeA)**
- **[SQLite + Python Hindi](https://www.youtube.com/watch?v=byHcYRpMgI4)**
- **[Jinja2 Templating Hindi](https://www.youtube.com/watch?v=mqhxxeeTbu0)**
- **[Bootstrap 5 Hindi by CodeWithHarry](https://www.youtube.com/watch?v=vpAJ0s5S2t0)**
- **[Flask Login System Hindi](https://www.youtube.com/watch?v=71EU8gnZqZQ)**
- **[ER Diagram Kaise Banaye (Hindi)](https://www.youtube.com/watch?v=QpdhBUYk7Kk)**
- **[SQLAlchemy ORM Hindi](https://www.youtube.com/watch?v=cYWiDiIUxQc)**

---

## TIMELINE AT A GLANCE

| Week | Task |
|------|------|
| Week 1 | GitHub Setup + Learn Flask Basics + Design ER Diagram |
| Week 2 | Create DB Models + Authentication (Login/Register) |
| Week 3 | Admin Dashboard + Trek CRUD |
| Week 4 | Trek Staff Dashboard |
| Week 5 | User Dashboard + Booking System |
| Week 6 | Booking History + Status Tracking |
| Week 7 | Optional Features + Testing |
| Week 8 | Video + Report + Final Submission |

**Submission Deadline (Theory Done Before May 2026): 14 July 2026**
**Submission Deadline (Theory in May 2026): 9 August 2026**

---

## TECH STACK (MANDATORY — Don't Use Anything Else)

| Layer | Technology |
|-------|-----------|
| Backend | **Flask** (Python) |
| Frontend | **Jinja2 templates + HTML + CSS + Bootstrap 5** |
| Database | **SQLite** (via SQLAlchemy ORM) |
| Auth | Flask sessions (or Flask-Login — optional) |
| JS |  NOT for core features. OK for Bootstrap tooltips/charts |

---
