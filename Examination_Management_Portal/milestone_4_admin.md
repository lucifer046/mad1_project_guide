## PHASE 4: ADMIN DASHBOARD & MANAGEMENT (Day 15–22)

> 📺 Watch: [Flask CRUD Operations in Hindi](https://www.youtube.com/watch?v=oA8brF3w5XQ)

---

## 🧠 Big Picture: What is CRUD in the Admin Module?

The administrator orchestrates the entire examination lifecycle through four foundational operations:
* **C (Create)**: Create Courses, Exams, Rubrics, and Assign Examiners.
* **R (Read)**: Monitor dashboard statistics, slot capacities, student rosters, and audit trails.
* **U (Update)**: Edit course and exam metadata, open/close scheduling windows, reschedule student bookings, and approve examiners.
* **D (Delete)**: Safely remove unneeded courses, exams, or rubrics (with foreign key checks).

---

## 📖 The SQLAlchemy Unit of Work Pattern (`db.session`)

Think of `db.session` as a staging cart:
1. `db.session.add(obj)`: Stages an item for insertion.
2. `db.session.delete(obj)`: Stages an item for removal.
3. `db.session.commit()`: Executes all staged SQL statements atomically in a single database transaction. If any error occurs, `db.session.rollback()` reverts everything safely.

---

## 💻 Step 4.1 — Create `routes/admin.py`

The complete administrative blueprint managing all system configurations, approvals, and scheduling oversight:

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

### Comprehensive Route Walkthrough:
1. **`/admin/dashboard`**: Gathers system-wide aggregates: total courses, exams, approved examiners, registered students, total slots, and booking volume.
2. **Courses (`/admin/courses`, `/new`, `/<id>/edit`, `/<id>/delete`)**: Full CRUD for course codes and titles. Deletion checks for attached exams to prevent orphan records.
3. **Exams (`/admin/exams`, `/new`, `/<id>/edit`, `/<id>/delete`)**: Configures title, course, exam type, duration, total marks, and crucially, the **slot creation window** and **booking window**.
4. **Assign Examiners (`/admin/exams/<id>/examiners`)**: Connects approved examiners to specific exams using the `exam_examiners` association table.
5. **Rubrics (`/admin/rubrics`, `/new`, `/<id>/edit`, `/<id>/delete`)**: Sets scoring criteria, descriptions, and max marks for each exam.
6. **Examiner Approvals (`/admin/examiners`, `/<id>/approve`, `/<id>/toggle`)**: Activates pending examiner registrations and allows deactivating problematic accounts.
7. **Student Oversight (`/admin/students`)**: Lists all registered students along with their booking counts.
8. **Slots Oversight (`/admin/slots`)**: Displays all slots across exams, with filtering and live capacity monitoring.
9. **Bookings & Cancellation (`/admin/bookings`, `/<id>/cancel`)**: Overview of all student bookings; allows administrative cancellation which automatically restores slot capacity.
10. **Reschedule Booking (`/admin/bookings/<id>/reschedule`)**: Solves scheduling emergencies by reassigning a student to a different slot or examiner.
11. **Multi-Entity Search (`/admin/search`)**: Allows searching across courses, exams, examiners, and students via keyword.
12. **History & Audit Logs (`/admin/history`)**: Renders all activity log records recorded by `log_activity()`.

---

## 🖼️ Step 4.2 — Admin HTML Templates

### 1. `templates/admin/dashboard.html`
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

### 2. `templates/admin/courses.html`
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

### 3. `templates/admin/course_form.html`
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

### 4. `templates/admin/exams.html`
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

### 5. `templates/admin/exam_form.html`
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

### 6. `templates/admin/exam_examiners.html`
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

### 7. `templates/admin/rubrics.html`
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

### 8. `templates/admin/rubric_form.html`
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

### 9. `templates/admin/examiners.html`
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

### 10. `templates/admin/examiner_form.html`
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

### 11. `templates/admin/students.html`
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

### 12. `templates/admin/slots.html`
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

### 13. `templates/admin/bookings.html`
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

### 14. `templates/admin/reschedule.html`
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

### 15. `templates/admin/search.html`
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

### 16. `templates/admin/history.html`
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

## 🧪 Step 4.3 — Verification & Testing Checklist

* [ ] Log in as Admin (`admin@emp.com` / `admin123`).
* [ ] Create a Course (`CS101`, `Introduction to Programming`).
* [ ] Create an Exam linked to that course (`Midterm Lab Exam`). Set slot creation and booking windows.
* [ ] Register an examiner account (`examiner@test.com`) in another browser / incognito tab.
* [ ] As Admin, visit `/admin/examiners` and click **Approve**.
* [ ] Visit `/admin/exams/<id>/examiners` and assign the approved examiner to the exam.
* [ ] Create rubrics under `/admin/rubrics` (`Code Quality`, `Viva Q&A`).
* [ ] Verify that audit logs under `/admin/history` record each action.
