## PHASE 9: OPTIONAL FEATURES (Do These for Higher Scores & Bonus Marks)

> 💡 **Tip**: Core requirements must be completely working and tested before adding optional features. These features elevate your project quality during viva evaluation.

---

## 9.1 — Automated Demo Data Seeder (`seed_demo.py`)

During project demonstration and viva, you need realistic sample courses, exams, slots, and students populated without manually typing them into web forms.

Create `seed_demo.py` in the root folder:

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

### How to Run:
```bash
python seed_demo.py
```
This populates sample courses (`MAD-1`, `DBMS`), exams, approved examiners, active students, slots, and bookings.

---

## 9.2 — RESTful JSON API (`routes/api.py`)

Providing a JSON API demonstrates backend versatility and separates data delivery from HTML rendering.

Create `routes/api.py`:

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

### API Endpoints:
* `GET /api/v1/exams`: Returns JSON array of published exams.
* `GET /api/v1/exams/<id>/slots`: Returns available slots and capacities for an exam.
* `GET /api/v1/results`: Returns the authenticated student's published examination results in JSON format.

---

## 9.3 — Chart.js Visual Analytics (Admin Dashboard)

Visualizing data with Chart.js on the admin dashboard provides instant insights.
Add this script block to `templates/admin/dashboard.html` inside `{% block content %}`:

```html
<!-- Include Chart.js via CDN -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<div class="row mt-4">
    <div class="col-md-6 mb-4">
        <div class="card shadow-sm">
            <div class="card-header bg-white fw-bold">Slot Capacity vs Booked</div>
            <div class="card-body">
                <canvas id="slotChart" width="400" height="250"></canvas>
            </div>
        </div>
    </div>
    <div class="col-md-6 mb-4">
        <div class="card shadow-sm">
            <div class="card-header bg-white fw-bold">Student Evaluation Status</div>
            <div class="card-body">
                <canvas id="statusChart" width="400" height="250"></canvas>
            </div>
        </div>
    </div>
</div>

<script>
const ctx1 = document.getElementById('slotChart').getContext('2d');
new Chart(ctx1, {
    type: 'bar',
    data: {
        labels: ['Total Slots Capacity', 'Total Booked Seats'],
        datasets: [{
            label: 'Counts',
            data: [{{ total_capacity|default(50) }}, {{ total_bookings|default(35) }}],
            backgroundColor: ['#0d6efd', '#198754']
        }]
    },
    options: { responsive: true }
});
</script>
```

---

## 9.4 — CSV Export Feature

To allow examiners and administrators to export student evaluation results into spreadsheet formats:

```python
import csv
import io
from flask import Response

@admin_bp.route('/export/results/<int:exam_id>')
@admin_required
def export_results(exam_id):
    exam = Exam.query.get_or_404(exam_id)
    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(['Student Name', 'Email', 'Roll Number', 'Slot Time', 'Marks Obtained', 'Max Marks', 'Attendance'])
    
    for slot in exam.slots:
        for booking in slot.bookings:
            eval_record = booking.evaluation
            marks = eval_record.marks_obtained if eval_record else 'Pending'
            attendance = eval_record.attendance if eval_record else 'N/A'
            writer.writerow([
                booking.student.full_name,
                booking.student.email,
                booking.student.roll_number or 'N/A',
                slot.start_time.strftime('%Y-%m-%d %H:%M'),
                marks,
                exam.max_marks,
                attendance
            ])
            
    output.seek(0)
    return Response(output, mimetype="text/csv", headers={"Content-Disposition": f"attachment;filename=results_exam_{exam_id}.csv"})
```
