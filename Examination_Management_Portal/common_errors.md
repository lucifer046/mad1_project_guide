## COMMON ERRORS & HOW TO FIX THEM

Understanding why errors happen is the fastest way to debug your Flask application.

---

### 1. `ModuleNotFoundError: No module named 'flask'`
* **Why it happens**: Python cannot find the Flask library. Either your virtual environment is not activated, or packages were not installed in the current environment.
* **Fix**:
  ```bash
  # Activate venv:
  source venv/bin/activate   # Linux/Mac
  venv\Scripts\activate    # Windows
  pip install -r requirements.txt
  ```

---

### 2. `OperationalError: no such table: user`
* **Why it happens**: SQLAlchemy tried to query a table that does not exist in SQLite. `db.create_all()` was not called within the Flask app context.
* **Fix**:
  Ensure `app.py` has:
  ```python
  with app.app_context():
      db.create_all()
  ```
  And verify that all models are imported before calling `create_all()`.

---

### 3. `IntegrityError: UNIQUE constraint failed: user.email`
* **Why it happens**: Code attempted to insert a record whose unique field (e.g., email or course code) already exists in the database.
* **Fix**:
  Always check for existing records before calling `db.session.add()`:
  ```python
  if User.query.filter_by(email=email).first():
      flash('Email already registered.', 'danger')
      return redirect(url_for('auth.register_student'))
  ```

---

### 4. `BuildError: Could not build url for endpoint 'admin.dashboard'`
* **Why it happens**: The name passed to `url_for()` does not match the registered Blueprint name and view function name.
* **Fix**:
  * Blueprint registration in `app.py`: `app.register_blueprint(admin_bp, url_prefix='/admin')` (where `admin_bp = Blueprint('admin', __name__)`).
  * Endpoint in `url_for`: `url_for('admin.dashboard')` (format is `blueprint_name.function_name`).

---

### 5. `TemplateNotFound: admin/dashboard.html`
* **Why it happens**: Flask cannot locate the template inside the `templates/` folder.
* **Fix**:
  Check directory nesting: `templates/admin/dashboard.html`. Ensure exact case sensitivity on Linux.

---

### 6. `ValueError: Invalid date format`
* **Why it happens**: Form submissions from `<input type="datetime-local">` produce strings in the format `YYYY-MM-DDTHH:MM`, which fails simple string parsing.
* **Fix**:
  Use the parsing helper from `utils.py`:
  ```python
  from utils import parse_dt
  start_dt = parse_dt(request.form.get('start_time'))
  ```

---

### 7. IDOR / Horizontal Privilege Vulnerabilities
* **Why it happens**: Trusting user-supplied IDs from URL paths without checking ownership.
* **Fix**:
  Always verify ownership:
  ```python
  slot = Slot.query.get_or_404(slot_id)
  if slot.examiner_id != current_user.id:
      flash('Unauthorized.', 'danger')
      return redirect(url_for('examiner.slots'))
  ```
