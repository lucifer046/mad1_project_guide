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