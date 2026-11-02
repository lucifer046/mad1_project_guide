## PHASE 7: BOOKING HISTORY & EVALUATION TRACKING (Day 36–40)

---

## 🧠 Big Picture: Why History & Evaluation Tracking Matter

An examination portal is only as reliable as its audit trail and results delivery:
1. **Students** need transparent visibility into past bookings, cancellations, attendance records, and graded evaluation results.
2. **Examiners** need to record evaluations against specific rubrics with constructive remarks.
3. **Admins** need to audit all changes and view global system event logs.

---

## 📖 Navigating Relationships & Lazy Loading

SQLAlchemy models define relationships that can be navigated directly in Jinja2 templates:

```html
<!-- Navigate: Booking -> Slot -> Exam -> Course -->
<td>{{ b.slot.exam.course.code }}</td>
<td>{{ b.slot.exam.title }}</td>

<!-- Navigate: Booking -> Slot -> Examiner (User) -->
<td>{{ b.slot.examiner.full_name }}</td>

<!-- Navigate: Booking -> Evaluation -->
<td>{{ b.evaluation.marks_obtained }} / {{ b.slot.exam.max_marks }}</td>
```

By default, SQLAlchemy uses **lazy loading**: data for related tables is fetched automatically from the database only when accessed by the template.

---

## 📊 Results Publication Workflow

1. Examiner grades a student via `/examiner/evaluate/<booking_id>`.
2. The evaluation record is created with `is_published=True` (or toggled by Admin).
3. The student visits `/student/results`.
4. Only evaluations with `is_published == True` are displayed. If an evaluation is in draft or unpublished state, students cannot view preliminary marks.

---

## 🖼️ Step 7.1 — History & Results Templates

### 1. `templates/student/history.html`
Shows all historical bookings (both active, completed, and cancelled) with status badges:
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

### 2. `templates/student/results.html`
Displays finalized evaluation results, marks scored, rubric details, and examiner feedback:
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

### 3. `templates/admin/history.html`
Administrative audit log displaying timestamped system actions:
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

## 🧪 Step 7.2 — End-to-End Workflow Verification

Execute this complete lifecycle to verify system integrity:
1. **Admin**: Create course `MAD1`, exam `Final Viva`, rubric `Code Review (50)` and `Viva Q&A (50)`.
2. **Examiner**: Approved by Admin, assigned to `Final Viva`, creates a slot for 10:00 AM.
3. **Student**: Searches for `Final Viva`, books the 10:00 AM slot.
4. **Examiner**: Opens `/examiner/slots/<id>/students`, clicks **Evaluate**, awards 45/50 and 42/50 with feedback *"Excellent preparation"*, and submits.
5. **Student**: Checks `/student/results` and verifies the score (87/100) and feedback are displayed accurately.
6. **Admin**: Checks `/admin/history` and verifies all actions are logged in order.
