# MAD-1 Trekking Management App — Complete Roadmap
### IITM BS | May 2026 Term | Flask + Jinja2 + SQLite

---

> **TL;DR Order:** GitHub Setup → ER Diagram → DB Models → Auth → Admin → Staff → User → Booking History → Optional Features → Submit

---

## RESOURCES (Watch Before Starting Each Section)

### Hindi Playlist — Flask Full Course
- **[Corey Schafer - Flask Series (English, Best Quality)](https://www.youtube.com/playlist?list=PL-osiE80TeTs4UjLw5MM6OjgkjFeUxCYH)**
- **[Hindi Flask Tutorial by CodeWithHarry](https://www.youtube.com/watch?v=oA8brF3w5XQ)** — Full Flask in Hindi
- **[Python + Flask Hindi by Apna College](https://www.youtube.com/watch?v=Z1RJmh_OqeA)**
- **[SQLite + Python Hindi](https://www.youtube.com/watch?v=byHcYRpMgI4)**
- **[Jinja2 Templating Hindi](https://www.youtube.com/watch?v=mqhxxeeTbu0)**
- **[Bootstrap 5 Hindi by CodeWithHarry](https://www.youtube.com/watch?v=vpAJ0s5S2t0)**
- **[Flask Login System Hindi](https://www.youtube.com/watch?v=71EU8gnZqZQ)**
- **[ER Diagram Kaise Banaye (Hindi)](https://www.youtube.com/watch?v=QpdhBUYk7Kk)**
- **[SQLAlchemy ORM Hindi](https://www.youtube.com/watch?v=cYWiDiIUxQc)**

---

## TIMELINE AT A GLANCE

| Week | Task |
|------|------|
| Week 1 | GitHub Setup + Learn Flask Basics + Design ER Diagram |
| Week 2 | Create DB Models + Authentication (Login/Register) |
| Week 3 | Admin Dashboard + Trek CRUD |
| Week 4 | Trek Staff Dashboard |
| Week 5 | User Dashboard + Booking System |
| Week 6 | Booking History + Status Tracking |
| Week 7 | Optional Features + Testing |
| Week 8 | Video + Report + Final Submission |

**Submission Deadline (Theory Done Before May 2026): 14 July 2026**
**Submission Deadline (Theory in May 2026): 9 August 2026**

---

## TECH STACK (MANDATORY — Don't Use Anything Else)

| Layer | Technology |
|-------|-----------|
| Backend | **Flask** (Python) |
| Frontend | **Jinja2 templates + HTML + CSS + Bootstrap 5** |
| Database | **SQLite** (via SQLAlchemy ORM) |
| Auth | Flask sessions (or Flask-Login — optional) |
| JS |  NOT for core features. OK for Bootstrap tooltips/charts |

---

## PHASE 0: SETUP (Day 1-2)

### Step 0.1 — Install Required Software

```bash
# 1. Install Python (if not installed)
# Download from: https://www.python.org/downloads/

# 2. Check Python version
python --version  # Should be 3.8+

# 3. Install Flask and dependencies
pip install flask flask-sqlalchemy flask-login

# 4. Install VS Code (recommended editor)
# Download from: https://code.visualstudio.com/
```

### Step 0.2 — Create Your Project Folder Structure

Create this folder structure **manually first**, then we'll fill files one by one:

```
trekking_app/
│
├── app.py                  ← Main Flask application file
├── config.py               ← Configuration (secret key, DB path)
├── models.py               ← All database models (tables)
├── requirements.txt        ← List of all pip packages
│
├── static/
│   ├── css/
│   │   └── style.css       ← Your custom CSS
│   └── images/             ← Any images you use
│
└── templates/
    ├── base.html           ← Common layout (navbar, footer)
    ├── index.html          ← Home page
    │
    ├── auth/
    │   ├── login.html
    │   └── register.html
    │
    ├── admin/
    │   ├── dashboard.html
    │   ├── manage_treks.html
    │   ├── manage_users.html
    │   ├── manage_staff.html
    │   └── assign_staff.html
    │
    ├── staff/
    │   ├── dashboard.html
    │   └── manage_trek.html
    │
    └── user/
        ├── dashboard.html
        ├── browse_treks.html
        ├── booking_history.html
        └── profile.html
```

### Step 0.3 — Create requirements.txt

```
flask
flask-sqlalchemy
flask-login
werkzeug
```

---

## PHASE 1: DESIGN ER DIAGRAM (Day 2-3)

>  **Do this BEFORE writing any code.** The ER diagram is the blueprint of your database.

### What is an ER Diagram?

An ER (Entity-Relationship) diagram shows:
- What **tables** (entities) you have
- What **columns** (attributes) each table has
- How tables are **connected** (relationships)

### Your Tables

#### Table 1: `User`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key, Auto |
| name | String | Full name |
| email | String | Unique |
| password | String | Hashed |
| role | String | 'admin', 'staff', 'user' |
| is_active | Boolean | Default True |
| is_blacklisted | Boolean | Default False |
| created_at | DateTime | Auto |

#### Table 2: `Trek`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key, Auto |
| name | String | Trek name |
| location | String | e.g. "Manali" |
| difficulty | String | 'Easy', 'Moderate', 'Hard' |
| duration_days | Integer | Number of days |
| available_slots | Integer | Max bookings allowed |
| start_date | Date | |
| end_date | Date | |
| status | String | 'Pending','Approved','Open','Closed','Completed' |
| assigned_staff_id | Integer | Foreign Key → User.id |
| description | Text | Optional |

#### Table 3: `Booking`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key, Auto |
| user_id | Integer | Foreign Key → User.id |
| trek_id | Integer | Foreign Key → Trek.id |
| booking_date | DateTime | Auto |
| status | String | 'Booked', 'Cancelled', 'Completed' |

#### Table 4: `StaffProfile` (Optional but Recommended)
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key |
| user_id | Integer | Foreign Key → User.id |
| contact | String | Phone number |
| experience | String | Experience details |
| approval_status | String | 'Pending', 'Approved', 'Rejected' |

### Relationships
```
User (staff) ─────< Trek         [One staff can have many treks]
User (trekker) ───< Booking       [One user can have many bookings]
Trek ──────────────< Booking      [One trek can have many bookings]
User (staff) ──────── StaffProfile [One-to-one]
```

### How to Draw ER Diagram
Use **draw.io** (free online): https://app.diagrams.net/
- Create boxes for each table
- Add columns inside each box
- Draw arrows for relationships (crow's foot notation)
- Export as PNG for your report

---

## PHASE 2: DATABASE MODELS (Day 3-5)

>  Watch: [SQLAlchemy Models Hindi](https://www.youtube.com/watch?v=cYWiDiIUxQc)

### Step 2.1 — Create `config.py`

```python
# config.py
import os

class Config:
    SECRET_KEY = 'your-super-secret-key-change-this'
    SQLALCHEMY_DATABASE_URI = 'sqlite:///trekking.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

### Step 2.2 — Create `models.py`

```python
# models.py
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime
from werkzeug.security import generate_password_hash, check_password_hash

db = SQLAlchemy()

class User(db.Model):
    __tablename__ = 'user'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password = db.Column(db.String(200), nullable=False)
    role = db.Column(db.String(20), nullable=False)  # 'admin', 'staff', 'user'
    is_active = db.Column(db.Boolean, default=True)
    is_blacklisted = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    # Relationships
    bookings = db.relationship('Booking', backref='trekker', lazy=True)
    assigned_treks = db.relationship('Trek', backref='assigned_staff', lazy=True,
                                      foreign_keys='Trek.assigned_staff_id')
    staff_profile = db.relationship('StaffProfile', backref='user', uselist=False)
    
    def set_password(self, password):
        self.password = generate_password_hash(password)
    
    def check_password(self, password):
        return check_password_hash(self.password, password)
    
    def __repr__(self):
        return f'<User {self.email}>'


class Trek(db.Model):
    __tablename__ = 'trek'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(150), nullable=False)
    location = db.Column(db.String(100), nullable=False)
    difficulty = db.Column(db.String(20), nullable=False)  # Easy/Moderate/Hard
    duration_days = db.Column(db.Integer, nullable=False)
    available_slots = db.Column(db.Integer, nullable=False)
    start_date = db.Column(db.Date, nullable=False)
    end_date = db.Column(db.Date, nullable=False)
    status = db.Column(db.String(20), default='Pending')  # Pending/Approved/Open/Closed/Completed
    assigned_staff_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=True)
    description = db.Column(db.Text, default='')
    
    # Relationships
    bookings = db.relationship('Booking', backref='trek', lazy=True)
    
    def get_booked_slots(self):
        return Booking.query.filter_by(trek_id=self.id, status='Booked').count()
    
    def get_remaining_slots(self):
        return self.available_slots - self.get_booked_slots()
    
    def __repr__(self):
        return f'<Trek {self.name}>'


class Booking(db.Model):
    __tablename__ = 'booking'
    
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    trek_id = db.Column(db.Integer, db.ForeignKey('trek.id'), nullable=False)
    booking_date = db.Column(db.DateTime, default=datetime.utcnow)
    status = db.Column(db.String(20), default='Booked')  # Booked/Cancelled/Completed
    
    def __repr__(self):
        return f'<Booking User:{self.user_id} Trek:{self.trek_id}>'


class StaffProfile(db.Model):
    __tablename__ = 'staff_profile'
    
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    contact = db.Column(db.String(20))
    experience = db.Column(db.String(200))
    approval_status = db.Column(db.String(20), default='Pending')  # Pending/Approved/Rejected
    
    def __repr__(self):
        return f'<StaffProfile user_id:{self.user_id}>'
```

### Step 2.3 — Create `app.py` (Basic Setup)

```python
# app.py
from flask import Flask
from config import Config
from models import db, User, StaffProfile
from werkzeug.security import generate_password_hash

def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)
    
    db.init_app(app)
    
    with app.app_context():
        db.create_all()
        create_admin()  # Create default admin
    
    return app

def create_admin():
    """Create admin user if doesn't exist"""
    admin = User.query.filter_by(email='admin@trek.com').first()
    if not admin:
        admin = User(
            name='Admin',
            email='admin@trek.com',
            role='admin',
            is_active=True
        )
        admin.set_password('admin123')  # Change this password!
        db.session.add(admin)
        db.session.commit()
        print(" Admin created: admin@trek.com / admin123")

app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
```

### Step 2.4 — Test Database Creation

```bash
# Run this in terminal from your project folder
python app.py
# You should see: * Running on http://127.0.0.1:5000
# Check that trekking.db file was created
```

---

## PHASE 3: AUTHENTICATION (Day 5-8)

>  Watch: [Flask Login System Hindi](https://www.youtube.com/watch?v=71EU8gnZqZQ)

### Step 3.1 — Create `templates/base.html`

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Trekking App{% endblock %}</title>
    <!-- Bootstrap 5 CDN -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
    <!-- Navbar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand" href="/"> TrekkingApp</a>
            <div class="navbar-nav ms-auto">
                {% if session.user_id %}
                    <span class="navbar-text text-light me-3">Hello, {{ session.user_name }}</span>
                    <a class="nav-link" href="/logout">Logout</a>
                {% else %}
                    <a class="nav-link" href="/login">Login</a>
                    <a class="nav-link" href="/register">Register</a>
                {% endif %}
            </div>
        </div>
    </nav>
    
    <!-- Flash Messages -->
    <div class="container mt-3">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category }} alert-dismissible fade show">
                        {{ message }}
                        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
    </div>
    
    <!-- Main Content -->
    <div class="container mt-4">
        {% block content %}{% endblock %}
    </div>
    
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

### Step 3.2 — Create `templates/auth/login.html`

```html
<!-- templates/auth/login.html -->
{% extends "base.html" %}
{% block title %}Login{% endblock %}
{% block content %}
<div class="row justify-content-center">
    <div class="col-md-5">
        <div class="card shadow mt-5">
            <div class="card-body p-4">
                <h3 class="card-title text-center mb-4"> Login</h3>
                <form method="POST" action="/login">
                    <div class="mb-3">
                        <label class="form-label">Email</label>
                        <input type="email" name="email" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Password</label>
                        <input type="password" name="password" class="form-control" required>
                    </div>
                    <button type="submit" class="btn btn-primary w-100">Login</button>
                </form>
                <p class="text-center mt-3">No account? <a href="/register">Register here</a></p>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### Step 3.3 — Create `templates/auth/register.html`

```html
<!-- templates/auth/register.html -->
{% extends "base.html" %}
{% block title %}Register{% endblock %}
{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6">
        <div class="card shadow mt-5">
            <div class="card-body p-4">
                <h3 class="text-center mb-4"> Register</h3>
                <form method="POST" action="/register">
                    <div class="mb-3">
                        <label class="form-label">Full Name</label>
                        <input type="text" name="name" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Email</label>
                        <input type="email" name="email" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Password</label>
                        <input type="password" name="password" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Register as</label>
                        <select name="role" class="form-select">
                            <option value="user">Trekker (User)</option>
                            <option value="staff">Trek Staff</option>
                        </select>
                    </div>
                    <!-- Show only if staff selected -->
                    <div class="mb-3" id="contact_field">
                        <label class="form-label">Contact Number (for Staff)</label>
                        <input type="text" name="contact" class="form-control">
                    </div>
                    <button type="submit" class="btn btn-success w-100">Register</button>
                </form>
                <p class="text-center mt-3">Already registered? <a href="/login">Login</a></p>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### Step 3.4 — Add Auth Routes to `app.py`

```python
# Add these imports at top of app.py
from flask import Flask, render_template, request, redirect, url_for, session, flash
from functools import wraps

# ────── DECORATORS (Access Control) ──────

def login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if 'user_id' not in session:
            flash('Please login first.', 'warning')
            return redirect('/login')
        return f(*args, **kwargs)
    return decorated

def admin_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if session.get('role') != 'admin':
            flash('Admin access required.', 'danger')
            return redirect('/')
        return f(*args, **kwargs)
    return decorated

def staff_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if session.get('role') != 'staff':
            flash('Staff access required.', 'danger')
            return redirect('/')
        return f(*args, **kwargs)
    return decorated

# ────── AUTH ROUTES ──────

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form.get('email')
        password = request.form.get('password')
        
        user = User.query.filter_by(email=email).first()
        
        if not user or not user.check_password(password):
            flash('Invalid email or password.', 'danger')
            return render_template('auth/login.html')
        
        if user.is_blacklisted:
            flash('Your account has been blacklisted. Contact admin.', 'danger')
            return render_template('auth/login.html')
        
        # Check staff approval
        if user.role == 'staff':
            profile = StaffProfile.query.filter_by(user_id=user.id).first()
            if not profile or profile.approval_status != 'Approved':
                flash('Your staff account is pending admin approval.', 'warning')
                return render_template('auth/login.html')
        
        # Set session
        session['user_id'] = user.id
        session['user_name'] = user.name
        session['role'] = user.role
        
        # Redirect based on role
        if user.role == 'admin':
            return redirect('/admin/dashboard')
        elif user.role == 'staff':
            return redirect('/staff/dashboard')
        else:
            return redirect('/user/dashboard')
    
    return render_template('auth/login.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        name = request.form.get('name')
        email = request.form.get('email')
        password = request.form.get('password')
        role = request.form.get('role')  # 'user' or 'staff'
        contact = request.form.get('contact', '')
        
        # Check if email exists
        if User.query.filter_by(email=email).first():
            flash('Email already registered. Try logging in.', 'danger')
            return render_template('auth/register.html')
        
        # Prevent admin registration
        if role == 'admin':
            flash('Cannot register as admin.', 'danger')
            return render_template('auth/register.html')
        
        # Create user
        user = User(name=name, email=email, role=role)
        user.set_password(password)
        db.session.add(user)
        db.session.flush()  # Get user.id without committing
        
        # If staff, create profile
        if role == 'staff':
            profile = StaffProfile(user_id=user.id, contact=contact, approval_status='Pending')
            db.session.add(profile)
            flash('Registration successful! Wait for admin approval.', 'success')
        else:
            flash('Registration successful! You can now login.', 'success')
        
        db.session.commit()
        return redirect('/login')
    
    return render_template('auth/register.html')

@app.route('/logout')
def logout():
    session.clear()
    flash('Logged out successfully.', 'info')
    return redirect('/login')
```

---

## PHASE 4: ADMIN DASHBOARD (Day 8-14)

>  Watch: [Flask CRUD Operations Hindi](https://www.youtube.com/watch?v=oA8brF3w5XQ) — Skip to Flask CRUD part

### Step 4.1 — Create `templates/admin/dashboard.html`

```html
<!-- templates/admin/dashboard.html -->
{% extends "base.html" %}
{% block title %}Admin Dashboard{% endblock %}
{% block content %}
<h2> Admin Dashboard</h2>

<!-- Stats Cards -->
<div class="row mb-4">
    <div class="col-md-3">
        <div class="card text-white bg-primary">
            <div class="card-body">
                <h5>Total Treks</h5>
                <h2>{{ stats.total_treks }}</h2>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card text-white bg-success">
            <div class="card-body">
                <h5>Total Users</h5>
                <h2>{{ stats.total_users }}</h2>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card text-white bg-warning">
            <div class="card-body">
                <h5>Total Staff</h5>
                <h2>{{ stats.total_staff }}</h2>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card text-white bg-info">
            <div class="card-body">
                <h5>Total Bookings</h5>
                <h2>{{ stats.total_bookings }}</h2>
            </div>
        </div>
    </div>
</div>

<!-- Quick Links -->
<div class="row">
    <div class="col-md-4">
        <a href="/admin/treks" class="btn btn-outline-primary w-100 mb-2"> Manage Treks</a>
    </div>
    <div class="col-md-4">
        <a href="/admin/staff" class="btn btn-outline-success w-100 mb-2"> Manage Staff</a>
    </div>
    <div class="col-md-4">
        <a href="/admin/users" class="btn btn-outline-warning w-100 mb-2"> Manage Users</a>
    </div>
</div>

<!-- Search Bar -->
<div class="mt-4">
    <form method="GET" action="/admin/search">
        <div class="input-group">
            <input type="text" name="q" class="form-control" placeholder="Search trek, user, or staff by name/ID...">
            <button class="btn btn-dark" type="submit">Search</button>
        </div>
    </form>
</div>
{% endblock %}
```

### Step 4.2 — Add Admin Routes to `app.py`

```python
# ────── ADMIN ROUTES ──────

@app.route('/admin/dashboard')
@login_required
@admin_required
def admin_dashboard():
    stats = {
        'total_treks': Trek.query.count(),
        'total_users': User.query.filter_by(role='user').count(),
        'total_staff': User.query.filter_by(role='staff').count(),
        'total_bookings': Booking.query.count()
    }
    return render_template('admin/dashboard.html', stats=stats)

# ── Trek Management ──

@app.route('/admin/treks')
@login_required
@admin_required
def admin_treks():
    treks = Trek.query.all()
    return render_template('admin/manage_treks.html', treks=treks)

@app.route('/admin/trek/add', methods=['GET', 'POST'])
@login_required
@admin_required
def add_trek():
    if request.method == 'POST':
        from datetime import datetime
        trek = Trek(
            name=request.form['name'],
            location=request.form['location'],
            difficulty=request.form['difficulty'],
            duration_days=int(request.form['duration_days']),
            available_slots=int(request.form['available_slots']),
            start_date=datetime.strptime(request.form['start_date'], '%Y-%m-%d').date(),
            end_date=datetime.strptime(request.form['end_date'], '%Y-%m-%d').date(),
            description=request.form.get('description', ''),
            status='Pending'
        )
        db.session.add(trek)
        db.session.commit()
        flash('Trek added successfully!', 'success')
        return redirect('/admin/treks')
    return render_template('admin/add_trek.html')

@app.route('/admin/trek/edit/<int:trek_id>', methods=['GET', 'POST'])
@login_required
@admin_required
def edit_trek(trek_id):
    trek = Trek.query.get_or_404(trek_id)
    if request.method == 'POST':
        from datetime import datetime
        trek.name = request.form['name']
        trek.location = request.form['location']
        trek.difficulty = request.form['difficulty']
        trek.duration_days = int(request.form['duration_days'])
        trek.available_slots = int(request.form['available_slots'])
        trek.start_date = datetime.strptime(request.form['start_date'], '%Y-%m-%d').date()
        trek.end_date = datetime.strptime(request.form['end_date'], '%Y-%m-%d').date()
        trek.status = request.form['status']
        trek.description = request.form.get('description', '')
        db.session.commit()
        flash('Trek updated!', 'success')
        return redirect('/admin/treks')
    return render_template('admin/edit_trek.html', trek=trek)

@app.route('/admin/trek/delete/<int:trek_id>')
@login_required
@admin_required
def delete_trek(trek_id):
    trek = Trek.query.get_or_404(trek_id)
    db.session.delete(trek)
    db.session.commit()
    flash('Trek deleted.', 'info')
    return redirect('/admin/treks')

# ── Staff Management ──

@app.route('/admin/staff')
@login_required
@admin_required
def admin_staff():
    staff_list = User.query.filter_by(role='staff').all()
    return render_template('admin/manage_staff.html', staff_list=staff_list)

@app.route('/admin/staff/approve/<int:user_id>')
@login_required
@admin_required
def approve_staff(user_id):
    profile = StaffProfile.query.filter_by(user_id=user_id).first()
    if profile:
        profile.approval_status = 'Approved'
        db.session.commit()
        flash('Staff approved!', 'success')
    return redirect('/admin/staff')

@app.route('/admin/staff/blacklist/<int:user_id>')
@login_required
@admin_required
def blacklist_staff(user_id):
    user = User.query.get_or_404(user_id)
    user.is_blacklisted = True
    db.session.commit()
    flash(f'{user.name} has been blacklisted.', 'warning')
    return redirect('/admin/staff')

# ── Assign Staff to Trek ──

@app.route('/admin/trek/assign/<int:trek_id>', methods=['GET', 'POST'])
@login_required
@admin_required
def assign_staff(trek_id):
    trek = Trek.query.get_or_404(trek_id)
    approved_staff = db.session.query(User).join(StaffProfile).filter(
        User.role == 'staff',
        StaffProfile.approval_status == 'Approved'
    ).all()
    
    if request.method == 'POST':
        staff_id = request.form.get('staff_id')
        trek.assigned_staff_id = int(staff_id) if staff_id else None
        trek.status = 'Approved'
        db.session.commit()
        flash('Staff assigned and trek approved!', 'success')
        return redirect('/admin/treks')
    
    return render_template('admin/assign_staff.html', trek=trek, staff_list=approved_staff)

# ── User Management ──

@app.route('/admin/users')
@login_required
@admin_required
def admin_users():
    users = User.query.filter_by(role='user').all()
    return render_template('admin/manage_users.html', users=users)

@app.route('/admin/user/blacklist/<int:user_id>')
@login_required
@admin_required
def blacklist_user(user_id):
    user = User.query.get_or_404(user_id)
    user.is_blacklisted = True
    db.session.commit()
    flash(f'{user.name} has been blacklisted.', 'warning')
    return redirect('/admin/users')

# ── Search ──

@app.route('/admin/search')
@login_required
@admin_required
def admin_search():
    q = request.args.get('q', '')
    treks = Trek.query.filter(Trek.name.contains(q)).all()
    users = User.query.filter(User.name.contains(q), User.role == 'user').all()
    staff = User.query.filter(User.name.contains(q), User.role == 'staff').all()
    return render_template('admin/search_results.html', q=q, treks=treks, users=users, staff=staff)
```

### Step 4.3 — Create Trek Management Template

```html
<!-- templates/admin/manage_treks.html -->
{% extends "base.html" %}
{% block title %}Manage Treks{% endblock %}
{% block content %}
<div class="d-flex justify-content-between mb-3">
    <h3> All Treks</h3>
    <a href="/admin/trek/add" class="btn btn-success">+ Add Trek</a>
</div>

<table class="table table-striped table-hover">
    <thead class="table-dark">
        <tr>
            <th>ID</th><th>Name</th><th>Location</th><th>Difficulty</th>
            <th>Slots</th><th>Status</th><th>Staff</th><th>Actions</th>
        </tr>
    </thead>
    <tbody>
        {% for trek in treks %}
        <tr>
            <td>{{ trek.id }}</td>
            <td>{{ trek.name }}</td>
            <td>{{ trek.location }}</td>
            <td>
                <span class="badge 
                    {% if trek.difficulty == 'Easy' %}bg-success
                    {% elif trek.difficulty == 'Moderate' %}bg-warning text-dark
                    {% else %}bg-danger{% endif %}">
                    {{ trek.difficulty }}
                </span>
            </td>
            <td>{{ trek.get_remaining_slots() }}/{{ trek.available_slots }}</td>
            <td><span class="badge bg-info">{{ trek.status }}</span></td>
            <td>
                {% if trek.assigned_staff %}
                    {{ trek.assigned_staff.name }}
                {% else %}
                    <span class="text-muted">Unassigned</span>
                {% endif %}
            </td>
            <td>
                <a href="/admin/trek/edit/{{ trek.id }}" class="btn btn-sm btn-warning">Edit</a>
                <a href="/admin/trek/assign/{{ trek.id }}" class="btn btn-sm btn-info">Assign</a>
                <a href="/admin/trek/delete/{{ trek.id }}" class="btn btn-sm btn-danger"
                   onclick="return confirm('Delete this trek?')">Delete</a>
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endblock %}
```

---

## PHASE 5: TREK STAFF DASHBOARD (Day 14-18)

### Step 5.1 — Create `templates/staff/dashboard.html`

```html
<!-- templates/staff/dashboard.html -->
{% extends "base.html" %}
{% block title %}Staff Dashboard{% endblock %}
{% block content %}
<h2> Staff Dashboard — Welcome, {{ session.user_name }}</h2>

{% if assigned_treks %}
    {% for trek in assigned_treks %}
    <div class="card mb-3">
        <div class="card-header d-flex justify-content-between">
            <strong>{{ trek.name }}</strong>
            <span class="badge bg-{{ 'success' if trek.status == 'Open' else 'secondary' }}">
                {{ trek.status }}
            </span>
        </div>
        <div class="card-body">
            <p> {{ trek.location }} |  {{ trek.duration_days }} days | 
                {{ trek.get_remaining_slots() }}/{{ trek.available_slots }} slots left</p>
            
            <!-- Update Trek Form -->
            <form method="POST" action="/staff/update_trek/{{ trek.id }}" class="row g-2">
                <div class="col-md-3">
                    <label class="form-label">Available Slots</label>
                    <input type="number" name="available_slots" class="form-control" 
                           value="{{ trek.available_slots }}" min="0">
                </div>
                <div class="col-md-3">
                    <label class="form-label">Status</label>
                    <select name="status" class="form-select">
                        {% for s in ['Open', 'Closed', 'Completed'] %}
                            <option value="{{ s }}" {{ 'selected' if trek.status == s }}>{{ s }}</option>
                        {% endfor %}
                    </select>
                </div>
                <div class="col-md-2 d-flex align-items-end">
                    <button type="submit" class="btn btn-primary">Update</button>
                </div>
            </form>
            
            <!-- Registered Trekkers -->
            <h6 class="mt-3">Registered Trekkers ({{ trek.bookings|selectattr('status', 'eq', 'Booked')|list|length }}):</h6>
            <ul class="list-group">
                {% for booking in trek.bookings %}
                    {% if booking.status == 'Booked' %}
                    <li class="list-group-item d-flex justify-content-between">
                        {{ booking.trekker.name }} ({{ booking.trekker.email }})
                        <span class="badge bg-success">{{ booking.status }}</span>
                    </li>
                    {% endif %}
                {% endfor %}
            </ul>
        </div>
    </div>
    {% endfor %}
{% else %}
    <div class="alert alert-info">No treks assigned to you yet.</div>
{% endif %}
{% endblock %}
```

### Step 5.2 — Add Staff Routes to `app.py`

```python
# ────── STAFF ROUTES ──────

@app.route('/staff/dashboard')
@login_required
@staff_required
def staff_dashboard():
    user_id = session['user_id']
    assigned_treks = Trek.query.filter_by(assigned_staff_id=user_id).all()
    return render_template('staff/dashboard.html', assigned_treks=assigned_treks)

@app.route('/staff/update_trek/<int:trek_id>', methods=['POST'])
@login_required
@staff_required
def staff_update_trek(trek_id):
    trek = Trek.query.get_or_404(trek_id)
    
    # Security: ensure staff can only modify their own trek
    if trek.assigned_staff_id != session['user_id']:
        flash('You are not assigned to this trek.', 'danger')
        return redirect('/staff/dashboard')
    
    trek.available_slots = int(request.form['available_slots'])
    trek.status = request.form['status']
    db.session.commit()
    flash('Trek updated successfully!', 'success')
    return redirect('/staff/dashboard')
```

---

## PHASE 6: USER DASHBOARD + BOOKING (Day 18-24)

### Step 6.1 — Create `templates/user/dashboard.html`

```html
<!-- templates/user/dashboard.html -->
{% extends "base.html" %}
{% block title %}My Dashboard{% endblock %}
{% block content %}
<h2> Welcome, {{ session.user_name }}!</h2>

<div class="row mb-4">
    <div class="col-md-4">
        <a href="/user/treks" class="btn btn-outline-success w-100">Browse Open Treks</a>
    </div>
    <div class="col-md-4">
        <a href="/user/bookings" class="btn btn-outline-primary w-100">My Bookings</a>
    </div>
    <div class="col-md-4">
        <a href="/user/profile" class="btn btn-outline-secondary w-100">Edit Profile</a>
    </div>
</div>

<!-- Recent Bookings -->
<h4>Recent Bookings</h4>
{% if recent_bookings %}
<table class="table">
    <thead><tr><th>Trek</th><th>Date Booked</th><th>Status</th><th>Action</th></tr></thead>
    <tbody>
        {% for b in recent_bookings %}
        <tr>
            <td>{{ b.trek.name }}</td>
            <td>{{ b.booking_date.strftime('%d %b %Y') }}</td>
            <td><span class="badge bg-{{ 'success' if b.status=='Booked' else 'secondary' }}">{{ b.status }}</span></td>
            <td>
                {% if b.status == 'Booked' %}
                <a href="/user/cancel_booking/{{ b.id }}" class="btn btn-sm btn-danger"
                   onclick="return confirm('Cancel this booking?')">Cancel</a>
                {% endif %}
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% else %}
    <p class="text-muted">No bookings yet. <a href="/user/treks">Explore treks!</a></p>
{% endif %}
{% endblock %}
```

### Step 6.2 — Create `templates/user/browse_treks.html`

```html
<!-- templates/user/browse_treks.html -->
{% extends "base.html" %}
{% block title %}Browse Treks{% endblock %}
{% block content %}
<h3> Available Treks</h3>

<!-- Search & Filter -->
<form method="GET" action="/user/treks" class="row g-2 mb-4">
    <div class="col-md-4">
        <input type="text" name="q" class="form-control" 
               placeholder="Search by name or location..." value="{{ q }}">
    </div>
    <div class="col-md-3">
        <select name="difficulty" class="form-select">
            <option value="">All Difficulties</option>
            <option value="Easy" {{ 'selected' if difficulty=='Easy' }}>Easy</option>
            <option value="Moderate" {{ 'selected' if difficulty=='Moderate' }}>Moderate</option>
            <option value="Hard" {{ 'selected' if difficulty=='Hard' }}>Hard</option>
        </select>
    </div>
    <div class="col-md-2">
        <button type="submit" class="btn btn-dark w-100">Filter</button>
    </div>
</form>

<div class="row">
{% for trek in treks %}
<div class="col-md-4 mb-3">
    <div class="card h-100 shadow-sm">
        <div class="card-body">
            <h5 class="card-title">{{ trek.name }}</h5>
            <p class="text-muted"> {{ trek.location }}</p>
            <p>
                <span class="badge 
                    {% if trek.difficulty == 'Easy' %}bg-success
                    {% elif trek.difficulty == 'Moderate' %}bg-warning text-dark
                    {% else %}bg-danger{% endif %}">
                    {{ trek.difficulty }}
                </span>
                <span class="ms-2"> {{ trek.duration_days }} days</span>
            </p>
            <p> {{ trek.get_remaining_slots() }} slots left</p>
            <p> {{ trek.start_date }} → {{ trek.end_date }}</p>
        </div>
        <div class="card-footer">
            {% if already_booked(trek.id) %}
                <span class="btn btn-secondary w-100 disabled">Already Booked</span>
            {% elif trek.get_remaining_slots() <= 0 %}
                <span class="btn btn-secondary w-100 disabled">Fully Booked</span>
            {% else %}
                <a href="/user/book/{{ trek.id }}" class="btn btn-success w-100">Book Now</a>
            {% endif %}
        </div>
    </div>
</div>
{% endfor %}
</div>
{% endblock %}
```

### Step 6.3 — Add User Routes to `app.py`

```python
# ────── USER ROUTES ──────

@app.route('/user/dashboard')
@login_required
def user_dashboard():
    if session.get('role') != 'user':
        return redirect('/')
    user_id = session['user_id']
    recent_bookings = Booking.query.filter_by(user_id=user_id).order_by(
        Booking.booking_date.desc()).limit(5).all()
    return render_template('user/dashboard.html', recent_bookings=recent_bookings)

@app.route('/user/treks')
@login_required
def browse_treks():
    q = request.args.get('q', '')
    difficulty = request.args.get('difficulty', '')
    
    query = Trek.query.filter_by(status='Open')
    if q:
        query = query.filter(
            (Trek.name.contains(q)) | (Trek.location.contains(q))
        )
    if difficulty:
        query = query.filter_by(difficulty=difficulty)
    
    treks = query.all()
    user_id = session['user_id']
    
    # Helper to check if user already booked
    def already_booked(trek_id):
        return Booking.query.filter_by(
            user_id=user_id, trek_id=trek_id, status='Booked'
        ).first() is not None
    
    return render_template('user/browse_treks.html', treks=treks, q=q, 
                          difficulty=difficulty, already_booked=already_booked)

@app.route('/user/book/<int:trek_id>')
@login_required
def book_trek(trek_id):
    if session.get('role') != 'user':
        return redirect('/')
    
    trek = Trek.query.get_or_404(trek_id)
    user_id = session['user_id']
    
    # Validations
    if trek.status != 'Open':
        flash('This trek is not open for booking.', 'danger')
        return redirect('/user/treks')
    
    if trek.get_remaining_slots() <= 0:
        flash('No slots available for this trek.', 'danger')
        return redirect('/user/treks')
    
    existing = Booking.query.filter_by(user_id=user_id, trek_id=trek_id, status='Booked').first()
    if existing:
        flash('You have already booked this trek.', 'warning')
        return redirect('/user/treks')
    
    # Create booking
    booking = Booking(user_id=user_id, trek_id=trek_id, status='Booked')
    db.session.add(booking)
    db.session.commit()
    flash(f'Successfully booked {trek.name}!', 'success')
    return redirect('/user/bookings')

@app.route('/user/bookings')
@login_required
def user_bookings():
    user_id = session['user_id']
    bookings = Booking.query.filter_by(user_id=user_id).order_by(
        Booking.booking_date.desc()).all()
    return render_template('user/booking_history.html', bookings=bookings)

@app.route('/user/cancel_booking/<int:booking_id>')
@login_required
def cancel_booking(booking_id):
    booking = Booking.query.get_or_404(booking_id)
    
    # Security: user can only cancel their own booking
    if booking.user_id != session['user_id']:
        flash('Unauthorized.', 'danger')
        return redirect('/user/bookings')
    
    booking.status = 'Cancelled'
    db.session.commit()
    flash('Booking cancelled.', 'info')
    return redirect('/user/bookings')

@app.route('/user/profile', methods=['GET', 'POST'])
@login_required
def user_profile():
    user = User.query.get(session['user_id'])
    if request.method == 'POST':
        user.name = request.form['name']
        if request.form.get('password'):
            user.set_password(request.form['password'])
        db.session.commit()
        session['user_name'] = user.name
        flash('Profile updated!', 'success')
    return render_template('user/profile.html', user=user)
```

---

## PHASE 7: BOOKING HISTORY & STATUS TRACKING (Day 24-26)

### Step 7.1 — Create `templates/user/booking_history.html`

```html
<!-- templates/user/booking_history.html -->
{% extends "base.html" %}
{% block title %}My Trek History{% endblock %}
{% block content %}
<h3> My Trekking History</h3>

{% if bookings %}
<table class="table table-hover">
    <thead class="table-dark">
        <tr>
            <th>#</th><th>Trek Name</th><th>Location</th><th>Difficulty</th>
            <th>Date Booked</th><th>Trek Status</th><th>Booking Status</th><th>Action</th>
        </tr>
    </thead>
    <tbody>
        {% for b in bookings %}
        <tr>
            <td>{{ b.id }}</td>
            <td>{{ b.trek.name }}</td>
            <td>{{ b.trek.location }}</td>
            <td>{{ b.trek.difficulty }}</td>
            <td>{{ b.booking_date.strftime('%d %b %Y') }}</td>
            <td>{{ b.trek.status }}</td>
            <td>
                <span class="badge 
                    {% if b.status == 'Booked' %}bg-success
                    {% elif b.status == 'Cancelled' %}bg-secondary
                    {% else %}bg-primary{% endif %}">
                    {{ b.status }}
                </span>
            </td>
            <td>
                {% if b.status == 'Booked' %}
                    <a href="/user/cancel_booking/{{ b.id }}" class="btn btn-sm btn-outline-danger"
                       onclick="return confirm('Cancel booking?')">Cancel</a>
                {% endif %}
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% else %}
    <div class="alert alert-info">No treks booked yet. <a href="/user/treks">Explore now!</a></div>
{% endif %}
{% endblock %}
```

---

## PHASE 8: GITHUB SETUP (Do This on Day 1 — Required Before Submission)

```bash
# 1. Create a GitHub account at github.com

# 2. Create a new PRIVATE repository named "trekking-management-app"

# 3. In VS Code terminal:
git init
git add .
git commit -m "Milestone-0 TMA-MAD-1"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/trekking-management-app.git
git push -u origin main

# 4. Add collaborator: MADI-cs2003
# Go to repo → Settings → Collaborators → Add people → type "MADI-cs2003"
```

### Commit After Each Milestone:
```bash
git add .
git commit -m "Milestone: Database Models and Schema Setup"
git push

# Later...
git add .
git commit -m "Milestone: Authentication and Role-Based Access"
git push
```

---

## PHASE 9: OPTIONAL FEATURES (Day 26-32 — Do These for Better Score)

### 9.1 — Add Flask-Login (Recommended)

```bash
pip install flask-login
```

```python
# In app.py
from flask_login import LoginManager, UserMixin, login_user, logout_user, current_user

login_manager = LoginManager()
login_manager.init_app(app)

# Make User model inherit UserMixin
class User(db.Model, UserMixin):
    # ... (rest of model stays same)
    pass
```

### 9.2 — Add Charts (Chart.js — Optional)

In your admin dashboard template, add at the bottom:

```html
<canvas id="bookingChart" width="400" height="200"></canvas>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
const ctx = document.getElementById('bookingChart').getContext('2d');
new Chart(ctx, {
    type: 'bar',
    data: {
        labels: {{ trek_names | tojson }},
        datasets: [{
            label: 'Bookings per Trek',
            data: {{ booking_counts | tojson }},
            backgroundColor: 'rgba(54, 162, 235, 0.6)'
        }]
    }
});
</script>
```

### 9.3 — Add API Endpoints (Optional)

```python
# Simple JSON API example
@app.route('/api/treks')
def api_treks():
    from flask import jsonify
    treks = Trek.query.filter_by(status='Open').all()
    result = [{
        'id': t.id,
        'name': t.name,
        'location': t.location,
        'difficulty': t.difficulty,
        'available_slots': t.get_remaining_slots()
    } for t in treks]
    return jsonify(result)

@app.route('/api/treks/<int:trek_id>')
def api_trek_detail(trek_id):
    from flask import jsonify
    trek = Trek.query.get_or_404(trek_id)
    return jsonify({
        'id': trek.id,
        'name': trek.name,
        'location': trek.location,
        'difficulty': trek.difficulty,
        'available_slots': trek.get_remaining_slots(),
        'status': trek.status
    })
```

---

## PHASE 10: TESTING CHECKLIST (Day 32-34)

Go through every feature and test it:

```
Auth:
[ ] Register as User → Login → See user dashboard
[ ] Register as Staff → Login → See "pending approval" message
[ ] Admin login → See admin dashboard
[ ] Blacklisted user cannot login

Admin:
[ ] Create a trek
[ ] Edit a trek
[ ] Delete a trek
[ ] Approve staff → staff can now login
[ ] Assign staff to trek
[ ] Blacklist user
[ ] Search for trek/user/staff

Staff:
[ ] See only assigned treks
[ ] Update trek slots
[ ] Change trek status to Open/Closed/Completed
[ ] View registered trekkers

User:
[ ] See only Open treks
[ ] Filter by difficulty/location
[ ] Book a trek → slot decreases by 1
[ ] Can't book same trek twice
[ ] Can't book fully booked trek
[ ] Cancel booking
[ ] View booking history
```

---

## PHASE 11: FINAL SUBMISSION (Day 34-38)

### Step 11.1 — Generate Checksum

```bash
pip install checksumdir

# Create check.py in folder ABOVE your project
# check.py
import checksumdir
hash = checksumdir.dirhash("trekking_app")  # Replace with your folder name
print("Directory Checksum:", hash)

python check.py  # Note down this hash
```

### Step 11.2 — Create ZIP File

```bash
# On Windows:
# Right-click your project folder → Send to → Compressed (zipped) folder

# On Mac/Linux:
zip -r trekking_app.zip trekking_app/

# Structure should be:
# trekking_app.zip
# └── trekking_app/
#   ├── app.py
#   ├── models.py
#   └── ...
```

### Step 11.3 — Write Project Report (3-5 Pages)

Your report must include:
1. **Student Details** — Name, Roll No, Email
2. **Problem Statement** — Brief summary of the project
3. **Your Approach** — How you structured the solution
4. **AI/LLM Declaration** — How much % AI was used (mandatory even if 0%)
5. **Frameworks Used** — Flask, Jinja2, SQLite, Bootstrap 5
6. **ER Diagram** — Screenshot from draw.io
7. **API Endpoints** — Table of your routes (if implemented)
8. **Google Drive Video Link** — 5-10 min demo video

### Step 11.4 — Record Demo Video (5-10 minutes)

Cover in this order:
1. Intro (30 sec) — "I built a Trekking Management App using Flask..."
2. Approach (30 sec) — "I started with ER design, then models, then auth..."
3. Demo (90 sec each):
   - Admin flow: login → add trek → approve staff → assign staff
   - Staff flow: login → view trek → update slots/status
   - User flow: register → login → browse → book → view history
4. Any extra features (30 sec)

Upload to Google Drive → Share → "Anyone with link" → Paste link in report.

---

## FINAL FOLDER STRUCTURE

```
trekking_app/
├── app.py
├── config.py
├── models.py
├── requirements.txt
├── static/
│   └── css/style.css
└── templates/
    ├── base.html
    ├── index.html
    ├── auth/
    │   ├── login.html
    │   └── register.html
    ├── admin/
    │   ├── dashboard.html
    │   ├── manage_treks.html
    │   ├── add_trek.html
    │   ├── edit_trek.html
    │   ├── assign_staff.html
    │   ├── manage_staff.html
    │   ├── manage_users.html
    │   └── search_results.html
    ├── staff/
    │   └── dashboard.html
    └── user/
        ├── dashboard.html
        ├── browse_treks.html
        ├── booking_history.html
        └── profile.html
```

---

## COMMON ERRORS AND FIXES

| Error | Fix |
|-------|-----|
| `ModuleNotFoundError: flask` | Run `pip install flask flask-sqlalchemy` |
| `OperationalError: no such table` | You didn't run `db.create_all()` inside app context |
| `KeyError: 'user_id'` | Session expired — add `@login_required` decorator |
| Template not found | Check spelling and that file is in `templates/` folder |
| `jinja2.UndefinedError` | Variable not passed to `render_template()` |
| Static files not loading | Use `url_for('static', filename='css/style.css')` |
| `IntegrityError` | Duplicate email — email must be unique |

---

## HOW TO RUN THE APP

```bash
# 1. Go to project folder
cd trekking_app

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run the app
python app.py

# 4. Open browser
# Go to: http://127.0.0.1:5000

# Default admin credentials:
# Email: admin@trek.com
# Password: admin123
```

---

## MILESTONE COMMIT MESSAGES (Copy Exactly)

| Milestone | Commit Message |
|-----------|---------------|
| 0 — GitHub Setup | `Milestone-0 TMA-MAD-1` |
| DB Models | `Milestone: Database Models and Schema Setup` |
| Auth | `Milestone: Authentication and Role-Based Access` |
| Admin Dashboard | `Milestone: Admin Dashboard and Management` |
| Staff Dashboard | `Milestone: Trek Staff Dashboard and Trek Management` |
| User Dashboard | `Milestone: User Dashboard and Trek Booking System` |
| Booking History | `Milestone: Trek Booking History and Trek Status Tracking` |
| Final | `Milestone-TMA Final-Submission` |

---

*Good luck! Start with GitHub setup + ER diagram today, then follow each phase. If you get stuck anywhere, come back and ask — happy to help with specific parts!* 
