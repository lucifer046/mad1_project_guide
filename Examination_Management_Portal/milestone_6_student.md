## PHASE 6: STUDENT DASHBOARD & EXAM BOOKING (Day 29–35)

---

## 🧠 Big Picture: What Can Students Do?

Students are the candidates using the portal to:
1. **Browse upcoming examinations** with course and keyword filters.
2. **View available evaluation slots** created by assigned examiners.
3. **Book a slot** while strictly obeying all scheduling rules.
4. **Cancel an active booking** if plans change before the examination begins.
5. **View their booking history and published results**.

---

## 📜 The 6 Critical Booking Rules

The booking endpoint is the most critical function in the entire application. It strictly enforces:

```
                  Student Clicks "Book Slot"
                              │
               1. Is account active?
                     ├── No  ──> Block & flash error
                     └── Yes ──> Next check
                              │
               2. Is booking window open?
               (booking_start <= now <= booking_end)
                     ├── No  ──> Block & flash error
                     └── Yes ──> Next check
                              │
               3. Does the slot have capacity?
                  (booked_count < capacity)
                     ├── No  ──> Block ("Slot Full")
                     └── Yes ──> Next check
                              │
               4. Has student already booked this exam?
                     ├── Yes ──> Block ("Already booked")
                     └── No  ──> Next check
                              │
               5. Does student have a conflicting slot?
               (overlapping start_time & end_time)
                     ├── Yes ──> Block ("Schedule conflict")
                     └── No  ──> Next check
                              │
                     6. Atomic Booking Creation:
                  - Create Booking(status='Booked')
                  - Increment slot.booked_count += 1
                  - db.session.commit()
```

---

## 🚫 No-JavaScript Rule Compliance

MAD-1 strictly prohibits JavaScript for core functionality.
Search and filtering on the Student Exam Catalog is accomplished purely with standard **HTML GET forms**:

```html
<form method="GET" action="{{ url_for('student.exams') }}">
    <select name="course_id">...</select>
    <input type="text" name="q" placeholder="Search by exam title...">
    <button type="submit">Filter</button>
</form>
```
When submitted, the browser executes a standard `GET` request: `/student/exams?course_id=2&q=Midterm`. The Flask route reads `request.args.get('course_id')` and filters the SQLAlchemy query accordingly.

---

## 💻 Step 6.1 — Create `routes/student.py`

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

---

## 🖼️ Step 6.2 — Student HTML Templates

### 1. `templates/student/dashboard.html`
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

### 2. `templates/student/exams.html`
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

### 3. `templates/student/exam_slots.html`
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

### 4. `templates/student/bookings.html`
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

### 5. `templates/student/profile.html`
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

## 🧪 Step 6.3 — Verification & Testing Checklist

* [ ] Log in as a registered Student (`student@test.com`).
* [ ] Visit `/student/exams` and filter by course. Verify filter works without JavaScript.
* [ ] Click **View Slots** for an exam within its booking window.
* [ ] Book a slot → Verify `slot.booked_count` increases by 1.
* [ ] Try booking another slot for the **same exam** → Verify blocked with error: *"You have already booked a slot for this exam."*
* [ ] Try booking another slot at the **same time** for a different exam → Verify blocked by conflict detection.
* [ ] Cancel the booking from `/student/bookings` → Verify `slot.booked_count` decreases by 1 and status changes to `Cancelled`.
