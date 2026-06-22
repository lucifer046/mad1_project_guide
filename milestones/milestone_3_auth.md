## PHASE 3: AUTHENTICATION (Day 5–8)

> 📺 Watch: [Flask Login System Hindi](https://www.youtube.com/watch?v=71EU8gnZqZQ)

---

## 🧠 Big Picture: What is Authentication?

**Authentication** = "Who are you?" — Verifying a user's identity.  
**Authorization** = "What can you do?" — Deciding what that identity is allowed to access.

These are two different things! Many beginners confuse them.

```
Authentication: "I am Raj, and my password proves it." → Login check
Authorization:  "Raj has role=user, so he cannot access /admin/dashboard" → Role check
```

Our system needs BOTH. In this phase, we handle authentication (login/register/logout). In Phase 4, we enforce authorization (admin-only routes, staff-only routes).

---

## 🌐 Understanding HTTP and Why Sessions Exist

HTTP (the protocol browsers use to talk to servers) is **stateless** by design.

This means: **every single request is treated as if it came from a stranger.**

```
Browser → GET /dashboard → Server sees: "Who are you? I don't know you."
Browser → POST /login    → Server sees: "Who are you? I don't know you."
```

After a user logs in, the server processes the login... and then **forgets** them on the next request. This is a problem! You'd have to log in for every single page.

**The solution: Cookies + Sessions**

```
1. User logs in with correct password
2. Server creates a "session" — a dictionary of safe data: {user_id: 2, role: 'user'}
3. Server SIGNS this data with SECRET_KEY → creates encrypted session cookie
4. Server sends cookie to browser
5. Browser stores cookie
6. On every future request, browser automatically sends cookie
7. Server reads cookie, VERIFIES the signature, trusts the data inside
8. Server now knows "This is user #2 with role=user"
```

This is why your SECRET_KEY is so important — it's what makes sessions tamper-proof.

Flask's `session` object is just a Python dictionary that automatically gets serialized into a signed cookie:

```python
session['user_id'] = user.id   # Writing to session
user_id = session['user_id']   # Reading from session
session.clear()                # Deletes session (logout)
```

---

## 🏗️ Understanding Jinja2 Template Inheritance

When building web pages, every page needs:
- The same `<head>` section (CSS, fonts, meta tags)
- The same navigation bar
- The same footer
- The same flash message display

If you copy-paste this HTML into every file, you'll have a maintenance nightmare. Change the navbar? Update 20 files.

**Jinja2 solves this with Template Inheritance** — just like class inheritance in Python.

```
base.html                    ← Parent template (the skeleton)
  ├── login.html             ← Child: fills in the "content" block
  ├── register.html          ← Child: fills in the "content" block
  └── dashboard.html         ← Child: fills in the "content" block
```

**How it works:**

In `base.html`, you define **blocks** — named placeholders:
```html
{% block content %}
<!-- Child templates replace this -->
{% endblock %}
```

In child templates:
```html
{% extends "base.html" %}       {# Inherit from parent #}
{% block content %}
  <h1>This replaces the placeholder</h1>
{% endblock %}
```

---

## Step 3.1 — Create `templates/base.html`

### Line-by-Line Explanation

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <!--
        viewport meta tag: Critical for mobile responsiveness.
        Without it, mobile browsers zoom out to show the "desktop" view.
        width=device-width: use the device's actual screen width
        initial-scale=1.0: don't zoom in or out by default
    -->
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <!--
        {% block title %} is a named placeholder.
        Child templates override this to set their own page titles.
        The text "Trekking App" is the DEFAULT if a child doesn't override.
    -->
    <title>{% block title %}Trekking App{% endblock %}</title>

    <!-- Bootstrap 5: A CSS framework that provides pre-built components -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">

    <!--
        url_for('static', filename='css/style.css'):
        Flask generates the correct URL to your static files.
        NEVER hardcode paths like '/static/css/style.css' — 
        url_for() handles different deployment environments automatically.
    -->
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
</head>
<body>
    <!-- Navigation Bar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand" href="/"> TrekkingApp</a>
            <div class="navbar-nav ms-auto">
                <!--
                    Jinja2 conditional: Check if user is logged in.
                    session is the Flask session dict — if user_id is in it,
                    they are authenticated.
                -->
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

    <!--
        Flash Messages System:
        Flash messages are one-time notifications stored in the session.
        They are displayed ONCE and then deleted automatically.
        Use flash('Your message', 'category') in Python routes.
        Categories map to Bootstrap alert colors: 'success', 'danger', 'warning', 'info'
    -->
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

    <!-- Main Content Block — Every child page fills this in -->
    <div class="container mt-4">
        {% block content %}{% endblock %}
    </div>

    <!-- Bootstrap JavaScript (needed for dropdowns, modals, alerts) -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

---

## Step 3.2 — Create `templates/auth/login.html`

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
                <!--
                    method="POST": Form data is sent in the request body (hidden from URL).
                    action="/login": Which route to send the form data to.
                    NEVER use method="GET" for sensitive forms (passwords would appear in URL!)
                -->
                <form method="POST" action="/login">
                    <div class="mb-3">
                        <label class="form-label">Email</label>
                        <input type="email" name="email" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Password</label>
                        <!--
                            type="password": Browser replaces typed characters with dots.
                            The actual text is still sent in the POST body.
                        -->
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

---

## Step 3.3 — Create `templates/auth/register.html`

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

---

## Step 3.4 — Add Auth Routes & Access Control to `app.py`

### 📖 Understanding Python Decorators (The `@` Symbol)

Decorators are one of the most powerful Python features. Let's understand them step by step.

**First, understand that Python functions are "first-class objects"** — they can be passed around like variables:

```python
def say_hello():
    print("Hello!")

greeting = say_hello   # Assign function to variable
greeting()             # Call it → prints "Hello!"
```

**A decorator is a function that takes another function and wraps it:**

```python
def my_decorator(func):
    def wrapper():
        print("Before the function")
        func()                       # Call the original function
        print("After the function")
    return wrapper                   # Return the WRAPPER, not the result

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# Output:
# Before the function
# Hello!
# After the function
```

The `@my_decorator` line is equivalent to: `say_hello = my_decorator(say_hello)`

**Now apply this to Flask route protection:**

```python
def login_required(f):
    @wraps(f)              # Preserve original function name/docstring
    def decorated(*args, **kwargs):
        if 'user_id' not in session:
            return redirect('/login')   # Short-circuit: stop here
        return f(*args, **kwargs)       # Pass-through: call original route
    return decorated
```

When you write:
```python
@login_required
def dashboard():
    ...
```

It means: `dashboard = login_required(dashboard)`

So when someone visits `/dashboard`, Flask calls the **wrapped** version, which first checks the session, THEN calls the original function.

**Why `@wraps(f)`?** Without it, all decorated functions have the same internal name (`decorated`). Flask uses function names as endpoint identifiers — if two routes have the same name, it crashes. `@wraps(f)` copies the original function's `__name__` to the wrapper.

### 📖 The HTTP Request-Response Cycle

Every time a browser visits a URL, this happens:

```
Browser                          Flask Server
  │                                  │
  │──── HTTP Request ───────────────>│
  │     GET /login                   │   ① Flask matches URL to route function
  │     Headers: {Cookie: ...}       │   ② Runs the function
  │                                  │   ③ Returns HTML
  │<─── HTTP Response ───────────────│
  │     200 OK                       │
  │     Body: <html>...</html>       │
```

For form submissions:
```
Browser                          Flask Server
  │──── HTTP Request ───────────────>│
  │     POST /login                  │   ① Flask runs login() route
  │     Body: email=x&password=y     │   ② request.form['email'] reads the body
  │                                  │   ③ Validates, sets session, redirects
  │<─── HTTP Response ───────────────│
  │     302 Redirect to /dashboard   │
```

`request.method == 'POST'` lets us handle both cases (showing the form vs processing it) in one function.

```python
# ───────────────────────────────────────────────────────────────────
# HOW THIS FITS INTO app.py:
#
# These imports are already included at the top of app.py from Phase 2.
# ───────────────────────────────────────────────────────────────────
# from flask import Flask, render_template, request, redirect, session, flash
# from functools import wraps
# from datetime import datetime
# from config import Config
# from models import db, User, Staff, Trek, Booking
#
# After that: create_admin(), create_app(), app = create_app()
#
# Then below app = create_app(), add the decorators and routes below:
# ───────────────────────────────────────────────────────────────────


# ═══════════════════════════════════════════════════════
# DECORATORS — Access Control Middleware
# ═══════════════════════════════════════════════════════

def login_required(f):
    """
    Decorator: Blocks access to a route if the user is not logged in.
    
    HOW IT WORKS:
    - Checks if 'user_id' is in Flask's session dictionary
    - If YES: continue to the actual route function
    - If NO: redirect to /login with a warning flash message
    
    USAGE: Add @login_required above any route that needs auth
    """
    @wraps(f)
    def decorated(*args, **kwargs):
        if 'user_id' not in session:
            flash('Please login first.', 'warning')
            return redirect('/login')
        return f(*args, **kwargs)
    return decorated


def admin_required(f):
    """
    Decorator: Blocks access unless the logged-in user has role='admin'.
    
    Always stack AFTER @login_required:
        @login_required    ← checked FIRST (outer wrapper)
        @admin_required    ← checked SECOND (inner wrapper)
        def my_route():...
    
    Order matters! login_required is applied last (outermost),
    so it runs first when the route is called.
    """
    @wraps(f)
    def decorated(*args, **kwargs):
        if session.get('role') != 'admin':
            flash('Admin access required.', 'danger')
            return redirect('/')
        return f(*args, **kwargs)
    return decorated


def staff_required(f):
    """
    Decorator: Blocks access unless the logged-in user has role='staff'.
    """
    @wraps(f)
    def decorated(*args, **kwargs):
        if session.get('role') != 'staff':
            flash('Staff access required.', 'danger')
            return redirect('/')
        return f(*args, **kwargs)
    return decorated


# ═══════════════════════════════════════════════════════
# AUTH ROUTES
# ═══════════════════════════════════════════════════════

@app.route('/')
def index():
    return render_template('index.html')


@app.route('/login', methods=['GET', 'POST'])
def login():
    """
    Handles both:
    - GET /login  → Show the login form
    - POST /login → Process the submitted form data
    
    VALIDATION CHAIN (order matters):
    1. Do credentials (email+password) match?
    2. Is the account blacklisted?
    3. If staff: is the account approved by admin?
    4. Set session and redirect based on role.
    """
    if request.method == 'POST':
        email    = request.form.get('email')
        password = request.form.get('password')

        # Step 1: Find user by email
        # .first() returns the User object or None (doesn't raise error)
        user = User.query.filter_by(email=email).first()

        # Step 1b: Verify password using Werkzeug's hash comparison
        # check_password() is a method we defined in the User model
        if not user or not user.check_password(password):
            flash('Invalid email or password.', 'danger')
            return render_template('auth/login.html')
            # Note: We give a VAGUE error — don't tell attacker which part was wrong

        # Step 2: Check if account is blacklisted
        if user.is_blacklisted:
            flash('Your account has been blacklisted. Contact admin.', 'danger')
            return render_template('auth/login.html')

        # Step 3: Staff-specific approval check
        # Even if credentials are correct, staff must wait for admin approval
        if user.role == 'staff':
            profile = Staff.query.filter_by(user_id=user.id).first()
            if not profile or profile.approval_status != 'Approved':
                flash('Your staff account is pending admin approval.', 'warning')
                return render_template('auth/login.html')

        # Step 4: Authentication successful — create session
        # Store minimal info in session: enough to identify and authorize the user
        session['user_id']   = user.id
        session['user_name'] = user.name
        session['role']      = user.role

        # Step 5: Role-based redirect
        # Each role has their own "home base" dashboard
        if user.role == 'admin':
            return redirect('/admin/dashboard')
        elif user.role == 'staff':
            return redirect('/staff/dashboard')
        else:
            return redirect('/user/dashboard')

    # GET request: just show the form
    return render_template('auth/login.html')


@app.route('/register', methods=['GET', 'POST'])
def register():
    """
    New user registration.
    
    IMPORTANT SECURITY CHECKS:
    1. Email uniqueness — duplicate emails must be rejected
    2. Role whitelist — only 'user' and 'staff' are valid; 'admin' is forbidden
    3. Password hashing — never store plaintext
    4. Staff profile creation — if registering as staff, create associated Staff record
    5. db.session.flush() — needed to get user.id before final commit
    """
    if request.method == 'POST':
        name     = request.form.get('name')
        email    = request.form.get('email')
        password = request.form.get('password')
        role     = request.form.get('role')    # 'user' or 'staff'
        contact  = request.form.get('contact', '')

        # Check 1: Email uniqueness
        if User.query.filter_by(email=email).first():
            flash('Email already registered. Try logging in.', 'danger')
            return render_template('auth/register.html')

        # Check 2: Prevent privilege escalation via role manipulation
        # A malicious user could modify the form HTML to submit role='admin'
        if role == 'admin':
            flash('Cannot register as admin.', 'danger')
            return render_template('auth/register.html')

        # Create User object (in memory, not yet in DB)
        user = User(name=name, email=email, role=role)
        user.set_password(password)  # Hashes password before setting
        db.session.add(user)

        # db.session.flush():
        # Sends INSERT to DB and retrieves the auto-generated user.id
        # WITHOUT committing the transaction.
        # We need user.id to create the Staff profile (foreign key).
        # If we used commit() here, a crash during Staff creation would
        # leave an orphaned User with no Staff profile.
        db.session.flush()

        # If registering as staff, create the Staff profile record
        if role == 'staff':
            profile = Staff(user_id=user.id, contact=contact, approval_status='Pending')
            db.session.add(profile)
            flash('Registration successful! Wait for admin approval.', 'success')
        else:
            flash('Registration successful! You can now login.', 'success')

        # Commit both User and Staff (if applicable) in ONE atomic transaction
        db.session.commit()
        return redirect('/login')

    return render_template('auth/register.html')


@app.route('/logout')
def logout():
    """
    Clears the session cookie — effectively "forgetting" who the user is.
    After this, every route with @login_required will redirect to /login.
    """
    session.clear()
    flash('Logged out successfully.', 'info')
    return redirect('/login')
```