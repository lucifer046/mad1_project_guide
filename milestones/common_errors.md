## COMMON ERRORS & WHY THEY HAPPEN

Understanding *why* errors occur makes you a better developer than just knowing the fix.

---

### Error: `ModuleNotFoundError: No module named 'flask'`

**What it means:**  
Python can't find the Flask library. Libraries must be explicitly installed before use.

**Why it happens:**  
You may be running Python in a different environment than where you installed Flask. This is common with virtual environments.

**Fix:**
```bash
pip install flask flask-sqlalchemy werkzeug
```
Or install from the requirements file:
```bash
pip install -r requirements.txt
```

---

### Error: `OperationalError: no such table: user`

**What it means:**  
SQLAlchemy tried to query a table that doesn't exist in the database file yet.

**Why it happens:**  
`db.create_all()` was never called inside an active application context. The `.db` file either doesn't exist or was created before your models were imported.

**Fix — Make sure your `app.py` has:**
```python
with app.app_context():
    db.create_all()   # ← This must run at startup
```
Also make sure all model files are imported BEFORE `db.create_all()` is called, otherwise SQLAlchemy doesn't know about them.

---

### Error: `KeyError: 'user_id'` (or `session['user_id']`)

**What it means:**  
Your code tried to read `session['user_id']` but that key doesn't exist in the session dictionary.

**Why it happens:**  
The user is not logged in (session was never set), or the session expired (browser closed, SECRET_KEY changed, etc.).

**Fix:**  
Use `session.get('user_id')` instead of `session['user_id']` to avoid the crash.  
More importantly, add `@login_required` to every route that needs authentication.

```python
# Safe way to read session:
user_id = session.get('user_id')  # Returns None if not found
if not user_id:
    return redirect('/login')
```

---

### Error: `TemplateNotFound: admin/dashboard.html`

**What it means:**  
Flask looked for the template file but couldn't find it at the expected path.

**Why it happens:**  
1. Typo in the file name or path
2. File is in the wrong folder
3. File doesn't exist yet

**Fix:**  
Check that your file is at: `templates/admin/dashboard.html`  
Flask always looks for templates inside the `templates/` folder.

```
trekking_app/
└── templates/         ← Flask looks here
    ├── base.html
    └── admin/
        └── dashboard.html   ← Must match exactly: 'admin/dashboard.html'
```

---

### Error: `jinja2.UndefinedError: 'treks' is undefined`

**What it means:**  
The template used `{{ treks }}` but `treks` was never passed to it.

**Why it happens:**  
You forgot to pass the variable in `render_template()`.

```python
# WRONG — template gets nothing
return render_template('admin/manage_treks.html')

# CORRECT — pass the variable
treks = Trek.query.all()
return render_template('admin/manage_treks.html', treks=treks)
```

The variable name in `render_template()` must match the name used in the template.

---

### Error: Static files (CSS/Images) not loading

**What it means:**  
The browser gets a 404 when trying to load your CSS file.

**Why it happens:**  
Hardcoded paths like `href="/static/css/style.css"` may break in different deployment environments.

**Fix — Always use `url_for()`:**
```html
<!-- WRONG — hardcoded, fragile -->
<link rel="stylesheet" href="/static/css/style.css">

<!-- CORRECT — Flask generates the correct URL dynamically -->
<link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
```

Also verify your file is at: `static/css/style.css`

---

### Error: `IntegrityError: UNIQUE constraint failed: user.email`

**What it means:**  
You tried to insert a User with an email that already exists in the database.

**Why it happens:**  
The `email` column is defined with `unique=True`. Duplicate emails violate this constraint.

**Fix — Check before inserting:**
```python
existing = User.query.filter_by(email=email).first()
if existing:
    flash('Email already registered.', 'danger')
    return render_template('auth/register.html')

# Only create user if email is new
user = User(email=email, ...)
```

---

### Error: `IntegrityError: UNIQUE constraint failed: bookings.user_id, bookings.trek_id`

**What it means:**  
A user tried to book the same trek twice. The composite unique constraint rejected the duplicate.

**Why it happens:**  
The `Booking` model has `UniqueConstraint('user_id', 'trek_id')`. This is working correctly!

**Fix — Check for existing booking before creating a new one:**
```python
existing = Booking.query.filter_by(user_id=user_id, trek_id=trek_id, status='Booked').first()
if existing:
    flash('You have already booked this trek.', 'warning')
    return redirect('/user/treks')
```

---

### Error: `RuntimeError: No application found` (when using db outside request)

**What it means:**  
SQLAlchemy tried to run a query but there's no active Flask application context.

**Why it happens:**  
Operations like `db.create_all()` run outside of a web request and need the application context pushed manually.

**Fix:**
```python
with app.app_context():
    db.create_all()
    # Any SQLAlchemy operations go here
```