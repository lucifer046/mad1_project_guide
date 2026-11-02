# EMP — Complete Copy-Paste Code (every file)

Stack: **Flask + Flask-SQLAlchemy + Flask-Login + Jinja2 + HTML + Bootstrap 5 (CSS only) + SQLite**. No JavaScript is used anywhere.
Use this file together with `EMP_Project_Roadmap.md` (that file explains the *plan*; this one gives the *code*).

> **How to use:** create each file with the exact path shown in the heading, paste the code, save. Follow the order in Section 2. Commit to Git after each group of files.

---

## 1. Folder structure

```
emp/
├── app.py
├── config.py
├── extensions.py
├── models.py
├── decorators.py
├── utils.py
├── seed_demo.py              (optional, demo data)
├── requirements.txt
├── README.md
├── .gitignore
├── routes/
│   ├── __init__.py           (empty file)
│   ├── auth.py
│   ├── admin.py
│   ├── examiner.py
│   ├── student.py
│   └── api.py                (optional)
├── static/css/style.css
└── templates/
    ├── base.html  home.html  error.html
    ├── auth/      login.html  register_student.html  register_examiner.html
    ├── admin/     dashboard.html courses.html course_form.html exams.html exam_form.html
    │              exam_examiners.html rubrics.html rubric_form.html examiners.html
    │              examiner_form.html students.html slots.html bookings.html
    │              reschedule.html search.html history.html
    ├── examiner/  dashboard.html profile.html slots.html slot_form.html slot_students.html evaluate.html
    └── student/   dashboard.html profile.html exams.html exam_slots.html bookings.html history.html results.html
```

## 2. Order of creation

1. Setup + Git (Milestone 0) → `requirements.txt`, `.gitignore`, `README.md`
2. `config.py` → `extensions.py` → `utils.py` → `models.py` → `decorators.py` → `app.py`
3. Empty blueprint files → run app once (database `emp.db` gets created)
4. `routes/auth.py` + `base.html`, `home.html`, `error.html`, auth templates
5. `routes/admin.py` + admin templates
6. `routes/examiner.py` + examiner templates
7. `routes/student.py` + student templates
8. `static/css/style.css`, `seed_demo.py`, optional `routes/api.py`

## 3. Setup commands

```bash
mkdir emp && cd emp
python -m venv venv
# Windows:        venv\Scripts\activate
# Linux/WSL/Mac:  source venv/bin/activate
pip install flask flask-sqlalchemy flask-login
pip freeze > requirements.txt
git init
```

---

## 4. Root files

### `requirements.txt` (use whatever `pip freeze` gives; minimum is:)
```
Flask>=3.0
Flask-SQLAlchemy>=3.1
Flask-Login>=0.6
```

### `.gitignore`
```
venv/
__pycache__/
*.pyc
instance/
*.db
.env
```

### `README.md`
```markdown
# Examination Management Portal (EMP)
Flask + SQLite web app where Admin, Examiners and Students manage exam slots, bookings and evaluations.

## Run
pip install -r requirements.txt
python app.py
Open http://127.0.0.1:5000

Admin login: admin@emp.com / admin123
```

### `config.py`
```python
import os

BASE_DIR = os.path.abspath(os.path.dirname(__file__))


class Config:
    SECRET_KEY = os.environ.get("SECRET_KEY", "change-this-secret-key")
    SQLALCHEMY_DATABASE_URI = "sqlite:///" + os.path.join(BASE_DIR, "emp.db")
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

### `extensions.py`
```python
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager

db = SQLAlchemy()
login_manager = LoginManager()
login_manager.login_view = "auth.login"
login_manager.login_message_category = "warning"
```

### `utils.py`
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

### `models.py`
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

### `decorators.py`
```python
from functools import wraps
from flask import abort
from flask_login import current_user
from extensions import login_manager


def role_required(role):
    """Allow only logged-in users with this role. Examiners must also be Approved."""
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            if not current_user.is_authenticated:
                return login_manager.unauthorized()
            if current_user.role != role:
                abort(403)
            if role == "examiner":
                p = current_user.examiner_profile
                if not p or p.status != "Approved":
                    abort(403)
            return f(*args, **kwargs)
        return wrapper
    return decorator
```

### `app.py`
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

> If you haven't made `routes/api.py` yet, create it now (Section 8) or temporarily remove the two `api` lines in `app.py`.

---

## 5. Routes

### `routes/__init__.py`
```python
```
(leave it empty)

### `routes/auth.py`
```python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from flask_login import login_user, logout_user, login_required, current_user
from extensions import db
from models import User, ExaminerProfile

auth_bp = Blueprint("auth", __name__)


@auth_bp.route("/register/student", methods=["GET", "POST"])
def register_student():
    if request.method == "POST":
        name = request.form.get("name", "").strip()
        email = request.form.get("email", "").strip().lower()
        pw = request.form.get("password", "")
        if not name or not email or len(pw) < 6:
            flash("Fill all fields; password must be at least 6 characters.", "danger")
        elif User.query.filter_by(email=email).first():
            flash("This email is already registered.", "danger")
        else:
            u = User(name=name, email=email, role="student",
                     roll_no=request.form.get("roll_no", "").strip(),
                     phone=request.form.get("phone", "").strip())
            u.set_password(pw)
            db.session.add(u)
            db.session.commit()
            flash("Registered successfully! Please log in.", "success")
            return redirect(url_for("auth.login"))
    return render_template("auth/register_student.html")


@auth_bp.route("/register/examiner", methods=["GET", "POST"])
def register_examiner():
    if request.method == "POST":
        name = request.form.get("name", "").strip()
        email = request.form.get("email", "").strip().lower()
        pw = request.form.get("password", "")
        if not name or not email or len(pw) < 6:
            flash("Fill all fields; password must be at least 6 characters.", "danger")
        elif User.query.filter_by(email=email).first():
            flash("This email is already registered.", "danger")
        else:
            u = User(name=name, email=email, role="examiner",
                     phone=request.form.get("contact", "").strip())
            u.set_password(pw)
            db.session.add(u)
            db.session.flush()                      # gives u.id before commit
            db.session.add(ExaminerProfile(user_id=u.id,
                                           department=request.form.get("department", "").strip(),
                                           contact=request.form.get("contact", "").strip(),
                                           employee_id=request.form.get("employee_id", "").strip(),
                                           status="Pending"))
            db.session.commit()
            flash("Registered. You can log in after the Admin approves your account.", "info")
            return redirect(url_for("auth.login"))
    return render_template("auth/register_examiner.html")


@auth_bp.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        email = request.form.get("email", "").strip().lower()
        user = User.query.filter_by(email=email).first()
        if not user or not user.check_password(request.form.get("password", "")):
            flash("Invalid email or password.", "danger")
        elif not user.active:
            flash("This account is deactivated.", "danger")
        elif user.role == "examiner" and (not user.examiner_profile
                                          or user.examiner_profile.status != "Approved"):
            status = user.examiner_profile.status if user.examiner_profile else "Pending"
            flash(f"Your examiner account is {status}. Admin approval is required.", "warning")
        else:
            login_user(user)
            return redirect(url_for("auth.dashboard_redirect"))
    return render_template("auth/login.html")


@auth_bp.route("/go")
@login_required
def dashboard_redirect():
    target = {"admin": "admin.dashboard",
              "examiner": "examiner.dashboard",
              "student": "student.dashboard"}[current_user.role]
    return redirect(url_for(target))


@auth_bp.route("/logout")
@login_required
def logout():
    logout_user()
    flash("You have been logged out.", "info")
    return redirect(url_for("home"))
```

### `routes/admin.py`
```python
from flask import Blueprint, render_template, request, redirect, url_for, flash
from extensions import db
from decorators import role_required
from models import (User, ExaminerProfile, Course, Examination, ExamExaminer,
                    Rubric, Slot, Booking)
from utils import parse_date, EXAM_TYPES, EXAM_STATUSES

admin_bp = Blueprint("admin", __name__)


def to_int(value, default=-1):
    return int(value) if str(value).isdigit() else default


# ------------------------------------------------------------------ dashboard
@admin_bp.route("/dashboard")
@role_required("admin")
def dashboard():
    stats = {
        "courses": Course.query.count(),
        "exams": Examination.query.count(),
        "examiners": User.query.filter_by(role="examiner").count(),
        "students": User.query.filter_by(role="student").count(),
        "slots": Slot.query.count(),
        "bookings": Booking.query.count(),
        "pending": ExaminerProfile.query.filter_by(status="Pending").count(),
    }
    return render_template("admin/dashboard.html", stats=stats)


# ------------------------------------------------------------------ courses
@admin_bp.route("/courses")
@role_required("admin")
def courses():
    return render_template("admin/courses.html", courses=Course.query.order_by(Course.code).all())


@admin_bp.route("/courses/add", methods=["GET", "POST"])
@role_required("admin")
def course_add():
    if request.method == "POST":
        code = request.form.get("code", "").strip().upper()
        name = request.form.get("name", "").strip()
        if not code or not name:
            flash("Course code and name are required.", "danger")
        elif Course.query.filter_by(code=code).first():
            flash("Course code already exists.", "danger")
        else:
            db.session.add(Course(code=code, name=name,
                                  description=request.form.get("description", "").strip(),
                                  status=request.form.get("status", "Active")))
            db.session.commit()
            flash("Course added.", "success")
            return redirect(url_for("admin.courses"))
    return render_template("admin/course_form.html", course=None)


@admin_bp.route("/courses/<int:cid>/edit", methods=["GET", "POST"])
@role_required("admin")
def course_edit(cid):
    c = db.get_or_404(Course, cid)
    if request.method == "POST":
        code = request.form.get("code", "").strip().upper()
        other = Course.query.filter(Course.code == code, Course.id != c.id).first()
        if not code or not request.form.get("name", "").strip():
            flash("Course code and name are required.", "danger")
        elif other:
            flash("Another course already uses this code.", "danger")
        else:
            c.code = code
            c.name = request.form["name"].strip()
            c.description = request.form.get("description", "").strip()
            c.status = request.form.get("status", "Active")
            db.session.commit()
            flash("Course updated.", "success")
            return redirect(url_for("admin.courses"))
    return render_template("admin/course_form.html", course=c)


@admin_bp.route("/courses/<int:cid>/delete", methods=["POST"])
@role_required("admin")
def course_delete(cid):
    c = db.get_or_404(Course, cid)
    if c.exams:
        flash("Cannot delete: this course has examinations. Mark it Inactive instead.", "warning")
    else:
        db.session.delete(c)
        db.session.commit()
        flash("Course deleted.", "success")
    return redirect(url_for("admin.courses"))


# ------------------------------------------------------------------ examinations
def validate_exam(f):
    if not f.get("name", "").strip():
        return "Examination name is required."
    try:
        if int(f["duration"]) <= 0 or int(f["max_marks"]) <= 0:
            return "Duration and maximum marks must be positive."
    except (ValueError, KeyError):
        return "Duration and maximum marks must be numbers."
    for a, b, label in [("slot_creation_start", "slot_creation_end", "Slot creation"),
                        ("booking_start", "booking_end", "Booking")]:
        s, e = parse_date(f.get(a)), parse_date(f.get(b))
        if s and e and s > e:
            return f"{label} start date cannot be after its end date."
    return None


def fill_exam(e, f):
    e.course_id = int(f["course_id"])
    e.name = f["name"].strip()
    e.exam_type = f["exam_type"]
    e.duration = int(f["duration"])
    e.max_marks = int(f["max_marks"])
    e.slot_creation_start = parse_date(f.get("slot_creation_start"))
    e.slot_creation_end = parse_date(f.get("slot_creation_end"))
    e.booking_start = parse_date(f.get("booking_start"))
    e.booking_end = parse_date(f.get("booking_end"))
    e.status = f.get("status", "Draft")


@admin_bp.route("/exams")
@role_required("admin")
def exams():
    return render_template("admin/exams.html", exams=Examination.query.order_by(Examination.id.desc()).all())


@admin_bp.route("/exams/add", methods=["GET", "POST"])
@role_required("admin")
def exam_add():
    courses_list = Course.query.filter_by(status="Active").all()
    if request.method == "POST":
        err = validate_exam(request.form)
        if err:
            flash(err, "danger")
        else:
            e = Examination()
            fill_exam(e, request.form)
            db.session.add(e)
            db.session.commit()
            flash("Examination created.", "success")
            return redirect(url_for("admin.exams"))
    return render_template("admin/exam_form.html", exam=None, courses=courses_list,
                           types=EXAM_TYPES, statuses=EXAM_STATUSES)


@admin_bp.route("/exams/<int:eid>/edit", methods=["GET", "POST"])
@role_required("admin")
def exam_edit(eid):
    e = db.get_or_404(Examination, eid)
    courses_list = Course.query.all()
    if request.method == "POST":
        err = validate_exam(request.form)
        if err:
            flash(err, "danger")
        else:
            fill_exam(e, request.form)
            db.session.commit()
            flash("Examination updated.", "success")
            return redirect(url_for("admin.exams"))
    return render_template("admin/exam_form.html", exam=e, courses=courses_list,
                           types=EXAM_TYPES, statuses=EXAM_STATUSES)


@admin_bp.route("/exams/<int:eid>/delete", methods=["POST"])
@role_required("admin")
def exam_delete(eid):
    e = db.get_or_404(Examination, eid)
    if e.slots:
        flash("Cannot delete: slots already exist for this examination. Set status to Closed/Completed.", "warning")
    else:
        db.session.delete(e)
        db.session.commit()
        flash("Examination deleted.", "success")
    return redirect(url_for("admin.exams"))


@admin_bp.route("/exams/<int:eid>/toggle/<what>", methods=["POST"])
@role_required("admin")
def exam_toggle(eid, what):
    e = db.get_or_404(Examination, eid)
    if what == "slots":
        e.slot_creation_open = not e.slot_creation_open
        if e.slot_creation_open:
            e.status = "Slot Creation"
        flash("Slot creation is now %s." % ("OPEN" if e.slot_creation_open else "CLOSED"), "success")
    elif what == "booking":
        e.booking_open = not e.booking_open
        e.status = "Booking Open" if e.booking_open else "Closed"
        flash("Student booking is now %s." % ("OPEN" if e.booking_open else "CLOSED"), "success")
    db.session.commit()
    return redirect(url_for("admin.exams"))


# ------------------------------------------------------------------ assign examiners to exam
@admin_bp.route("/exams/<int:eid>/examiners", methods=["GET", "POST"])
@role_required("admin")
def exam_examiners(eid):
    exam = db.get_or_404(Examination, eid)
    if request.method == "POST":
        uid = to_int(request.form.get("examiner_id"))
        if ExamExaminer.query.filter_by(exam_id=eid, examiner_id=uid).first():
            flash("Already assigned.", "warning")
        else:
            db.session.add(ExamExaminer(exam_id=eid, examiner_id=uid))
            db.session.commit()
            flash("Examiner assigned.", "success")
        return redirect(url_for("admin.exam_examiners", eid=eid))
    assigned_ids = [a.examiner_id for a in exam.assignments]
    available = (User.query.join(ExaminerProfile, ExaminerProfile.user_id == User.id)
                 .filter(User.role == "examiner", ExaminerProfile.status == "Approved",
                         ~User.id.in_(assigned_ids)).all())
    return render_template("admin/exam_examiners.html", exam=exam, available=available)


@admin_bp.route("/exams/<int:eid>/examiners/<int:aid>/remove", methods=["POST"])
@role_required("admin")
def exam_examiner_remove(eid, aid):
    a = db.get_or_404(ExamExaminer, aid)
    has_slots = Slot.query.filter_by(exam_id=eid, examiner_id=a.examiner_id).first()
    if has_slots:
        flash("This examiner already has slots in this exam. Use 'Change examiner' on the Slots page.", "warning")
    else:
        db.session.delete(a)
        db.session.commit()
        flash("Assignment removed.", "success")
    return redirect(url_for("admin.exam_examiners", eid=eid))


# ------------------------------------------------------------------ rubrics
@admin_bp.route("/exams/<int:eid>/rubrics", methods=["GET", "POST"])
@role_required("admin")
def rubrics(eid):
    exam = db.get_or_404(Examination, eid)
    if request.method == "POST":
        try:
            mx = float(request.form["max_marks"])
            wt = float(request.form.get("weightage") or 1)
            if mx <= 0 or wt <= 0 or not request.form.get("criterion", "").strip():
                raise ValueError
        except (ValueError, KeyError):
            flash("Enter a criterion name and positive numbers for marks and weightage.", "danger")
        else:
            db.session.add(Rubric(exam_id=eid, criterion=request.form["criterion"].strip(),
                                  max_marks=mx, weightage=wt,
                                  description=request.form.get("description", "").strip()))
            db.session.commit()
            flash("Rubric criterion added.", "success")
        return redirect(url_for("admin.rubrics", eid=eid))
    total = sum(r.max_marks for r in exam.rubrics)
    return render_template("admin/rubrics.html", exam=exam, total=total)


@admin_bp.route("/rubrics/<int:rid>/edit", methods=["GET", "POST"])
@role_required("admin")
def rubric_edit(rid):
    r = db.get_or_404(Rubric, rid)
    if request.method == "POST":
        try:
            mx = float(request.form["max_marks"])
            wt = float(request.form.get("weightage") or 1)
            if mx <= 0 or wt <= 0:
                raise ValueError
        except (ValueError, KeyError):
            flash("Marks and weightage must be positive numbers.", "danger")
        else:
            r.criterion = request.form["criterion"].strip()
            r.max_marks, r.weightage = mx, wt
            r.description = request.form.get("description", "").strip()
            db.session.commit()
            flash("Rubric updated.", "success")
            return redirect(url_for("admin.rubrics", eid=r.exam_id))
    return render_template("admin/rubric_form.html", rubric=r)


@admin_bp.route("/rubrics/<int:rid>/delete", methods=["POST"])
@role_required("admin")
def rubric_delete(rid):
    r = db.get_or_404(Rubric, rid)
    eid = r.exam_id
    db.session.delete(r)
    db.session.commit()
    flash("Rubric criterion deleted.", "success")
    return redirect(url_for("admin.rubrics", eid=eid))


# ------------------------------------------------------------------ examiners
@admin_bp.route("/examiners")
@role_required("admin")
def examiners():
    profiles = ExaminerProfile.query.order_by(ExaminerProfile.status.desc()).all()
    return render_template("admin/examiners.html", profiles=profiles)


@admin_bp.route("/examiners/add", methods=["GET", "POST"])
@role_required("admin")
def examiner_add():
    if request.method == "POST":
        name = request.form.get("name", "").strip()
        email = request.form.get("email", "").strip().lower()
        pw = request.form.get("password", "")
        if not name or not email or len(pw) < 6:
            flash("Fill all fields; password must be at least 6 characters.", "danger")
        elif User.query.filter_by(email=email).first():
            flash("Email already registered.", "danger")
        else:
            u = User(name=name, email=email, role="examiner")
            u.set_password(pw)
            db.session.add(u)
            db.session.flush()
            db.session.add(ExaminerProfile(user_id=u.id,
                                           department=request.form.get("department", "").strip(),
                                           contact=request.form.get("contact", "").strip(),
                                           employee_id=request.form.get("employee_id", "").strip(),
                                           status="Approved"))
            db.session.commit()
            flash("Examiner added and approved.", "success")
            return redirect(url_for("admin.examiners"))
    return render_template("admin/examiner_form.html")


@admin_bp.route("/examiners/<int:pid>/<action>", methods=["POST"])
@role_required("admin")
def examiner_action(pid, action):
    p = db.get_or_404(ExaminerProfile, pid)
    if action == "approve":
        p.status = "Approved"
    elif action == "deactivate":
        p.status = "Deactivated"
    else:
        flash("Unknown action.", "danger")
        return redirect(url_for("admin.examiners"))
    db.session.commit()
    flash(f"Examiner {p.user.name} is now {p.status}.", "success")
    return redirect(url_for("admin.examiners"))


# ------------------------------------------------------------------ students
@admin_bp.route("/students")
@role_required("admin")
def students():
    return render_template("admin/students.html",
                           students=User.query.filter_by(role="student").order_by(User.id).all())


# ------------------------------------------------------------------ slots
@admin_bp.route("/slots")
@role_required("admin")
def slots():
    all_slots = Slot.query.order_by(Slot.date.desc(), Slot.start_time).all()
    approved = (User.query.join(ExaminerProfile, ExaminerProfile.user_id == User.id)
                .filter(User.role == "examiner", ExaminerProfile.status == "Approved").all())
    return render_template("admin/slots.html", slots=all_slots, examiners=approved)


@admin_bp.route("/slots/<int:sid>/change-examiner", methods=["POST"])
@role_required("admin")
def change_examiner(sid):
    s = db.get_or_404(Slot, sid)
    uid = to_int(request.form.get("examiner_id"))
    new = db.session.get(User, uid)
    if not new or new.role != "examiner" or not new.examiner_profile \
            or new.examiner_profile.status != "Approved":
        flash("Choose an approved examiner.", "danger")
    else:
        s.examiner_id = new.id
        if not ExamExaminer.query.filter_by(exam_id=s.exam_id, examiner_id=new.id).first():
            db.session.add(ExamExaminer(exam_id=s.exam_id, examiner_id=new.id))
        db.session.commit()
        flash(f"Slot #{s.id} is now assigned to {new.name}.", "success")
    return redirect(url_for("admin.slots"))


# ------------------------------------------------------------------ bookings
@admin_bp.route("/bookings")
@role_required("admin")
def bookings():
    status = request.args.get("status", "")
    q = Booking.query
    if status:
        q = q.filter(Booking.status == status)
    return render_template("admin/bookings.html",
                           bookings=q.order_by(Booking.booking_date.desc()).all(), status=status)


@admin_bp.route("/bookings/<int:bid>/reschedule", methods=["GET", "POST"])
@role_required("admin")
def reschedule(bid):
    b = db.get_or_404(Booking, bid)
    if b.status != "Booked":
        flash("Only active (Booked) bookings can be rescheduled.", "warning")
        return redirect(url_for("admin.bookings"))
    old = b.slot
    options = Slot.query.filter(Slot.exam_id == old.exam_id, Slot.id != old.id,
                                Slot.status == "Available", Slot.available_seats > 0).all()
    if request.method == "POST":
        new = db.session.get(Slot, to_int(request.form.get("slot_id")))
        if not new or new.exam_id != old.exam_id or new.status != "Available" or new.available_seats <= 0:
            flash("Invalid or full slot.", "danger")
        else:
            old.available_seats += 1
            if old.status == "Full":
                old.status = "Available"
            new.available_seats -= 1
            if new.available_seats == 0:
                new.status = "Full"
            b.slot_id = new.id
            db.session.commit()
            flash("Booking rescheduled.", "success")
            return redirect(url_for("admin.bookings"))
    return render_template("admin/reschedule.html", booking=b, options=options)


# ------------------------------------------------------------------ search
@admin_bp.route("/search")
@role_required("admin")
def search():
    q = request.args.get("q", "").strip()
    kind = request.args.get("kind", "students")
    results = []
    if q:
        like = f"%{q}%"
        num = to_int(q)
        if kind == "students":
            results = User.query.filter(User.role == "student",
                                        db.or_(User.name.ilike(like), User.id == num)).all()
        elif kind == "examiners":
            results = User.query.filter(User.role == "examiner",
                                        db.or_(User.name.ilike(like), User.id == num)).all()
        elif kind == "exams":
            results = Examination.query.filter(
                db.or_(Examination.name.ilike(like), Examination.id == num)).all()
        elif kind == "bookings":
            results = (Booking.query.join(User, Booking.student_id == User.id)
                       .filter(db.or_(User.name.ilike(like), Booking.id == num,
                                      Booking.student_id == num)).all())
    return render_template("admin/search.html", results=results, q=q, kind=kind)


# ------------------------------------------------------------------ history
@admin_bp.route("/history")
@role_required("admin")
def history():
    rows = Booking.query.order_by(Booking.booking_date.desc()).all()
    return render_template("admin/history.html", rows=rows)
```

### `routes/examiner.py`
```python
from flask import Blueprint, render_template, request, redirect, url_for, flash, abort
from flask_login import current_user
from extensions import db
from decorators import role_required
from models import (Examination, ExamExaminer, Slot, Booking, Evaluation, EvaluationScore)
from utils import parse_date, parse_time

examiner_bp = Blueprint("examiner", __name__)


def my_exams():
    return (Examination.query.join(ExamExaminer, ExamExaminer.exam_id == Examination.id)
            .filter(ExamExaminer.examiner_id == current_user.id).all())


def own_slot_or_403(sid):
    s = db.get_or_404(Slot, sid)
    if s.examiner_id != current_user.id:
        abort(403)
    return s


def overlaps(examiner_id, d, st, et, exclude_id=None):
    q = Slot.query.filter(Slot.examiner_id == examiner_id, Slot.date == d,
                          Slot.status != "Cancelled", Slot.start_time < et, Slot.end_time > st)
    if exclude_id:
        q = q.filter(Slot.id != exclude_id)
    return q.first() is not None


# ------------------------------------------------------------------ dashboard
@examiner_bp.route("/dashboard")
@role_required("examiner")
def dashboard():
    slots = Slot.query.filter_by(examiner_id=current_user.id).order_by(Slot.date).all()
    base = Booking.query.join(Slot, Booking.slot_id == Slot.id).filter(Slot.examiner_id == current_user.id)
    booked = base.filter(Booking.status.in_(["Booked", "Completed"])).count()
    pending = base.filter(Booking.status == "Booked").count()
    return render_template("examiner/dashboard.html", exams=my_exams(), slots=slots,
                           booked=booked, pending=pending)


@examiner_bp.route("/slots")
@role_required("examiner")
def slots():
    mine = Slot.query.filter_by(examiner_id=current_user.id).order_by(Slot.date, Slot.start_time).all()
    return render_template("examiner/slots.html", slots=mine)


# ------------------------------------------------------------------ profile
@examiner_bp.route("/profile", methods=["GET", "POST"])
@role_required("examiner")
def profile():
    p = current_user.examiner_profile
    if request.method == "POST":
        name = request.form.get("name", "").strip()
        if not name:
            flash("Name is required.", "danger")
        else:
            current_user.name = name
            current_user.phone = request.form.get("contact", "").strip()
            p.contact = request.form.get("contact", "").strip()
            p.department = request.form.get("department", "").strip()
            p.employee_id = request.form.get("employee_id", "").strip()
            if request.form.get("password"):
                if len(request.form["password"]) < 6:
                    flash("Password must be at least 6 characters.", "danger")
                    return render_template("examiner/profile.html", p=p)
                current_user.set_password(request.form["password"])
            db.session.commit()
            flash("Profile updated.", "success")
            return redirect(url_for("examiner.profile"))
    return render_template("examiner/profile.html", p=p)


# ------------------------------------------------------------------ slots CRUD
def read_slot_form(f):
    d = parse_date(f.get("date"))
    st, et = parse_time(f.get("start_time")), parse_time(f.get("end_time"))
    try:
        cap = int(f.get("capacity", 0))
    except ValueError:
        cap = 0
    if not d or not st or not et:
        return None, "Enter a valid date, start time and end time."
    if et <= st:
        return None, "End time must be after start time."
    if cap < 1:
        return None, "Capacity must be at least 1."
    return (d, st, et, cap), None


@examiner_bp.route("/exams/<int:eid>/slots/add", methods=["GET", "POST"])
@role_required("examiner")
def slot_add(eid):
    exam = db.get_or_404(Examination, eid)
    if not ExamExaminer.query.filter_by(exam_id=eid, examiner_id=current_user.id).first():
        abort(403)
    if not exam.slot_creation_allowed():
        flash("Slot creation is closed for this examination.", "warning")
        return redirect(url_for("examiner.dashboard"))
    if request.method == "POST":
        data, err = read_slot_form(request.form)
        if err:
            flash(err, "danger")
        elif overlaps(current_user.id, *data[:3]):
            flash("You already have a slot overlapping this time.", "danger")
        else:
            d, st, et, cap = data
            db.session.add(Slot(exam_id=eid, examiner_id=current_user.id, date=d,
                                start_time=st, end_time=et, capacity=cap, available_seats=cap))
            db.session.commit()
            flash("Slot created.", "success")
            return redirect(url_for("examiner.slots"))
    return render_template("examiner/slot_form.html", exam=exam, slot=None)


@examiner_bp.route("/slots/<int:sid>/edit", methods=["GET", "POST"])
@role_required("examiner")
def slot_edit(sid):
    s = own_slot_or_403(sid)
    if s.exam.booking_started() or s.bookings:
        flash("This slot can no longer be edited: booking has started.", "warning")
        return redirect(url_for("examiner.slots"))
    if request.method == "POST":
        data, err = read_slot_form(request.form)
        if err:
            flash(err, "danger")
        elif overlaps(current_user.id, *data[:3], exclude_id=s.id):
            flash("You already have a slot overlapping this time.", "danger")
        else:
            s.date, s.start_time, s.end_time, s.capacity = data
            s.available_seats = s.capacity
            s.status = "Available"
            db.session.commit()
            flash("Slot updated.", "success")
            return redirect(url_for("examiner.slots"))
    return render_template("examiner/slot_form.html", exam=s.exam, slot=s)


@examiner_bp.route("/slots/<int:sid>/delete", methods=["POST"])
@role_required("examiner")
def slot_delete(sid):
    s = own_slot_or_403(sid)
    if s.exam.booking_started() or s.bookings:
        flash("Cannot delete: booking has started.", "warning")
    else:
        db.session.delete(s)
        db.session.commit()
        flash("Slot deleted.", "success")
    return redirect(url_for("examiner.slots"))


@examiner_bp.route("/slots/<int:sid>/students")
@role_required("examiner")
def slot_students(sid):
    s = own_slot_or_403(sid)
    return render_template("examiner/slot_students.html", slot=s, bookings=s.bookings)


# ------------------------------------------------------------------ evaluation
@examiner_bp.route("/bookings/<int:bid>/evaluate", methods=["GET", "POST"])
@role_required("examiner")
def evaluate(bid):
    b = db.get_or_404(Booking, bid)
    if b.slot.examiner_id != current_user.id:        # KEY RULE: only own slots
        abort(403)
    if b.status == "Cancelled":
        flash("This booking was cancelled; it cannot be evaluated.", "warning")
        return redirect(url_for("examiner.slot_students", sid=b.slot_id))
    exam = b.slot.exam
    rubrics = exam.rubrics
    if not rubrics:
        flash("No rubric has been defined for this examination yet. Ask the Admin to add one.", "warning")
        return redirect(url_for("examiner.slot_students", sid=b.slot_id))

    ev = b.evaluation
    existing = {s.rubric_id: s.marks for s in ev.scores} if ev else {}

    if request.method == "POST":
        scores, ok = {}, True
        for r in rubrics:
            try:
                val = float(request.form.get(f"r_{r.id}", ""))
            except ValueError:
                flash(f"'{r.criterion}': enter a number.", "danger")
                ok = False
                continue
            if val < 0 or val > r.max_marks:
                flash(f"'{r.criterion}': marks must be between 0 and {r.max_marks:g}.", "danger")
                ok = False
            scores[r.id] = val
        if ok:
            # total is scaled to the exam's maximum marks using rubric weightage
            den = sum(r.max_marks * r.weightage for r in rubrics)
            num = sum(scores[r.id] * r.weightage for r in rubrics)
            total = round(num / den * exam.max_marks, 2) if den else 0
            if not ev:
                ev = Evaluation(booking_id=b.id)
                db.session.add(ev)
                db.session.flush()
            else:
                EvaluationScore.query.filter_by(evaluation_id=ev.id).delete()
            for rid, val in scores.items():
                db.session.add(EvaluationScore(evaluation_id=ev.id, rubric_id=rid, marks=val))
            ev.total_marks = total
            ev.remarks = request.form.get("remarks", "").strip()
            ev.published = bool(request.form.get("publish"))
            b.status = "Completed"
            db.session.flush()
            slot = b.slot
            if all(x.status != "Booked" for x in slot.bookings):
                slot.status = "Completed"
            db.session.commit()
            flash("Evaluation saved.", "success")
            return redirect(url_for("examiner.slot_students", sid=b.slot_id))
        existing = {rid: v for rid, v in scores.items()}
    return render_template("examiner/evaluate.html", booking=b, rubrics=rubrics, ev=ev, existing=existing)
```

### `routes/student.py`
```python
from datetime import date
from flask import Blueprint, render_template, request, redirect, url_for, flash, abort
from flask_login import current_user
from extensions import db
from decorators import role_required
from models import Examination, Course, Slot, Booking, Evaluation
from utils import EXAM_TYPES

student_bp = Blueprint("student", __name__)


def my_bookings():
    return Booking.query.filter_by(student_id=current_user.id).order_by(Booking.booking_date.desc()).all()


# ------------------------------------------------------------------ dashboard
@student_bp.route("/dashboard")
@role_required("student")
def dashboard():
    today = date.today()
    bookings = my_bookings()
    open_exams = [e for e in Examination.query.all() if e.booking_allowed()]
    open_slots = [s for s in Slot.query.filter(Slot.status == "Available", Slot.available_seats > 0).all()
                  if s.exam.booking_allowed()]
    upcoming = [b for b in bookings if b.status == "Booked" and b.slot.date >= today]
    booked = [b for b in bookings if b.status == "Booked"]
    results = [b for b in bookings if b.evaluation and b.evaluation.published]
    return render_template("student/dashboard.html", open_exams=open_exams, open_slots=open_slots,
                           upcoming=upcoming, booked=booked, results=results)


# ------------------------------------------------------------------ profile
@student_bp.route("/profile", methods=["GET", "POST"])
@role_required("student")
def profile():
    if request.method == "POST":
        name = request.form.get("name", "").strip()
        if not name:
            flash("Name is required.", "danger")
        else:
            current_user.name = name
            current_user.phone = request.form.get("phone", "").strip()
            current_user.roll_no = request.form.get("roll_no", "").strip()
            pw = request.form.get("password", "")
            if pw:
                if len(pw) < 6:
                    flash("Password must be at least 6 characters.", "danger")
                    return render_template("student/profile.html")
                current_user.set_password(pw)
            db.session.commit()
            flash("Profile updated.", "success")
            return redirect(url_for("student.profile"))
    return render_template("student/profile.html")


# ------------------------------------------------------------------ exams (search + filter)
@student_bp.route("/exams")
@role_required("student")
def exams():
    q = request.args.get("q", "").strip()
    etype = request.args.get("type", "")
    query = Examination.query.join(Course, Examination.course_id == Course.id) \
        .filter(Examination.status != "Draft")
    if q:
        like = f"%{q}%"
        query = query.filter(db.or_(Examination.name.ilike(like), Course.name.ilike(like),
                                    Course.code.ilike(like)))
    if etype:
        query = query.filter(Examination.exam_type == etype)
    return render_template("student/exams.html", exams=query.all(), q=q, etype=etype, types=EXAM_TYPES)


@student_bp.route("/exams/<int:eid>/slots")
@role_required("student")
def exam_slots(eid):
    exam = db.get_or_404(Examination, eid)
    if exam.status == "Draft":
        abort(404)
    slots = (Slot.query.filter(Slot.exam_id == eid, Slot.status.in_(["Available", "Full"]))
             .order_by(Slot.date, Slot.start_time).all())
    my = (Booking.query.join(Slot, Booking.slot_id == Slot.id)
          .filter(Booking.student_id == current_user.id, Slot.exam_id == eid,
                  Booking.status.in_(["Booked", "Completed"])).first())
    return render_template("student/exam_slots.html", exam=exam, slots=slots, my=my,
                           allowed=exam.booking_allowed())


# ------------------------------------------------------------------ book / cancel
@student_bp.route("/slots/<int:sid>/book", methods=["POST"])
@role_required("student")
def book(sid):
    slot = db.get_or_404(Slot, sid)
    exam = slot.exam
    if not exam.booking_allowed():                                   # rule 1: window open
        flash("Booking is closed for this examination.", "warning")
    elif slot.status != "Available" or slot.available_seats <= 0:    # rule 2: seats left
        flash("This slot is full.", "danger")
    elif (Booking.query.join(Slot, Booking.slot_id == Slot.id)       # rule 3: no duplicates
          .filter(Booking.student_id == current_user.id, Slot.exam_id == exam.id,
                  Booking.status.in_(["Booked", "Completed"])).first()):
        flash("You already have a booking for this examination.", "danger")
    else:
        db.session.add(Booking(student_id=current_user.id, slot_id=slot.id))
        slot.available_seats -= 1
        if slot.available_seats == 0:
            slot.status = "Full"                                     # auto-close at capacity
        db.session.commit()
        flash("Slot booked successfully!", "success")
    return redirect(url_for("student.exam_slots", eid=exam.id))


@student_bp.route("/bookings/<int:bid>/cancel", methods=["POST"])
@role_required("student")
def cancel(bid):
    b = db.get_or_404(Booking, bid)
    if b.student_id != current_user.id:
        abort(403)
    exam = b.slot.exam
    if b.status != "Booked":
        flash("Only active bookings can be cancelled.", "warning")
    elif not exam.booking_end or date.today() > exam.booking_end:
        flash("The booking deadline has passed; you can no longer cancel.", "danger")
    else:
        b.status = "Cancelled"
        b.slot.available_seats += 1
        if b.slot.status == "Full":
            b.slot.status = "Available"
        db.session.commit()
        flash("Booking cancelled.", "success")
    return redirect(url_for("student.bookings"))


@student_bp.route("/bookings")
@role_required("student")
def bookings():
    active = [b for b in my_bookings() if b.status == "Booked"]
    return render_template("student/bookings.html", bookings=active, today=date.today())


@student_bp.route("/history")
@role_required("student")
def history():
    return render_template("student/history.html", bookings=my_bookings())


@student_bp.route("/results")
@role_required("student")
def results():
    rows = (Booking.query.join(Evaluation, Evaluation.booking_id == Booking.id)
            .filter(Booking.student_id == current_user.id, Evaluation.published.is_(True)).all())
    return render_template("student/results.html", rows=rows)
```

### `routes/api.py` (optional JSON API)
```python
from flask import Blueprint, jsonify
from flask_login import current_user
from models import Examination, Slot, Booking

api_bp = Blueprint("api", __name__)


def need_login():
    return jsonify({"error": "login required"}), 401


@api_bp.route("/exams")
def exams():
    if not current_user.is_authenticated:
        return need_login()
    return jsonify([{"id": e.id, "course": e.course.code, "name": e.name, "type": e.exam_type,
                     "max_marks": e.max_marks, "status": e.status} for e in Examination.query.all()])


@api_bp.route("/exams/<int:eid>")
def exam(eid):
    if not current_user.is_authenticated:
        return need_login()
    e = Examination.query.get(eid)
    if not e:
        return jsonify({"error": "not found"}), 404
    return jsonify({"id": e.id, "course": e.course.code, "name": e.name, "type": e.exam_type,
                    "duration": e.duration, "max_marks": e.max_marks, "status": e.status,
                    "booking_start": str(e.booking_start), "booking_end": str(e.booking_end)})


@api_bp.route("/slots")
def slots():
    if not current_user.is_authenticated:
        return need_login()
    return jsonify([{"id": s.id, "exam_id": s.exam_id, "examiner": s.examiner.name,
                     "date": str(s.date), "start": s.start_time.strftime("%H:%M"),
                     "end": s.end_time.strftime("%H:%M"), "available_seats": s.available_seats,
                     "status": s.status} for s in Slot.query.all()])


@api_bp.route("/bookings")
def bookings():
    """Admin sees all bookings; a student sees only their own."""
    if not current_user.is_authenticated:
        return need_login()
    if current_user.role == "admin":
        rows = Booking.query.all()
    elif current_user.role == "student":
        rows = Booking.query.filter_by(student_id=current_user.id).all()
    else:
        return jsonify({"error": "forbidden"}), 403
    return jsonify([{"id": b.id, "student": b.student.name, "slot_id": b.slot_id,
                     "status": b.status, "date": b.booking_date.isoformat()} for b in rows])
```

---

## 6. Static

### `static/css/style.css`
```css
body { min-height: 100vh; }
.card { border-radius: 10px; }
.stat-card h2 { font-weight: 700; }
table td, table th { vertical-align: middle; }
.navbar-nav { flex-wrap: wrap; }
```

---

## 7. Templates

### `templates/base.html`
```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}EMP - Examination Management Portal{% endblock %}</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
  <link href="{{ url_for('static', filename='css/style.css') }}" rel="stylesheet">
</head>
<body class="bg-light">
<nav class="navbar navbar-expand navbar-dark bg-dark mb-4">
  <div class="container">
    <a class="navbar-brand" href="{{ url_for('home') }}">EMP</a>
    <ul class="navbar-nav ms-auto">
      {% if current_user.is_authenticated %}
        {% if current_user.role == 'admin' %}
          <li class="nav-item"><a class="nav-link" href="{{ url_for('admin.dashboard') }}">Dashboard</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('admin.courses') }}">Courses</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('admin.exams') }}">Exams</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('admin.examiners') }}">Examiners</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('admin.students') }}">Students</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('admin.slots') }}">Slots</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('admin.bookings') }}">Bookings</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('admin.history') }}">History</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('admin.search') }}">Search</a></li>
        {% elif current_user.role == 'examiner' %}
          <li class="nav-item"><a class="nav-link" href="{{ url_for('examiner.dashboard') }}">Dashboard</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('examiner.slots') }}">My Slots</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('examiner.profile') }}">Profile</a></li>
        {% else %}
          <li class="nav-item"><a class="nav-link" href="{{ url_for('student.dashboard') }}">Dashboard</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('student.exams') }}">Exams</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('student.bookings') }}">My Bookings</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('student.history') }}">History</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('student.results') }}">Results</a></li>
          <li class="nav-item"><a class="nav-link" href="{{ url_for('student.profile') }}">Profile</a></li>
        {% endif %}
        <li class="nav-item"><a class="nav-link" href="{{ url_for('auth.logout') }}">Logout ({{ current_user.name }})</a></li>
      {% else %}
        <li class="nav-item"><a class="nav-link" href="{{ url_for('auth.login') }}">Login</a></li>
      {% endif %}
    </ul>
  </div>
</nav>

<div class="container pb-5">
  {% with messages = get_flashed_messages(with_categories=true) %}
    {% for category, msg in messages %}
      <div class="alert alert-{{ category }}">{{ msg }}</div>
    {% endfor %}
  {% endwith %}
  {% block content %}{% endblock %}
</div>
</body>
</html>
```

### `templates/home.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="text-center py-5">
  <h1 class="display-5">Examination Management Portal</h1>
  <p class="lead text-muted">Manage exam slots, bookings and evaluations in one place.</p>
  <a href="{{ url_for('auth.login') }}" class="btn btn-primary btn-lg m-1">Login</a>
  <a href="{{ url_for('auth.register_student') }}" class="btn btn-outline-primary btn-lg m-1">Student Register</a>
  <a href="{{ url_for('auth.register_examiner') }}" class="btn btn-outline-secondary btn-lg m-1">Examiner Register</a>
</div>
{% endblock %}
```

### `templates/error.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="text-center py-5">
  <h1 class="display-4">{{ code }}</h1>
  <p class="lead">{{ msg }}</p>
  <a href="{{ url_for('home') }}" class="btn btn-primary">Go Home</a>
</div>
{% endblock %}
```

### `templates/auth/login.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center">
  <div class="col-md-5">
    <div class="card shadow-sm"><div class="card-body">
      <h4 class="mb-3">Login</h4>
      <form method="post">
        <div class="mb-3"><label class="form-label">Email</label>
          <input type="email" name="email" class="form-control" required></div>
        <div class="mb-3"><label class="form-label">Password</label>
          <input type="password" name="password" class="form-control" required></div>
        <button class="btn btn-primary w-100">Login</button>
      </form>
      <hr>
      <a href="{{ url_for('auth.register_student') }}">Student register</a> |
      <a href="{{ url_for('auth.register_examiner') }}">Examiner register</a>
    </div></div>
  </div>
</div>
{% endblock %}
```

### `templates/auth/register_student.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-6">
  <div class="card shadow-sm"><div class="card-body">
    <h4 class="mb-3">Student Registration</h4>
    <form method="post">
      <div class="mb-3"><label class="form-label">Full name</label>
        <input name="name" class="form-control" required></div>
      <div class="mb-3"><label class="form-label">Roll number</label>
        <input name="roll_no" class="form-control"></div>
      <div class="mb-3"><label class="form-label">Email</label>
        <input type="email" name="email" class="form-control" required></div>
      <div class="mb-3"><label class="form-label">Phone</label>
        <input name="phone" class="form-control"></div>
      <div class="mb-3"><label class="form-label">Password (min 6 characters)</label>
        <input type="password" name="password" minlength="6" class="form-control" required></div>
      <button class="btn btn-primary w-100">Register</button>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

### `templates/auth/register_examiner.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-6">
  <div class="card shadow-sm"><div class="card-body">
    <h4 class="mb-3">Examiner Registration</h4>
    <p class="text-muted small">Your account must be approved by the Admin before you can log in.</p>
    <form method="post">
      <div class="mb-3"><label class="form-label">Full name</label>
        <input name="name" class="form-control" required></div>
      <div class="mb-3"><label class="form-label">Email</label>
        <input type="email" name="email" class="form-control" required></div>
      <div class="mb-3"><label class="form-label">Department</label>
        <input name="department" class="form-control" required></div>
      <div class="mb-3"><label class="form-label">Contact number</label>
        <input name="contact" class="form-control"></div>
      <div class="mb-3"><label class="form-label">Employee ID</label>
        <input name="employee_id" class="form-control"></div>
      <div class="mb-3"><label class="form-label">Password (min 6 characters)</label>
        <input type="password" name="password" minlength="6" class="form-control" required></div>
      <button class="btn btn-primary w-100">Register</button>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

---

### Admin templates

### `templates/admin/dashboard.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Admin Dashboard</h3>
{% if stats.pending %}
<div class="alert alert-warning">{{ stats.pending }} examiner(s) waiting for approval.
  <a href="{{ url_for('admin.examiners') }}">Review</a></div>
{% endif %}
<div class="row g-3 my-2">
  {% for label, value in [('Courses', stats.courses), ('Examinations', stats.exams),
                          ('Examiners', stats.examiners), ('Students', stats.students),
                          ('Examination Slots', stats.slots), ('Bookings', stats.bookings)] %}
  <div class="col-md-4">
    <div class="card text-center shadow-sm stat-card"><div class="card-body">
      <h6 class="text-muted">{{ label }}</h6><h2>{{ value }}</h2>
    </div></div>
  </div>
  {% endfor %}
</div>
{% endblock %}
```

### `templates/admin/courses.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="d-flex justify-content-between mb-3">
  <h3>Courses</h3>
  <a href="{{ url_for('admin.course_add') }}" class="btn btn-primary">+ Add Course</a>
</div>
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>ID</th><th>Code</th><th>Name</th><th>Description</th><th>Status</th><th>Actions</th></tr></thead>
  <tbody>
  {% for c in courses %}
  <tr>
    <td>{{ c.id }}</td><td>{{ c.code }}</td><td>{{ c.name }}</td><td>{{ c.description }}</td>
    <td><span class="badge bg-{{ 'success' if c.status == 'Active' else 'secondary' }}">{{ c.status }}</span></td>
    <td>
      <a class="btn btn-sm btn-outline-primary" href="{{ url_for('admin.course_edit', cid=c.id) }}">Edit</a>
      <form method="post" action="{{ url_for('admin.course_delete', cid=c.id) }}" class="d-inline">
        <button class="btn btn-sm btn-outline-danger">Delete</button>
      </form>
    </td>
  </tr>
  {% else %}
  <tr><td colspan="6" class="text-center text-muted">No courses yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

### `templates/admin/course_form.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-6">
  <div class="card shadow-sm"><div class="card-body">
    <h4>{{ 'Edit' if course else 'Add' }} Course</h4>
    <form method="post">
      <div class="mb-3"><label class="form-label">Course code</label>
        <input name="code" class="form-control" value="{{ course.code if course else '' }}" required></div>
      <div class="mb-3"><label class="form-label">Course name</label>
        <input name="name" class="form-control" value="{{ course.name if course else '' }}" required></div>
      <div class="mb-3"><label class="form-label">Description</label>
        <textarea name="description" class="form-control" rows="3">{{ course.description if course else '' }}</textarea></div>
      <div class="mb-3"><label class="form-label">Status</label>
        <select name="status" class="form-select">
          {% for s in ['Active', 'Inactive'] %}
          <option {{ 'selected' if course and course.status == s else '' }}>{{ s }}</option>
          {% endfor %}
        </select></div>
      <button class="btn btn-primary">Save</button>
      <a href="{{ url_for('admin.courses') }}" class="btn btn-secondary">Cancel</a>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

### `templates/admin/exams.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="d-flex justify-content-between mb-3">
  <h3>Examinations</h3>
  <a href="{{ url_for('admin.exam_add') }}" class="btn btn-primary">+ Add Examination</a>
</div>
<div class="table-responsive">
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>ID</th><th>Course</th><th>Examination</th><th>Timeline</th><th>Status</th><th>Controls</th></tr></thead>
  <tbody>
  {% for e in exams %}
  <tr>
    <td>{{ e.id }}</td>
    <td>{{ e.course.code }}</td>
    <td>{{ e.name }}<br><small class="text-muted">{{ e.exam_type }} · {{ e.duration }} min · {{ e.max_marks }} marks</small></td>
    <td><small>Slots: {{ e.slot_creation_start or '-' }} → {{ e.slot_creation_end or '-' }}<br>
        Booking: {{ e.booking_start or '-' }} → {{ e.booking_end or '-' }}</small></td>
    <td><span class="badge bg-secondary">{{ e.status }}</span><br>
        <small>Slot creation: {{ 'OPEN' if e.slot_creation_open else 'closed' }}<br>
        Booking: {{ 'OPEN' if e.booking_open else 'closed' }}</small></td>
    <td>
      <a class="btn btn-sm btn-outline-primary mb-1" href="{{ url_for('admin.exam_edit', eid=e.id) }}">Edit</a>
      <a class="btn btn-sm btn-outline-info mb-1" href="{{ url_for('admin.exam_examiners', eid=e.id) }}">Examiners</a>
      <a class="btn btn-sm btn-outline-secondary mb-1" href="{{ url_for('admin.rubrics', eid=e.id) }}">Rubrics</a>
      <form method="post" action="{{ url_for('admin.exam_toggle', eid=e.id, what='slots') }}" class="d-inline">
        <button class="btn btn-sm btn-warning mb-1">{{ 'Close' if e.slot_creation_open else 'Open' }} slot creation</button></form>
      <form method="post" action="{{ url_for('admin.exam_toggle', eid=e.id, what='booking') }}" class="d-inline">
        <button class="btn btn-sm btn-success mb-1">{{ 'Close' if e.booking_open else 'Open' }} booking</button></form>
      <form method="post" action="{{ url_for('admin.exam_delete', eid=e.id) }}" class="d-inline">
        <button class="btn btn-sm btn-outline-danger mb-1">Delete</button></form>
    </td>
  </tr>
  {% else %}
  <tr><td colspan="6" class="text-center text-muted">No examinations yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
</div>
{% endblock %}
```

### `templates/admin/exam_form.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-8">
  <div class="card shadow-sm"><div class="card-body">
    <h4>{{ 'Edit' if exam else 'Add' }} Examination</h4>
    <form method="post">
      <div class="row">
        <div class="col-md-6 mb-3"><label class="form-label">Course</label>
          <select name="course_id" class="form-select" required>
            {% for c in courses %}
            <option value="{{ c.id }}" {{ 'selected' if exam and exam.course_id == c.id else '' }}>{{ c.code }} - {{ c.name }}</option>
            {% endfor %}
          </select></div>
        <div class="col-md-6 mb-3"><label class="form-label">Examination name</label>
          <input name="name" class="form-control" value="{{ exam.name if exam else '' }}" required></div>
        <div class="col-md-4 mb-3"><label class="form-label">Type</label>
          <select name="exam_type" class="form-select">
            {% for t in types %}<option {{ 'selected' if exam and exam.exam_type == t else '' }}>{{ t }}</option>{% endfor %}
          </select></div>
        <div class="col-md-4 mb-3"><label class="form-label">Duration (minutes)</label>
          <input type="number" min="1" name="duration" class="form-control" value="{{ exam.duration if exam else 30 }}" required></div>
        <div class="col-md-4 mb-3"><label class="form-label">Maximum marks</label>
          <input type="number" min="1" name="max_marks" class="form-control" value="{{ exam.max_marks if exam else 100 }}" required></div>
        <div class="col-md-6 mb-3"><label class="form-label">Slot creation start</label>
          <input type="date" name="slot_creation_start" class="form-control" value="{{ exam.slot_creation_start or '' if exam else '' }}"></div>
        <div class="col-md-6 mb-3"><label class="form-label">Slot creation end</label>
          <input type="date" name="slot_creation_end" class="form-control" value="{{ exam.slot_creation_end or '' if exam else '' }}"></div>
        <div class="col-md-6 mb-3"><label class="form-label">Booking start</label>
          <input type="date" name="booking_start" class="form-control" value="{{ exam.booking_start or '' if exam else '' }}"></div>
        <div class="col-md-6 mb-3"><label class="form-label">Booking end</label>
          <input type="date" name="booking_end" class="form-control" value="{{ exam.booking_end or '' if exam else '' }}"></div>
        <div class="col-md-6 mb-3"><label class="form-label">Status</label>
          <select name="status" class="form-select">
            {% for s in statuses %}<option {{ 'selected' if exam and exam.status == s else '' }}>{{ s }}</option>{% endfor %}
          </select></div>
      </div>
      <button class="btn btn-primary">Save</button>
      <a href="{{ url_for('admin.exams') }}" class="btn btn-secondary">Cancel</a>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

### `templates/admin/exam_examiners.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Examiners for: {{ exam.name }}</h3>
<div class="row">
  <div class="col-md-7">
    <table class="table table-bordered bg-white">
      <thead><tr><th>ID</th><th>Name</th><th>Email</th><th></th></tr></thead>
      <tbody>
      {% for a in exam.assignments %}
      <tr><td>{{ a.examiner.id }}</td><td>{{ a.examiner.name }}</td><td>{{ a.examiner.email }}</td>
        <td><form method="post" action="{{ url_for('admin.exam_examiner_remove', eid=exam.id, aid=a.id) }}">
          <button class="btn btn-sm btn-outline-danger">Remove</button></form></td></tr>
      {% else %}
      <tr><td colspan="4" class="text-muted text-center">No examiner assigned.</td></tr>
      {% endfor %}
      </tbody>
    </table>
  </div>
  <div class="col-md-5">
    <div class="card shadow-sm"><div class="card-body">
      <h6>Assign an approved examiner</h6>
      <form method="post">
        <select name="examiner_id" class="form-select mb-2" required>
          {% for u in available %}<option value="{{ u.id }}">{{ u.name }} ({{ u.email }})</option>{% endfor %}
        </select>
        <button class="btn btn-primary" {{ 'disabled' if not available else '' }}>Assign</button>
      </form>
    </div></div>
  </div>
</div>
<a href="{{ url_for('admin.exams') }}" class="btn btn-secondary mt-3">Back</a>
{% endblock %}
```

### `templates/admin/rubrics.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Rubrics: {{ exam.name }}</h3>
<p class="text-muted">Exam maximum marks: {{ exam.max_marks }} · Sum of criteria marks: {{ '%g' % total }}</p>
<div class="row">
  <div class="col-md-8">
    <table class="table table-bordered bg-white">
      <thead><tr><th>Criterion</th><th>Max marks</th><th>Weightage</th><th>Description</th><th></th></tr></thead>
      <tbody>
      {% for r in exam.rubrics %}
      <tr><td>{{ r.criterion }}</td><td>{{ '%g' % r.max_marks }}</td><td>{{ '%g' % r.weightage }}</td><td>{{ r.description }}</td>
        <td>
          <a class="btn btn-sm btn-outline-primary" href="{{ url_for('admin.rubric_edit', rid=r.id) }}">Edit</a>
          <form method="post" action="{{ url_for('admin.rubric_delete', rid=r.id) }}" class="d-inline">
            <button class="btn btn-sm btn-outline-danger">Delete</button></form>
        </td></tr>
      {% else %}
      <tr><td colspan="5" class="text-center text-muted">No criteria yet.</td></tr>
      {% endfor %}
      </tbody>
    </table>
  </div>
  <div class="col-md-4">
    <div class="card shadow-sm"><div class="card-body">
      <h6>Add criterion</h6>
      <form method="post">
        <input name="criterion" class="form-control mb-2" placeholder="Criterion name" required>
        <input type="number" step="0.5" min="0.5" name="max_marks" class="form-control mb-2" placeholder="Max marks" required>
        <input type="number" step="0.1" min="0.1" name="weightage" class="form-control mb-2" placeholder="Weightage (default 1)" value="1">
        <textarea name="description" class="form-control mb-2" placeholder="Description"></textarea>
        <button class="btn btn-primary">Add</button>
      </form>
    </div></div>
  </div>
</div>
<a href="{{ url_for('admin.exams') }}" class="btn btn-secondary mt-3">Back</a>
{% endblock %}
```

### `templates/admin/rubric_form.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-6">
  <div class="card shadow-sm"><div class="card-body">
    <h4>Edit Rubric Criterion</h4>
    <form method="post">
      <div class="mb-3"><label class="form-label">Criterion</label>
        <input name="criterion" class="form-control" value="{{ rubric.criterion }}" required></div>
      <div class="mb-3"><label class="form-label">Max marks</label>
        <input type="number" step="0.5" min="0.5" name="max_marks" class="form-control" value="{{ '%g' % rubric.max_marks }}" required></div>
      <div class="mb-3"><label class="form-label">Weightage</label>
        <input type="number" step="0.1" min="0.1" name="weightage" class="form-control" value="{{ '%g' % rubric.weightage }}"></div>
      <div class="mb-3"><label class="form-label">Description</label>
        <textarea name="description" class="form-control">{{ rubric.description or '' }}</textarea></div>
      <button class="btn btn-primary">Save</button>
      <a href="{{ url_for('admin.rubrics', eid=rubric.exam_id) }}" class="btn btn-secondary">Cancel</a>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

### `templates/admin/examiners.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="d-flex justify-content-between mb-3">
  <h3>Examiners</h3>
  <a href="{{ url_for('admin.examiner_add') }}" class="btn btn-primary">+ Add Examiner</a>
</div>
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>ID</th><th>Name</th><th>Email</th><th>Department</th><th>Contact</th><th>Status</th><th>Actions</th></tr></thead>
  <tbody>
  {% for p in profiles %}
  <tr>
    <td>{{ p.user.id }}</td><td>{{ p.user.name }}</td><td>{{ p.user.email }}</td>
    <td>{{ p.department }}</td><td>{{ p.contact }}</td>
    <td><span class="badge bg-{{ {'Approved': 'success', 'Pending': 'warning', 'Deactivated': 'danger'}[p.status] }}">{{ p.status }}</span></td>
    <td>
      {% if p.status != 'Approved' %}
      <form method="post" action="{{ url_for('admin.examiner_action', pid=p.id, action='approve') }}" class="d-inline">
        <button class="btn btn-sm btn-success">Approve</button></form>
      {% endif %}
      {% if p.status == 'Approved' %}
      <form method="post" action="{{ url_for('admin.examiner_action', pid=p.id, action='deactivate') }}" class="d-inline">
        <button class="btn btn-sm btn-outline-danger">Deactivate</button></form>
      {% endif %}
    </td>
  </tr>
  {% else %}
  <tr><td colspan="7" class="text-center text-muted">No examiners yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

### `templates/admin/examiner_form.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-6">
  <div class="card shadow-sm"><div class="card-body">
    <h4>Add Examiner</h4>
    <form method="post">
      <div class="mb-3"><label class="form-label">Name</label><input name="name" class="form-control" required></div>
      <div class="mb-3"><label class="form-label">Email</label><input type="email" name="email" class="form-control" required></div>
      <div class="mb-3"><label class="form-label">Department</label><input name="department" class="form-control"></div>
      <div class="mb-3"><label class="form-label">Contact</label><input name="contact" class="form-control"></div>
      <div class="mb-3"><label class="form-label">Employee ID</label><input name="employee_id" class="form-control"></div>
      <div class="mb-3"><label class="form-label">Password</label><input type="password" minlength="6" name="password" class="form-control" required></div>
      <button class="btn btn-primary">Add (auto-approved)</button>
      <a href="{{ url_for('admin.examiners') }}" class="btn btn-secondary">Cancel</a>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

### `templates/admin/students.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Students</h3>
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>ID</th><th>Name</th><th>Roll no</th><th>Email</th><th>Phone</th><th>Bookings</th></tr></thead>
  <tbody>
  {% for s in students %}
  <tr><td>{{ s.id }}</td><td>{{ s.name }}</td><td>{{ s.roll_no }}</td><td>{{ s.email }}</td>
      <td>{{ s.phone }}</td><td>{{ s.bookings|length }}</td></tr>
  {% else %}
  <tr><td colspan="6" class="text-center text-muted">No students yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

### `templates/admin/slots.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>All Examination Slots</h3>
<div class="table-responsive">
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>ID</th><th>Exam</th><th>Date</th><th>Time</th><th>Examiner</th><th>Seats</th><th>Status</th><th>Change examiner</th></tr></thead>
  <tbody>
  {% for s in slots %}
  <tr>
    <td>{{ s.id }}</td><td>{{ s.exam.name }}</td>
    <td>{{ s.date.strftime('%d %b %Y') }}</td>
    <td>{{ s.start_time.strftime('%H:%M') }} - {{ s.end_time.strftime('%H:%M') }}</td>
    <td>{{ s.examiner.name }}</td>
    <td>{{ s.available_seats }} / {{ s.capacity }}</td>
    <td><span class="badge bg-secondary">{{ s.status }}</span></td>
    <td>
      <form method="post" action="{{ url_for('admin.change_examiner', sid=s.id) }}" class="d-flex gap-1">
        <select name="examiner_id" class="form-select form-select-sm">
          {% for u in examiners %}<option value="{{ u.id }}" {{ 'selected' if u.id == s.examiner_id else '' }}>{{ u.name }}</option>{% endfor %}
        </select>
        <button class="btn btn-sm btn-outline-primary">Save</button>
      </form>
    </td>
  </tr>
  {% else %}
  <tr><td colspan="8" class="text-center text-muted">No slots yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
</div>
{% endblock %}
```

### `templates/admin/bookings.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>All Bookings</h3>
<form method="get" class="row g-2 mb-3">
  <div class="col-auto">
    <select name="status" class="form-select">
      <option value="">All statuses</option>
      {% for s in ['Booked', 'Cancelled', 'Completed'] %}
      <option {{ 'selected' if status == s else '' }}>{{ s }}</option>{% endfor %}
    </select></div>
  <div class="col-auto"><button class="btn btn-secondary">Filter</button></div>
</form>
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>ID</th><th>Student</th><th>Exam</th><th>Slot</th><th>Booked on</th><th>Status</th><th></th></tr></thead>
  <tbody>
  {% for b in bookings %}
  <tr>
    <td>{{ b.id }}</td><td>{{ b.student.name }} (#{{ b.student.id }})</td><td>{{ b.slot.exam.name }}</td>
    <td>{{ b.slot.date.strftime('%d %b') }} {{ b.slot.start_time.strftime('%H:%M') }}</td>
    <td>{{ b.booking_date.strftime('%d %b %Y %H:%M') }}</td>
    <td><span class="badge bg-{{ {'Booked': 'primary', 'Cancelled': 'secondary', 'Completed': 'success'}[b.status] }}">{{ b.status }}</span></td>
    <td>{% if b.status == 'Booked' %}
      <a class="btn btn-sm btn-outline-primary" href="{{ url_for('admin.reschedule', bid=b.id) }}">Reschedule</a>{% endif %}</td>
  </tr>
  {% else %}
  <tr><td colspan="7" class="text-center text-muted">No bookings.</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

### `templates/admin/reschedule.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-6">
  <div class="card shadow-sm"><div class="card-body">
    <h4>Reschedule Booking #{{ booking.id }}</h4>
    <p>Student: <b>{{ booking.student.name }}</b><br>
       Exam: <b>{{ booking.slot.exam.name }}</b><br>
       Current slot: {{ booking.slot.date.strftime('%d %b %Y') }}, {{ booking.slot.start_time.strftime('%H:%M') }}</p>
    <form method="post">
      <label class="form-label">Move to slot</label>
      <select name="slot_id" class="form-select mb-3" required>
        {% for s in options %}
        <option value="{{ s.id }}">{{ s.date.strftime('%d %b %Y') }} {{ s.start_time.strftime('%H:%M') }}-{{ s.end_time.strftime('%H:%M') }}
          ({{ s.examiner.name }}, {{ s.available_seats }} seats)</option>
        {% endfor %}
      </select>
      <button class="btn btn-primary" {{ 'disabled' if not options else '' }}>Reschedule</button>
      <a href="{{ url_for('admin.bookings') }}" class="btn btn-secondary">Cancel</a>
    </form>
    {% if not options %}<p class="text-muted mt-2">No other available slot for this examination.</p>{% endif %}
  </div></div>
</div></div>
{% endblock %}
```

### `templates/admin/search.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Search</h3>
<form method="get" class="row g-2 mb-4">
  <div class="col-md-3">
    <select name="kind" class="form-select">
      {% for k in ['students', 'examiners', 'exams', 'bookings'] %}
      <option value="{{ k }}" {{ 'selected' if kind == k else '' }}>{{ k|capitalize }}</option>{% endfor %}
    </select></div>
  <div class="col-md-6"><input name="q" value="{{ q }}" class="form-control" placeholder="Name or ID" required></div>
  <div class="col-md-3"><button class="btn btn-primary">Search</button></div>
</form>

{% if q %}
  <p class="text-muted">{{ results|length }} result(s) for "{{ q }}" in {{ kind }}</p>
  <table class="table table-bordered bg-white">
    {% if kind in ['students', 'examiners'] %}
    <thead><tr><th>ID</th><th>Name</th><th>Email</th><th>Phone</th></tr></thead>
    {% for u in results %}<tr><td>{{ u.id }}</td><td>{{ u.name }}</td><td>{{ u.email }}</td><td>{{ u.phone }}</td></tr>{% endfor %}
    {% elif kind == 'exams' %}
    <thead><tr><th>ID</th><th>Name</th><th>Type</th><th>Status</th></tr></thead>
    {% for e in results %}<tr><td>{{ e.id }}</td><td>{{ e.name }}</td><td>{{ e.exam_type }}</td><td>{{ e.status }}</td></tr>{% endfor %}
    {% else %}
    <thead><tr><th>ID</th><th>Student</th><th>Exam</th><th>Status</th></tr></thead>
    {% for b in results %}<tr><td>{{ b.id }}</td><td>{{ b.student.name }}</td><td>{{ b.slot.exam.name }}</td><td>{{ b.status }}</td></tr>{% endfor %}
    {% endif %}
  </table>
{% endif %}
{% endblock %}
```

### `templates/admin/history.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Complete Examination & Booking History</h3>
<div class="table-responsive">
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>Booking</th><th>Student</th><th>Exam</th><th>Date</th><th>Examiner</th><th>Status</th><th>Marks</th><th>Remarks</th></tr></thead>
  <tbody>
  {% for b in rows %}
  <tr>
    <td>#{{ b.id }}</td><td>{{ b.student.name }}</td><td>{{ b.slot.exam.name }}</td>
    <td>{{ b.slot.date.strftime('%d %b %Y') }}</td><td>{{ b.slot.examiner.name }}</td>
    <td>{{ b.status }}</td>
    <td>{% if b.evaluation %}{{ '%g' % b.evaluation.total_marks }} / {{ b.slot.exam.max_marks }}
        {% if not b.evaluation.published %}<small class="text-muted">(unpublished)</small>{% endif %}{% else %}-{% endif %}</td>
    <td>{{ b.evaluation.remarks if b.evaluation else '' }}</td>
  </tr>
  {% else %}
  <tr><td colspan="8" class="text-center text-muted">No history yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
</div>
{% endblock %}
```

---

### Examiner templates

### `templates/examiner/dashboard.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Examiner Dashboard</h3>
<div class="row g-3 my-2">
  <div class="col-md-3"><div class="card text-center shadow-sm stat-card"><div class="card-body"><h6 class="text-muted">Assigned exams</h6><h2>{{ exams|length }}</h2></div></div></div>
  <div class="col-md-3"><div class="card text-center shadow-sm stat-card"><div class="card-body"><h6 class="text-muted">My slots</h6><h2>{{ slots|length }}</h2></div></div></div>
  <div class="col-md-3"><div class="card text-center shadow-sm stat-card"><div class="card-body"><h6 class="text-muted">Students booked</h6><h2>{{ booked }}</h2></div></div></div>
  <div class="col-md-3"><div class="card text-center shadow-sm stat-card"><div class="card-body"><h6 class="text-muted">Pending evaluations</h6><h2>{{ pending }}</h2></div></div></div>
</div>

<h5 class="mt-4">Assigned Examinations</h5>
<table class="table table-bordered bg-white">
  <thead><tr><th>Exam</th><th>Type</th><th>Slot creation period</th><th>Status</th><th></th></tr></thead>
  <tbody>
  {% for e in exams %}
  <tr>
    <td>{{ e.course.code }} - {{ e.name }}</td><td>{{ e.exam_type }}</td>
    <td>{{ e.slot_creation_start or '-' }} → {{ e.slot_creation_end or '-' }}
        {% if e.slot_creation_allowed() %}<span class="badge bg-success">Open</span>{% else %}<span class="badge bg-secondary">Closed</span>{% endif %}</td>
    <td>{{ e.status }}</td>
    <td>{% if e.slot_creation_allowed() %}
      <a class="btn btn-sm btn-primary" href="{{ url_for('examiner.slot_add', eid=e.id) }}">+ Create slot</a>
      {% else %}<button class="btn btn-sm btn-outline-secondary" disabled>Create slot</button>{% endif %}</td>
  </tr>
  {% else %}
  <tr><td colspan="5" class="text-center text-muted">No examinations assigned yet.</td></tr>
  {% endfor %}
  </tbody>
</table>

<h5 class="mt-4">My Slots <a class="btn btn-sm btn-outline-primary ms-2" href="{{ url_for('examiner.slots') }}">Manage</a></h5>
<table class="table table-bordered bg-white">
  <thead><tr><th>Exam</th><th>Date</th><th>Time</th><th>Seats left</th><th>Status</th><th></th></tr></thead>
  <tbody>
  {% for s in slots %}
  <tr><td>{{ s.exam.name }}</td><td>{{ s.date.strftime('%d %b %Y') }}</td>
      <td>{{ s.start_time.strftime('%H:%M') }}-{{ s.end_time.strftime('%H:%M') }}</td>
      <td>{{ s.available_seats }}/{{ s.capacity }}</td><td>{{ s.status }}</td>
      <td><a class="btn btn-sm btn-outline-info" href="{{ url_for('examiner.slot_students', sid=s.id) }}">Students</a></td></tr>
  {% else %}
  <tr><td colspan="6" class="text-center text-muted">No slots created.</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

### `templates/examiner/slots.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>My Slots</h3>
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>ID</th><th>Exam</th><th>Date</th><th>Time</th><th>Seats</th><th>Status</th><th>Actions</th></tr></thead>
  <tbody>
  {% for s in slots %}
  <tr>
    <td>{{ s.id }}</td><td>{{ s.exam.name }}</td><td>{{ s.date.strftime('%d %b %Y') }}</td>
    <td>{{ s.start_time.strftime('%H:%M') }}-{{ s.end_time.strftime('%H:%M') }}</td>
    <td>{{ s.available_seats }}/{{ s.capacity }}</td><td>{{ s.status }}</td>
    <td>
      <a class="btn btn-sm btn-outline-info" href="{{ url_for('examiner.slot_students', sid=s.id) }}">Students</a>
      {% if not s.exam.booking_started() and not s.bookings %}
      <a class="btn btn-sm btn-outline-primary" href="{{ url_for('examiner.slot_edit', sid=s.id) }}">Edit</a>
      <form method="post" action="{{ url_for('examiner.slot_delete', sid=s.id) }}" class="d-inline">
        <button class="btn btn-sm btn-outline-danger">Delete</button></form>
      {% endif %}
    </td>
  </tr>
  {% else %}
  <tr><td colspan="7" class="text-center text-muted">No slots.</td></tr>
  {% endfor %}
  </tbody>
</table>
<p class="text-muted small">Edit/Delete is available only before booking starts.</p>
{% endblock %}
```

### `templates/examiner/slot_form.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-6">
  <div class="card shadow-sm"><div class="card-body">
    <h4>{{ 'Edit' if slot else 'Create' }} Slot</h4>
    <p class="text-muted">Examination: <b>{{ exam.name }}</b> ({{ exam.duration }} min)</p>
    <form method="post">
      <div class="mb-3"><label class="form-label">Date</label>
        <input type="date" name="date" class="form-control" value="{{ slot.date if slot else '' }}" required></div>
      <div class="row">
        <div class="col mb-3"><label class="form-label">Start time</label>
          <input type="time" name="start_time" class="form-control" value="{{ slot.start_time.strftime('%H:%M') if slot else '' }}" required></div>
        <div class="col mb-3"><label class="form-label">End time</label>
          <input type="time" name="end_time" class="form-control" value="{{ slot.end_time.strftime('%H:%M') if slot else '' }}" required></div>
      </div>
      <div class="mb-3"><label class="form-label">Maximum student capacity</label>
        <input type="number" min="1" name="capacity" class="form-control" value="{{ slot.capacity if slot else 1 }}" required></div>
      <button class="btn btn-primary">Save</button>
      <a href="{{ url_for('examiner.slots') }}" class="btn btn-secondary">Cancel</a>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

### `templates/examiner/slot_students.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Students in Slot #{{ slot.id }}</h3>
<p class="text-muted">{{ slot.exam.name }} · {{ slot.date.strftime('%d %b %Y') }} · {{ slot.start_time.strftime('%H:%M') }}-{{ slot.end_time.strftime('%H:%M') }}</p>
<table class="table table-bordered bg-white">
  <thead><tr><th>Booking</th><th>Student</th><th>Roll no</th><th>Status</th><th>Marks</th><th></th></tr></thead>
  <tbody>
  {% for b in bookings %}
  <tr>
    <td>#{{ b.id }}</td><td>{{ b.student.name }}</td><td>{{ b.student.roll_no }}</td><td>{{ b.status }}</td>
    <td>{% if b.evaluation %}{{ '%g' % b.evaluation.total_marks }} / {{ slot.exam.max_marks }}{% else %}-{% endif %}</td>
    <td>{% if b.status != 'Cancelled' %}
      <a class="btn btn-sm btn-primary" href="{{ url_for('examiner.evaluate', bid=b.id) }}">{{ 'Edit evaluation' if b.evaluation else 'Evaluate' }}</a>
      {% endif %}</td>
  </tr>
  {% else %}
  <tr><td colspan="6" class="text-center text-muted">No students booked yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
<a href="{{ url_for('examiner.slots') }}" class="btn btn-secondary">Back</a>
{% endblock %}
```

### `templates/examiner/evaluate.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-8">
  <div class="card shadow-sm"><div class="card-body">
    <h4>Evaluate: {{ booking.student.name }}</h4>
    <p class="text-muted">{{ booking.slot.exam.name }} · max marks {{ booking.slot.exam.max_marks }}</p>
    <form method="post">
      <table class="table">
        <thead><tr><th>Criterion</th><th>Max</th><th>Weight</th><th>Marks</th></tr></thead>
        <tbody>
        {% for r in rubrics %}
        <tr>
          <td>{{ r.criterion }}<br><small class="text-muted">{{ r.description }}</small></td>
          <td>{{ '%g' % r.max_marks }}</td><td>{{ '%g' % r.weightage }}</td>
          <td style="width:140px"><input type="number" step="0.5" min="0" max="{{ r.max_marks }}"
              name="r_{{ r.id }}" value="{{ existing.get(r.id, '') }}" class="form-control" required></td>
        </tr>
        {% endfor %}
        </tbody>
      </table>
      <div class="mb-3"><label class="form-label">Remarks / feedback</label>
        <textarea name="remarks" rows="3" class="form-control">{{ ev.remarks if ev and ev.remarks else '' }}</textarea></div>
      <div class="form-check mb-3">
        <input class="form-check-input" type="checkbox" name="publish" id="pub" {{ 'checked' if ev and ev.published else '' }}>
        <label class="form-check-label" for="pub">Publish result to the student</label>
      </div>
      <button class="btn btn-primary">Submit marks</button>
      <a href="{{ url_for('examiner.slot_students', sid=booking.slot_id) }}" class="btn btn-secondary">Back</a>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

### `templates/examiner/profile.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-6">
  <div class="card shadow-sm"><div class="card-body">
    <h4>My Profile</h4>
    <p class="text-muted">Email: {{ current_user.email }} · Status: {{ p.status }}</p>
    <form method="post">
      <div class="mb-3"><label class="form-label">Name</label>
        <input name="name" class="form-control" value="{{ current_user.name }}" required></div>
      <div class="mb-3"><label class="form-label">Department</label>
        <input name="department" class="form-control" value="{{ p.department or '' }}"></div>
      <div class="mb-3"><label class="form-label">Contact</label>
        <input name="contact" class="form-control" value="{{ p.contact or '' }}"></div>
      <div class="mb-3"><label class="form-label">Employee ID</label>
        <input name="employee_id" class="form-control" value="{{ p.employee_id or '' }}"></div>
      <div class="mb-3"><label class="form-label">New password (leave blank to keep)</label>
        <input type="password" name="password" class="form-control"></div>
      <button class="btn btn-primary">Update</button>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

---

### Student templates

### `templates/student/dashboard.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Welcome, {{ current_user.name }}</h3>
<div class="row g-3 my-2">
  <div class="col-md-3"><div class="card text-center shadow-sm stat-card"><div class="card-body"><h6 class="text-muted">Exams open for booking</h6><h2>{{ open_exams|length }}</h2></div></div></div>
  <div class="col-md-3"><div class="card text-center shadow-sm stat-card"><div class="card-body"><h6 class="text-muted">Available slots</h6><h2>{{ open_slots|length }}</h2></div></div></div>
  <div class="col-md-3"><div class="card text-center shadow-sm stat-card"><div class="card-body"><h6 class="text-muted">Upcoming</h6><h2>{{ upcoming|length }}</h2></div></div></div>
  <div class="col-md-3"><div class="card text-center shadow-sm stat-card"><div class="card-body"><h6 class="text-muted">Results published</h6><h2>{{ results|length }}</h2></div></div></div>
</div>

<h5 class="mt-4">Available Examinations</h5>
<table class="table table-bordered bg-white">
  <thead><tr><th>Exam</th><th>Type</th><th>Booking until</th><th></th></tr></thead>
  <tbody>
  {% for e in open_exams %}
  <tr><td>{{ e.course.code }} - {{ e.name }}</td><td>{{ e.exam_type }}</td><td>{{ e.booking_end }}</td>
      <td><a class="btn btn-sm btn-primary" href="{{ url_for('student.exam_slots', eid=e.id) }}">View slots</a></td></tr>
  {% else %}
  <tr><td colspan="4" class="text-center text-muted">No examination is open for booking right now.</td></tr>
  {% endfor %}
  </tbody>
</table>

<h5 class="mt-4">Upcoming Examinations (my booked slots)</h5>
<table class="table table-bordered bg-white">
  <thead><tr><th>Exam</th><th>Date</th><th>Time</th><th>Examiner</th></tr></thead>
  <tbody>
  {% for b in upcoming %}
  <tr><td>{{ b.slot.exam.name }}</td><td>{{ b.slot.date.strftime('%d %b %Y') }}</td>
      <td>{{ b.slot.start_time.strftime('%H:%M') }}-{{ b.slot.end_time.strftime('%H:%M') }}</td><td>{{ b.slot.examiner.name }}</td></tr>
  {% else %}
  <tr><td colspan="4" class="text-center text-muted">Nothing upcoming.</td></tr>
  {% endfor %}
  </tbody>
</table>

<h5 class="mt-4">Published Results</h5>
<table class="table table-bordered bg-white">
  <thead><tr><th>Exam</th><th>Marks</th><th>Remarks</th></tr></thead>
  <tbody>
  {% for b in results %}
  <tr><td>{{ b.slot.exam.name }}</td><td>{{ '%g' % b.evaluation.total_marks }} / {{ b.slot.exam.max_marks }}</td><td>{{ b.evaluation.remarks }}</td></tr>
  {% else %}
  <tr><td colspan="3" class="text-center text-muted">No published results yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

### `templates/student/exams.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>Examinations</h3>
<form method="get" class="row g-2 mb-3">
  <div class="col-md-5"><input name="q" value="{{ q }}" class="form-control" placeholder="Search by exam, course name or code"></div>
  <div class="col-md-3">
    <select name="type" class="form-select">
      <option value="">All types</option>
      {% for t in types %}<option {{ 'selected' if etype == t else '' }}>{{ t }}</option>{% endfor %}
    </select></div>
  <div class="col-md-2"><button class="btn btn-primary">Search</button></div>
  <div class="col-md-2"><a href="{{ url_for('student.exams') }}" class="btn btn-outline-secondary">Reset</a></div>
</form>
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>Course</th><th>Exam</th><th>Type</th><th>Duration</th><th>Marks</th><th>Booking window</th><th>Status</th><th></th></tr></thead>
  <tbody>
  {% for e in exams %}
  <tr>
    <td>{{ e.course.code }}</td><td>{{ e.name }}</td><td>{{ e.exam_type }}</td>
    <td>{{ e.duration }} min</td><td>{{ e.max_marks }}</td>
    <td>{{ e.booking_start or '-' }} → {{ e.booking_end or '-' }}</td>
    <td>{% if e.booking_allowed() %}<span class="badge bg-success">Booking open</span>{% else %}<span class="badge bg-secondary">{{ e.status }}</span>{% endif %}</td>
    <td><a class="btn btn-sm btn-outline-primary" href="{{ url_for('student.exam_slots', eid=e.id) }}">View slots</a></td>
  </tr>
  {% else %}
  <tr><td colspan="8" class="text-center text-muted">No examinations found.</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

### `templates/student/exam_slots.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>{{ exam.name }} — Slots</h3>
<p class="text-muted">{{ exam.course.code }} · {{ exam.exam_type }} · {{ exam.duration }} min ·
  Booking: {{ exam.booking_start or '-' }} → {{ exam.booking_end or '-' }}</p>

{% if my %}
  <div class="alert alert-info">You already have a booking for this examination:
    {{ my.slot.date.strftime('%d %b %Y') }}, {{ my.slot.start_time.strftime('%H:%M') }} ({{ my.status }}).</div>
{% elif not allowed %}
  <div class="alert alert-warning">Booking is currently closed for this examination.</div>
{% endif %}

<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>Date</th><th>Time</th><th>Examiner</th><th>Seats left</th><th>Status</th><th></th></tr></thead>
  <tbody>
  {% for s in slots %}
  <tr>
    <td>{{ s.date.strftime('%d %b %Y') }}</td>
    <td>{{ s.start_time.strftime('%H:%M') }}-{{ s.end_time.strftime('%H:%M') }}</td>
    <td>{{ s.examiner.name }}</td><td>{{ s.available_seats }} / {{ s.capacity }}</td>
    <td><span class="badge bg-{{ 'success' if s.status == 'Available' else 'secondary' }}">{{ s.status }}</span></td>
    <td>
      {% if allowed and not my and s.status == 'Available' %}
      <form method="post" action="{{ url_for('student.book', sid=s.id) }}">
        <button class="btn btn-sm btn-primary">Book</button></form>
      {% endif %}
    </td>
  </tr>
  {% else %}
  <tr><td colspan="6" class="text-center text-muted">No slots available yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
<a href="{{ url_for('student.exams') }}" class="btn btn-secondary">Back</a>
{% endblock %}
```

### `templates/student/bookings.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>My Active Bookings</h3>
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>Booking</th><th>Exam</th><th>Date</th><th>Time</th><th>Examiner</th><th>Cancel until</th><th>Status</th><th></th></tr></thead>
  <tbody>
  {% for b in bookings %}
  <tr>
    <td>#{{ b.id }}</td><td>{{ b.slot.exam.name }}</td><td>{{ b.slot.date.strftime('%d %b %Y') }}</td>
    <td>{{ b.slot.start_time.strftime('%H:%M') }}-{{ b.slot.end_time.strftime('%H:%M') }}</td>
    <td>{{ b.slot.examiner.name }}</td><td>{{ b.slot.exam.booking_end }}</td><td>{{ b.status }}</td>
    <td>{% if b.slot.exam.booking_end and today <= b.slot.exam.booking_end %}
      <form method="post" action="{{ url_for('student.cancel', bid=b.id) }}">
        <button class="btn btn-sm btn-outline-danger">Cancel</button></form>
      {% else %}<small class="text-muted">Deadline passed</small>{% endif %}</td>
  </tr>
  {% else %}
  <tr><td colspan="8" class="text-center text-muted">No active bookings.</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

### `templates/student/history.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>My Booking & Examination History</h3>
<table class="table table-striped table-bordered bg-white">
  <thead><tr><th>Booking</th><th>Exam</th><th>Slot date</th><th>Booked on</th><th>Status</th><th>Result</th></tr></thead>
  <tbody>
  {% for b in bookings %}
  <tr>
    <td>#{{ b.id }}</td><td>{{ b.slot.exam.name }}</td><td>{{ b.slot.date.strftime('%d %b %Y') }}</td>
    <td>{{ b.booking_date.strftime('%d %b %Y %H:%M') }}</td>
    <td><span class="badge bg-{{ {'Booked': 'primary', 'Cancelled': 'secondary', 'Completed': 'success'}[b.status] }}">{{ b.status }}</span></td>
    <td>{% if b.evaluation and b.evaluation.published %}{{ '%g' % b.evaluation.total_marks }} / {{ b.slot.exam.max_marks }}{% else %}-{% endif %}</td>
  </tr>
  {% else %}
  <tr><td colspan="6" class="text-center text-muted">No history yet.</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

### `templates/student/results.html`
```html
{% extends "base.html" %}
{% block content %}
<h3>My Results & Feedback</h3>
{% for b in rows %}
<div class="card shadow-sm mb-3"><div class="card-body">
  <h5>{{ b.slot.exam.name }}
    <span class="float-end">{{ '%g' % b.evaluation.total_marks }} / {{ b.slot.exam.max_marks }}</span></h5>
  <p class="text-muted mb-2">Examiner: {{ b.slot.examiner.name }} · {{ b.slot.date.strftime('%d %b %Y') }}</p>
  <table class="table table-sm">
    <thead><tr><th>Criterion</th><th>Marks</th></tr></thead>
    <tbody>
    {% for s in b.evaluation.scores %}
    <tr><td>{{ s.rubric.criterion }}</td><td>{{ '%g' % s.marks }} / {{ '%g' % s.rubric.max_marks }}</td></tr>
    {% endfor %}
    </tbody>
  </table>
  <p><b>Remarks:</b> {{ b.evaluation.remarks or 'No remarks.' }}</p>
</div></div>
{% else %}
<div class="alert alert-info">No results have been published yet.</div>
{% endfor %}
{% endblock %}
```

### `templates/student/profile.html`
```html
{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center"><div class="col-md-6">
  <div class="card shadow-sm"><div class="card-body">
    <h4>My Profile</h4>
    <p class="text-muted">Email: {{ current_user.email }}</p>
    <form method="post">
      <div class="mb-3"><label class="form-label">Name</label>
        <input name="name" class="form-control" value="{{ current_user.name }}" required></div>
      <div class="mb-3"><label class="form-label">Roll number</label>
        <input name="roll_no" class="form-control" value="{{ current_user.roll_no or '' }}"></div>
      <div class="mb-3"><label class="form-label">Phone</label>
        <input name="phone" class="form-control" value="{{ current_user.phone or '' }}"></div>
      <div class="mb-3"><label class="form-label">New password (leave blank to keep)</label>
        <input type="password" name="password" class="form-control"></div>
      <button class="btn btn-primary">Update</button>
    </form>
  </div></div>
</div></div>
{% endblock %}
```

---

## 8. Optional: demo data — `seed_demo.py`

Run once with `python seed_demo.py` to get a ready demo (this only adds data through code; the admin is created by `app.py`).

```python
from datetime import date, timedelta
from app import create_app
from extensions import db
from models import (User, ExaminerProfile, Course, Examination, ExamExaminer, Rubric)

app = create_app()

with app.app_context():
    if Course.query.filter_by(code="CS101").first():
        print("Demo data already exists.")
    else:
        course = Course(code="CS101", name="Programming Basics", description="Intro course")
        db.session.add(course)
        db.session.flush()

        today = date.today()
        exam = Examination(course_id=course.id, name="CS101 Viva", exam_type="Viva",
                           duration=15, max_marks=50,
                           slot_creation_start=today - timedelta(days=1),
                           slot_creation_end=today + timedelta(days=7),
                           booking_start=today, booking_end=today + timedelta(days=14),
                           slot_creation_open=True, booking_open=False, status="Slot Creation")
        db.session.add(exam)
        db.session.flush()

        ex = User(name="Dr. Sharma", email="examiner@emp.com", role="examiner")
        ex.set_password("exam123")
        db.session.add(ex)
        db.session.flush()
        db.session.add(ExaminerProfile(user_id=ex.id, department="CSE", contact="9999999999",
                                       employee_id="E001", status="Approved"))
        db.session.add(ExamExaminer(exam_id=exam.id, examiner_id=ex.id))

        for name, mx in [("Concept clarity", 20), ("Coding", 20), ("Communication", 10)]:
            db.session.add(Rubric(exam_id=exam.id, criterion=name, max_marks=mx, weightage=1))

        for i in range(1, 4):
            s = User(name=f"Student {i}", email=f"student{i}@emp.com", role="student", roll_no=f"R00{i}")
            s.set_password("student123")
            db.session.add(s)

        db.session.commit()
        print("Demo data added.")
        print("Admin:    admin@emp.com / admin123")
        print("Examiner: examiner@emp.com / exam123")
        print("Student:  student1@emp.com / student123")
```

---

## 9. Run & test

```bash
python app.py          # then open http://127.0.0.1:5000
python seed_demo.py    # optional, in another terminal after the first run
```

**Full test story (do it once end-to-end):**
1. Log in as **admin** → Courses → Exams → check the demo exam → *Rubrics* → *Examiners*.
2. Log in as **examiner** → Dashboard → *Create slot* (works only while the slot creation window is open).
3. As admin → *Open booking* on the exam (booking dates must include today).
4. Log in as **student1** → Exams → *View slots* → Book → try booking another slot of the same exam (must be blocked) → Cancel → Book again.
5. Examiner → My Slots → *Students* → Evaluate → tick *Publish*.
6. Student → Results shows marks and remarks.
7. Try opening `/admin/dashboard` as a student → **403 page**.

## 10. Common errors and fixes

| Error | Cause / Fix |
|---|---|
| `ModuleNotFoundError: No module named 'flask_login'` | Virtual environment not active, or `pip install flask-login` missing |
| `ImportError: cannot import name 'api_bp'` | Create `routes/api.py` (Section 5) |
| `BuildError: Could not build url for endpoint ...` | Wrong endpoint name. Format is `blueprint.function`, e.g. `admin.course_add` |
| `jinja2.exceptions.TemplateNotFound` | Template folder/file name or path is wrong (must be inside `templates/`) |
| `OperationalError: no such column ...` | You changed `models.py` after the DB was created. During development, stop the app, **delete `emp.db`**, run again (admin is re-created automatically) |
| Changes not saved | Missing `db.session.commit()` |
| Examiner can't log in | Admin has not approved the account (Examiners page) |
| Student can't book | Admin hasn't opened booking, or today's date is outside the booking dates |
| Examiner can't create slot | Slot creation not opened, or today is outside the slot creation dates |

## 11. Git commits (one unique message each)

```bash
git add . && git commit -m "Milestone-0 EMP-MAD-1"
git add . && git commit -m "DB models and schema setup - EMP"
git add . && git commit -m "Authentication and role based access - EMP"
git add . && git commit -m "Admin dashboard and examination management - EMP"
git add . && git commit -m "Examiner dashboard and slot management - EMP"
git add . && git commit -m "Student dashboard and slot booking - EMP"
git add . && git commit -m "Booking history and evaluation tracking - EMP"
git push
```

## 12. Important for the viva

Because the rules say AI-assisted code is accepted only if you understand it: read each file once, and be able to explain the 5 key ideas — (1) `models.py` tables and relationships, (2) `seed_admin()` in `app.py`, (3) `role_required` decorator, (4) the three rules inside `book()` in `routes/student.py`, (5) the `b.slot.examiner_id != current_user.id` check in `evaluate()`. Also declare your AI/LLM usage honestly in the report.
