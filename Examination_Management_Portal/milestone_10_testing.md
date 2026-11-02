## PHASE 10: COMPREHENSIVE TESTING CHECKLIST (Day 41–42)

> 💡 Verify each item manually in the browser before submitting your project.

---

### 1. Authentication & Security Tests
* [ ] Register as a Student → Verify immediate login works and lands on `/student/dashboard`.
* [ ] Register as an Examiner → Verify login is blocked with *"Account pending approval"*.
* [ ] As Admin, approve the Examiner → Verify Examiner can now log in successfully.
* [ ] Attempt to access `/admin/dashboard` while logged in as a Student → Verify 403 / redirect to login.
* [ ] Attempt to access `/examiner/dashboard` while logged in as a Student → Verify 403 / redirect to login.
* [ ] Log out → Verify browser Back button cannot submit authenticated forms.
* [ ] Verify passwords stored in `emp.db` are hashed (`scrypt:...` or `pbkdf2:...`) and not plain text.

---

### 2. Admin Module Tests
* [ ] Create a Course (`CS201`) → Verify it appears in `/admin/courses`.
* [ ] Edit Course details → Verify changes save and reflect.
* [ ] Create an Exam linked to the course → Set future slot creation and booking windows.
* [ ] Assign an approved Examiner to the Exam → Verify assignment appears on `/admin/exams/<id>/examiners`.
* [ ] Create Rubrics for the Exam with max marks.
* [ ] Toggle Examiner status to deactivated → Verify Examiner can no longer log in.
* [ ] View all slots on `/admin/slots`.
* [ ] View student bookings on `/admin/bookings`.
* [ ] Reschedule a student booking to a different slot and examiner → Verify slot capacities update accurately.
* [ ] Test keyword search on `/admin/search` for course code, student name, and exam title.
* [ ] Check `/admin/history` to ensure administrative operations were logged.

---

### 3. Examiner Module Tests
* [ ] Log in as Examiner → Verify assigned exams are visible on dashboard.
* [ ] Attempt to create a slot outside the exam's slot creation window → Verify blocked.
* [ ] Create a slot within the window → Verify slot appears under `/examiner/slots`.
* [ ] Attempt to edit/delete another examiner's slot via URL manipulation → Verify IDOR check triggers.
* [ ] View attendance roster for a slot with booked students.
* [ ] Evaluate a student: Enter rubric scores, feedback, and mark attendance (`Present`) → Submit evaluation.
* [ ] Verify evaluation displays as completed.

---

### 4. Student Module Tests
* [ ] Log in as Student → View available exams on `/student/exams`.
* [ ] Test Course filter and search input → Verify filtering functions correctly without JavaScript.
* [ ] Attempt to book a slot for an exam whose booking window is closed → Verify blocked.
* [ ] Book a valid slot → Verify slot's `booked_count` increments by 1.
* [ ] Attempt to book a second slot for the same exam → Verify blocked with error: *"You have already booked a slot for this exam."*
* [ ] Attempt to book a conflicting slot (same date/time) for another exam → Verify blocked by schedule conflict detection.
* [ ] View active bookings under `/student/bookings`.
* [ ] Cancel active booking → Verify `booked_count` decrements by 1 and status updates to `Cancelled`.
* [ ] Visit `/student/history` → Verify cancelled booking is recorded.
* [ ] Visit `/student/results` → Verify marks, feedback, and rubric scores are displayed once evaluated.
