## FINAL FOLDER STRUCTURE

The complete, clean directory structure of the Examination Management Portal (EMP):

```
emp/
├── app.py                      # Application factory, DB initialization, admin seeding
├── config.py                   # Configuration settings (SECRET_KEY, DB URI)
├── extensions.py               # Shared extensions (SQLAlchemy, LoginManager)
├── models.py                   # Complete SQLAlchemy models
├── decorators.py               # Role-based access control decorators
├── utils.py                    # Helper functions (datetime parsing, logging)
├── seed_demo.py                # Optional: populated demo data seeder
├── requirements.txt            # Dependency manifest
├── README.md                   # Project description
├── .gitignore                  # Git ignore rules
│
├── routes/                     # Modular blueprints
│   ├── __init__.py             # Package marker
│   ├── auth.py                 # Login, logout, student & examiner registration
│   ├── admin.py                # Admin dashboard, CRUD, approvals, rescheduling
│   ├── examiner.py             # Examiner dashboard, slots, evaluations
│   ├── student.py              # Student dashboard, exam search, slot booking
│   └── api.py                  # Optional: RESTful JSON endpoints
│
├── static/
│   └── css/
│       └── style.css           # Custom CSS styling
│
└── templates/                  # Jinja2 HTML templates
    ├── base.html               # Master layout with navbar & flash alerts
    ├── home.html               # Welcome landing page
    ├── error.html              # Friendly error display
    │
    ├── auth/
    │   ├── login.html          # Unified login
    │   ├── register_student.html
    │   └── register_examiner.html
    │
    ├── admin/
    │   ├── dashboard.html      # Overview statistics
    │   ├── courses.html        # Course list
    │   ├── course_form.html    # Add / edit course
    │   ├── exams.html          # Exam list & window toggles
    │   ├── exam_form.html      # Add / edit exam
    │   ├── exam_examiners.html # Assign examiners to exam
    │   ├── rubrics.html        # Rubrics list
    │   ├── rubric_form.html    # Add / edit rubric
    │   ├── examiners.html      # Examiner approval / deactivation
    │   ├── examiner_form.html  # Examiner management form
    │   ├── students.html       # Student roster & booking counts
    │   ├── slots.html          # Global slots oversight
    │   ├── bookings.html       # All bookings & cancellation
    │   ├── reschedule.html     # Reschedule slot & change examiner
    │   ├── search.html         # Multi-entity keyword search
    │   └── history.html        # Audit log trail
    │
    ├── examiner/
    │   ├── dashboard.html      # Assigned exams & upcoming slots
    │   ├── slots.html          # Manage own slots
    │   ├── slot_form.html      # Create / edit slot
    │   ├── slot_students.html  # View registered students
    │   ├── evaluate.html       # Rubric grading & feedback form
    │   └── profile.html        # Examiner profile
    │
    └── student/
        ├── dashboard.html      # Active bookings & upcoming exams
        ├── exams.html          # Browse & filter exams (pure GET)
        ├── exam_slots.html     # Choose available slot
        ├── bookings.html       # Active bookings & cancellation
        ├── history.html        # Past bookings & cancellations
        ├── results.html        # Published grades & feedback
        └── profile.html        # Student profile
```
