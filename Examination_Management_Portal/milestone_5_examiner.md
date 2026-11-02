## PHASE 5: EXAMINER DASHBOARD & SLOT MANAGEMENT (Day 23–28)

---

## 🧠 Big Picture: What Can an Examiner Do?

An Examiner represents academic staff responsible for:
1. Managing their own **evaluation slots** for assigned exams.
2. Viewing the **student attendance roster** for scheduled slots.
3. Conducting evaluations using defined **rubric criteria**, submitting scores, and giving personalized feedback.

---

## 🔒 Security: IDOR (Insecure Direct Object Reference) Prevention

Examiners are peers at the same role level. Without rigorous checks, Examiner A could modify Examiner B's slots by guessing their slot ID in the URL.
This vulnerability is called **IDOR** (Insecure Direct Object Reference).

To prevent this, every examiner route enforces **Ownership Verification**:
```python
slot = Slot.query.get_or_404(slot_id)
if slot.examiner_id != current_user.id:
    flash('Unauthorized: You can only manage your own slots.', 'danger')
    return redirect(url_for('examiner.slots'))
```
This is a **horizontal access control check** — verifying that the user owns the specific record they are modifying.

---

## ⏳ Business Rules: Slot Lifecycle & Boundaries

When creating or modifying slots, the application enforces:
1. **Assignment Check**: The examiner must be explicitly assigned to the exam by an Admin.
2. **Date Window Check**: Slots can only be created between the exam's `slot_creation_start` and `slot_creation_end` dates.
3. **Capacity Check**: Slot capacity must be at least 1 (`capacity >= 1`).
4. **Time Sanity Check**: Slot `start_time` must be before `end_time`.
5. **Immutability Once Booked**: An examiner cannot delete a slot if students have already booked it (`slot.booked_count > 0`).

---

## 💻 Step 5.1 — Create `routes/examiner.py`

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

### Route Highlights:
* **`/examiner/dashboard`**: Displays the examiner's assigned exams, upcoming scheduled slots, and pending evaluations awaiting grading.
* **`/examiner/slots/new`**: Validates assignment, date windows, and capacity before creating a new slot.
* **`/examiner/slots/<id>/edit` & `delete`**: Checks ownership and prevents deletion if students have already reserved seats.
* **`/examiner/slots/<id>/students`**: Lists all student candidates booked for that slot.
* **`/examiner/evaluate/<booking_id>`**: Evaluates a student against each exam rubric criterion, computes the total score, sets attendance (`Present` / `Absent`), and saves detailed feedback.

---

## 🖼️ Step 5.2 — Examiner HTML Templates

### 1. `templates/examiner/dashboard.html`
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

### 2. `templates/examiner/slots.html`
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

### 3. `templates/examiner/slot_form.html`
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

### 4. `templates/examiner/slot_students.html`
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

### 5. `templates/examiner/evaluate.html`
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

### 6. `templates/examiner/profile.html`
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

## 🧪 Step 5.3 — Verification & Testing Checklist

* [ ] Log in as an approved Examiner.
* [ ] Verify the dashboard displays the exams assigned to you by Admin.
* [ ] Attempt to create a slot outside the exam's `slot_creation` date window → Verify blocked with appropriate error.
* [ ] Create a slot within valid dates → Verify it appears in your slot list.
* [ ] Try accessing the edit URL of another examiner's slot → Verify IDOR check triggers and redirects with error.
* [ ] Open a slot with capacity 5 → Confirm `booked_count` starts at 0.
