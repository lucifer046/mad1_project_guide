## PHASE 3: AUTHENTICATION & ROLE-BASED ACCESS (Day 11–14)

> 📺 Watch: [Flask Login System in Hindi](https://www.youtube.com/watch?v=71EU8gnZqZQ)

---

## 🧠 Big Picture: Authentication vs Authorization

* **Authentication ("Who are you?")**: Validating credentials (email and password) to verify identity.
* **Authorization ("What can you do?")**: Checking permissions to ensure a user has the appropriate role (`admin`, `examiner`, `student`) before accessing restricted endpoints.

---

## 🌐 HTTP Statelessness & Signed Session Cookies

HTTP is stateless — every incoming web request is independent.
To remember logged-in users, Flask-Login stores an encrypted session cookie in the user's browser. On every request, Flask:
1. Decrypts the session cookie using `SECRET_KEY`.
2. Retrieves the `user_id`.
3. Calls the `@login_manager.user_loader` in `app.py`.
4. Populates `current_user` in Flask's request context and Jinja2 templates.

---

## 🛡️ Step 3.1 — Create `decorators.py`

Custom decorators for Role-Based Access Control (RBAC):

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

### How `@admin_required`, `@examiner_required`, and `@student_required` Work:
1. `@wraps(f)`: Preserves the original function's name and docstring.
2. Checks if `current_user.is_authenticated`. If not, redirects to `/login`.
3. Checks if `current_user.role` matches the required role. If not, flashes an unauthorized warning and redirects safely.

---

## 📁 Step 3.2 — Create `routes/__init__.py`

```python

```

*(This empty or minimal file marks `routes/` as a Python package).*

---

## 🔐 Step 3.3 — Create `routes/auth.py`

The authentication blueprint managing login, logout, student registration, and examiner registration:

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

### Key Business Logic:
* **Student Registration**: Creates an account with `role='student'` and `is_active=True`. Students can log in immediately.
* **Examiner Registration**: Creates an account with `role='examiner'` and `is_active=False`. Examiners cannot log in until an Administrator approves their profile from the Admin dashboard.
* **Login Handler**: Checks password hash using `check_password_hash()`. If `is_active` is False, login is blocked with a message: *"Account pending approval or deactivated."*
* **Role-Based Redirects**: After login, users are routed automatically to their role's dashboard:
  - Admin → `/admin/dashboard`
  - Examiner → `/examiner/dashboard`
  - Student → `/student/dashboard`

---

## 🎨 Step 3.4 — Create `static/css/style.css`

```css
body { min-height: 100vh; }
.card { border-radius: 10px; }
.stat-card h2 { font-weight: 700; }
table td, table th { vertical-align: middle; }
.navbar-nav { flex-wrap: wrap; }
```

---

## 🖼️ Step 3.5 — Base & Authentication Templates

### 1. `templates/base.html`
The master layout containing Bootstrap 5 CDN, navigation bar, dynamic role links, flash messages, and footer:

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

### 2. `templates/home.html`
Landing page with overview and quick access buttons:

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

### 3. `templates/error.html`
Clean error handler template:

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

### 4. `templates/auth/login.html`
Login form supporting all three roles:

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

### 5. `templates/auth/register_student.html`
Student self-registration form:

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

### 6. `templates/auth/register_examiner.html`
Examiner registration form (informs user of required admin approval):

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

## 🧪 Step 3.6 — Verification & Testing Checklist

Test the authentication flow manually:
* [ ] Visit `http://127.0.0.1:5000/login`
* [ ] Log in with default admin credentials (`admin@emp.com` / `admin123`) → Verify redirect to `/admin/dashboard`.
* [ ] Register a new student (`student1@test.com`) → Log in → Verify redirect to `/student/dashboard`.
* [ ] Register a new examiner (`examiner1@test.com`) → Attempt login → Verify blocked with *"pending approval"* message.
* [ ] Log out → Verify session is cleared and redirected to login/home.
