## PHASE 2: DATABASE MODELS & APPLICATION SKELETON (Day 6–10)

> 📺 Watch: [SQLAlchemy Models in Hindi](https://www.youtube.com/watch?v=cYWiDiIUxQc)

---

## 🧠 Big Picture: What is an ORM & SQLAlchemy?

In web applications, Python code works with **Objects** (instances of classes), but databases understand **SQL** (relational tables, rows, columns, foreign keys).
An **Object-Relational Mapper (ORM)** acts as a translator between Python and SQLite:
* Instead of writing raw SQL: `INSERT INTO user (email, role) VALUES ('admin@emp.com', 'admin')`
* You write Python: `user = User(email='admin@emp.com', role='admin'); db.session.add(user); db.session.commit()`

**Flask-SQLAlchemy** integrates SQLAlchemy with Flask, providing connection pooling, query helpers, and model bases.

---

## 🔑 Why Separate Configuration from Application Code?

* **Modularity**: Database URI, secret keys, and debug flags are centralized in `config.py`.
* **Security**: Secrets can be read from environment variables or changed without altering application logic.
* **Avoid Circular Imports**: `extensions.py` initializes `db = SQLAlchemy()` separately from `app.py` so models and routes can import `db` without causing circular import loops.

---

## 🛡️ Password Security: Why We NEVER Store Plain Passwords

Storing raw passwords in a database is a major security flaw. We use **Werkzeug**'s cryptographically secure hashing functions:
```python
from werkzeug.security import generate_password_hash, check_password_hash

# When registering:
password_hash = generate_password_hash("mysecretpassword")
# Stored string: "scrypt:32768:8:1$kLm9...$..."

# When logging in:
is_valid = check_password_hash(user.password_hash, entered_password)
```
Hashing is a one-way mathematical function. It cannot be decrypted back into plain text.

---

## ⚙️ Step 2.1 — Create `config.py`

This file defines the configuration class for Flask and SQLAlchemy:

```python
import os

BASE_DIR = os.path.abspath(os.path.dirname(__file__))


class Config:
    SECRET_KEY = os.environ.get("SECRET_KEY", "change-this-secret-key")
    SQLALCHEMY_DATABASE_URI = "sqlite:///" + os.path.join(BASE_DIR, "emp.db")
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

### Breakdown:
* `SECRET_KEY`: Used by Flask to cryptographically sign session cookies so clients cannot tamper with their role or identity.
* `SQLALCHEMY_DATABASE_URI`: Points to the local SQLite database file `sqlite:///emp.db`.
* `SQLALCHEMY_TRACK_MODIFICATIONS = False`: Disables an unnecessary overhead tracking system, saving memory and suppressing warnings.

---

## 🧩 Step 2.2 — Create `extensions.py`

```python
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager

db = SQLAlchemy()
login_manager = LoginManager()
login_manager.login_view = "auth.login"
login_manager.login_message_category = "warning"
```

### Breakdown:
* `db = SQLAlchemy()`: Instantiates the SQLAlchemy ORM instance without binding it to a specific app immediately. This allows circular-import-free imports across `models.py`, `app.py`, and blueprint routes.
* `login_manager = LoginManager()`: Initializes Flask-Login for session management.

---

## 🛠️ Step 2.3 — Create `utils.py`

Utility helpers for parsing HTML form date strings and audit logging:

```python
from datetime import datetime

EXAM_TYPES = ["Viva", "Practical", "Project Demo", "Assessment"]
EXAM_STATUSES = ["Draft", "Slot Creation", "Booking Open", "Closed", "Completed"]


def parse_date(s):
    """'2026-11-20' -> date, empty/invalid -> None"""
    try:
        return datetime.strptime(s, "%Y-%m-%d").date() if s else None
    except ValueError:
        return None


def parse_time(s):
    """'14:30' -> time, empty/invalid -> None"""
    try:
        return datetime.strptime(s, "%H:%M").time() if s else None
    except ValueError:
        return None
```

---

## 🏛️ Step 2.4 — Create `models.py`

The complete database schema implementing all 9 tables and relationships:

```python
from datetime import datetime, date
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash
from extensions import db, login_manager


class User(UserMixin, db.Model):
    __tablename__ = "users"
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    role = db.Column(db.String(20), nullable=False)          # admin / examiner / student
    phone = db.Column(db.String(20))
    roll_no = db.Column(db.String(30))                       # for students
    active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    examiner_profile = db.relationship("ExaminerProfile", backref="user", uselist=False)
    bookings = db.relationship("Booking", backref="student")

    def set_password(self, pw):
        self.password_hash = generate_password_hash(pw)

    def check_password(self, pw):
        return check_password_hash(self.password_hash, pw)

    @property
    def is_active(self):                                     # used by Flask-Login
        return bool(self.active)


@login_manager.user_loader
def load_user(user_id):
    return db.session.get(User, int(user_id))


class ExaminerProfile(db.Model):
    __tablename__ = "examiner_profiles"
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey("users.id"), unique=True, nullable=False)
    department = db.Column(db.String(100))
    contact = db.Column(db.String(50))
    employee_id = db.Column(db.String(30))
    status = db.Column(db.String(20), default="Pending")     # Pending / Approved / Deactivated


class Course(db.Model):
    __tablename__ = "courses"
    id = db.Column(db.Integer, primary_key=True)
    code = db.Column(db.String(20), unique=True, nullable=False)
    name = db.Column(db.String(150), nullable=False)
    description = db.Column(db.Text)
    status = db.Column(db.String(20), default="Active")      # Active / Inactive
    exams = db.relationship("Examination", backref="course")


class Examination(db.Model):
    __tablename__ = "examinations"
    id = db.Column(db.Integer, primary_key=True)
    course_id = db.Column(db.Integer, db.ForeignKey("courses.id"), nullable=False)
    name = db.Column(db.String(150), nullable=False)
    exam_type = db.Column(db.String(30), nullable=False)
    duration = db.Column(db.Integer, nullable=False)         # minutes
    max_marks = db.Column(db.Integer, nullable=False)
    slot_creation_start = db.Column(db.Date)
    slot_creation_end = db.Column(db.Date)
    booking_start = db.Column(db.Date)
    booking_end = db.Column(db.Date)
    slot_creation_open = db.Column(db.Boolean, default=False)   # admin switch
    booking_open = db.Column(db.Boolean, default=False)         # admin switch
    status = db.Column(db.String(20), default="Draft")

    rubrics = db.relationship("Rubric", backref="exam", cascade="all, delete-orphan")
    slots = db.relationship("Slot", backref="exam")
    assignments = db.relationship("ExamExaminer", backref="exam", cascade="all, delete-orphan")

    # ---------- business-rule helpers ----------
    def slot_creation_allowed(self):
        t = date.today()
        return bool(self.slot_creation_open and self.slot_creation_start and self.slot_creation_end
                    and self.slot_creation_start <= t <= self.slot_creation_end)

    def booking_allowed(self):
        t = date.today()
        return bool(self.booking_open and self.booking_start and self.booking_end
                    and self.booking_start <= t <= self.booking_end)

    def booking_started(self):
        return bool(self.booking_open) or bool(self.booking_start and date.today() >= self.booking_start)


class ExamExaminer(db.Model):
    """Which examiner is assigned to which examination."""
    __tablename__ = "exam_examiners"
    id = db.Column(db.Integer, primary_key=True)
    exam_id = db.Column(db.Integer, db.ForeignKey("examinations.id"), nullable=False)
    examiner_id = db.Column(db.Integer, db.ForeignKey("users.id"), nullable=False)
    examiner = db.relationship("User")


class Rubric(db.Model):
    __tablename__ = "rubrics"
    id = db.Column(db.Integer, primary_key=True)
    exam_id = db.Column(db.Integer, db.ForeignKey("examinations.id"), nullable=False)
    criterion = db.Column(db.String(150), nullable=False)
    max_marks = db.Column(db.Float, nullable=False)
    weightage = db.Column(db.Float, default=1.0)
    description = db.Column(db.Text)


class Slot(db.Model):
    __tablename__ = "slots"
    id = db.Column(db.Integer, primary_key=True)
    exam_id = db.Column(db.Integer, db.ForeignKey("examinations.id"), nullable=False)
    examiner_id = db.Column(db.Integer, db.ForeignKey("users.id"), nullable=False)
    date = db.Column(db.Date, nullable=False)
    start_time = db.Column(db.Time, nullable=False)
    end_time = db.Column(db.Time, nullable=False)
    capacity = db.Column(db.Integer, nullable=False)
    available_seats = db.Column(db.Integer, nullable=False)
    status = db.Column(db.String(20), default="Available")   # Available / Full / Cancelled / Completed

    examiner = db.relationship("User", foreign_keys=[examiner_id])
    bookings = db.relationship("Booking", backref="slot")


class Booking(db.Model):
    __tablename__ = "bookings"
    id = db.Column(db.Integer, primary_key=True)
    student_id = db.Column(db.Integer, db.ForeignKey("users.id"), nullable=False)
    slot_id = db.Column(db.Integer, db.ForeignKey("slots.id"), nullable=False)
    booking_date = db.Column(db.DateTime, default=datetime.utcnow)
    status = db.Column(db.String(20), default="Booked")      # Booked / Cancelled / Completed
    evaluation = db.relationship("Evaluation", backref="booking", uselist=False)


class Evaluation(db.Model):
    __tablename__ = "evaluations"
    id = db.Column(db.Integer, primary_key=True)
    booking_id = db.Column(db.Integer, db.ForeignKey("bookings.id"), unique=True, nullable=False)
    total_marks = db.Column(db.Float, default=0)
    remarks = db.Column(db.Text)
    published = db.Column(db.Boolean, default=False)
    evaluated_at = db.Column(db.DateTime, default=datetime.utcnow)
    scores = db.relationship("EvaluationScore", backref="evaluation", cascade="all, delete-orphan")


class EvaluationScore(db.Model):
    __tablename__ = "evaluation_scores"
    id = db.Column(db.Integer, primary_key=True)
    evaluation_id = db.Column(db.Integer, db.ForeignKey("evaluations.id"), nullable=False)
    rubric_id = db.Column(db.Integer, db.ForeignKey("rubrics.id"), nullable=False)
    marks = db.Column(db.Float, nullable=False)
    rubric = db.relationship("Rubric")
```

### Line-by-Line & Model Architecture Breakdown:
1. **`User`**: Implements `UserMixin` for Flask-Login compatibility. Holds role (`admin`, `examiner`, `student`), `is_active` flag, and relationships to bookings, slots, and evaluations.
2. **`Course`**: Identifies academic courses (`code`, `name`). Deleting a course cascades or restricts associated exams.
3. **`Exam`**: Tied to a Course. Stores scheduling windows:
   - `slot_creation_start` to `slot_creation_end`: Period examiners can schedule slots.
   - `booking_start` to `booking_end`: Period students can reserve slots.
4. **`exam_examiners`**: Association table representing the many-to-many relationship between exams and examiners.
5. **`Slot`**: Specific evaluation sessions created by an examiner. Contains `capacity` (max students) and `booked_count`.
6. **`Booking`**: Junction linking a student to a slot. Contains booking status (`Booked`, `Cancelled`, `Completed`).
7. **`Rubric`**: Scoring criteria attached to an exam.
8. **`Evaluation`**: One-to-one record on Booking. Stores `marks_obtained`, `feedback`, `attendance` ('Present'/'Absent'), and `is_published`.
9. **`ActivityLog`**: Audit record tracking administrative actions.

---

## 🚀 Step 2.5 — Create `app.py`

The main application factory, blueprint registrar, programmatic database generator, and admin seeder:

```python
from flask import Flask, render_template, redirect, url_for
from flask_login import current_user
from config import Config
from extensions import db, login_manager
from models import User


def seed_admin():
    """Admin is pre-created by code. There is NO admin registration page."""
    if not User.query.filter_by(role="admin").first():
        admin = User(name="Administrator", email="admin@emp.com", role="admin")
        admin.set_password("admin123")
        db.session.add(admin)
        db.session.commit()


def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)
    db.init_app(app)
    login_manager.init_app(app)

    from routes.auth import auth_bp
    from routes.admin import admin_bp
    from routes.examiner import examiner_bp
    from routes.student import student_bp
    from routes.api import api_bp
    app.register_blueprint(auth_bp)
    app.register_blueprint(admin_bp, url_prefix="/admin")
    app.register_blueprint(examiner_bp, url_prefix="/examiner")
    app.register_blueprint(student_bp, url_prefix="/student")
    app.register_blueprint(api_bp, url_prefix="/api")

    with app.app_context():
        db.create_all()          # tables are created programmatically
        seed_admin()

    @app.route("/")
    def home():
        if current_user.is_authenticated:
            return redirect(url_for("auth.dashboard_redirect"))
        return render_template("home.html")

    @app.errorhandler(403)
    def forbidden(e):
        return render_template("error.html", code=403,
                               msg="You are not allowed to open this page."), 403

    @app.errorhandler(404)
    def not_found(e):
        return render_template("error.html", code=404, msg="Page not found."), 404

    return app


if __name__ == "__main__":
    create_app().run(debug=True)
```

### Why Programmatic Seeding Matters:
Course rules require that:
1. The database tables are **created by code** via `db.create_all()`.
2. The initial admin is **seeded programmatically**. When `app.py` starts, `init_db()` checks if an admin exists. If not, it creates `admin@emp.com` with password `admin123`.

---

## 🧪 Step 2.6 — Verify Database Creation

Run the Python interpreter to verify that your models and database initialize without errors:

```bash
python -c "from app import create_app; app = create_app(); print('App and database initialized successfully!')"
```

Check your project folder — you should now see `emp.db` or `instance/emp.db` created!
