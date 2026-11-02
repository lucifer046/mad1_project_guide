## PHASE 4: ADMIN DASHBOARD (Day 8–14)

> 📺 Watch: [Flask CRUD Operations Hindi](https://www.youtube.com/watch?v=oA8brF3w5XQ) — Skip to Flask CRUD part

---

## 🧠 Big Picture: What is CRUD?

Every data-driven application revolves around four fundamental operations:

| Letter | Operation | SQL Command | HTTP Method | Example |
|--------|-----------|-------------|-------------|---------|
| **C**  | Create    | `INSERT`    | `POST`      | Add a new trek |
| **R**  | Read      | `SELECT`    | `GET`       | List all treks |
| **U**  | Update    | `UPDATE`    | `POST`      | Edit trek details |
| **D**  | Delete    | `DELETE`    | `GET/POST`  | Remove a trek |

The Admin dashboard is essentially a CRUD interface for **Treks**, **Users**, and **Staff**.

---

## 📖 Understanding the SQLAlchemy Query API

Before we write routes, you need to understand how to query the database with SQLAlchemy:

```python
# SELECT all treks
Trek.query.all()

# SELECT treks WHERE status = 'open'
Trek.query.filter_by(status='open').all()

# SELECT one trek by primary key, raise 404 if not found
Trek.query.get_or_404(trek_id)

# SELECT COUNT(*) — efficient, doesn't load all rows into memory
Trek.query.count()

# SELECT trek WHERE id=5
Trek.query.filter_by(id=5).first()

# SELECT trek WHERE name LIKE '%valley%'
Trek.query.filter(Trek.name.contains('valley')).all()

# Complex filter: WHERE name LIKE '%valley%' OR id=5
Trek.query.filter((Trek.name.contains('valley')) | (Trek.id == 5)).all()

# SQL JOIN: Get all Users who have a Staff profile
db.session.query(User).join(Staff).filter(Staff.approval_status == 'Approved').all()
```

### 📖 The SQLAlchemy Session (Unit of Work Pattern)

Think of `db.session` as a **shopping cart** for database operations:

```python
user = User(name='Raj')   # Create object in memory
db.session.add(user)      # Add to "cart"
db.session.add(trek)      # Add more to "cart"
db.session.commit()       # "Checkout" — apply all changes to DB atomically
```

**Atomicity**: If any operation fails during commit (e.g., constraint violation), ALL operations are rolled back. Either everything succeeds or nothing changes. This prevents partial/corrupted data.

To update, just modify the object — SQLAlchemy tracks changes:
```python
trek = Trek.query.get(1)
trek.name = "New Name"   # Modify in memory
db.session.commit()       # SQLAlchemy detects change, runs UPDATE automatically
```

To delete:
```python
trek = Trek.query.get(1)
db.session.delete(trek)
db.session.commit()
```

---

## 📖 How the Slot Reconciliation Formula Works

When an admin edits a trek and changes the total capacity (`max_slots`), we cannot simply reset `available_slot` to the new value — there may already be bookings that consumed some slots.

**Real-world analogy**: A bus has 40 seats. 10 passengers have already boarded. Admin upgrades the bus to 50 seats. How many available seats now? Answer: 50 - 10 = 40 (not 50!).

**Mathematical formula**:
```
new_available = current_available + (new_max - old_max)
```

Example:
- `old_max = 20`, `available_slot = 13` (so 7 people booked)
- Admin changes max to 25 (adds 5 seats)
- `new_available = 13 + (25 - 20) = 18` ✅ (still reflects 7 bookings)

We also use `max(0, ...)` to prevent going negative if admin drastically reduces capacity:
```python
trek.available_slot = max(0, trek.available_slot + (new_max - old_max))
```

---

## 📖 Understanding SQL JOINs

The `assign_staff` route needs to show only approved staff members. These people exist across **two tables**: `user` (their name, email) and `staff` (their approval_status).

A **JOIN** combines rows from two tables based on a matching condition:

```
user table:         staff table:
id | name           user_id | approval_status
1  | Raj            3       | Pending
2  | Admin          5       | Approved
3  | Priya
5  | Kiran
```

After `JOIN user ON staff.user_id = user.id` and `WHERE approval_status = 'Approved'`:
```
Result:
id | name  | approval_status
5  | Kiran | Approved
```

In SQLAlchemy:
```python
db.session.query(User).join(Staff).filter(
    User.role == 'staff',
    Staff.approval_status == 'Approved'
).all()
```

---

## Step 4.1 — Create `templates/admin/dashboard.html`

```html
<!-- templates/admin/dashboard.html -->
{% extends "base.html" %}
{% block title %}Admin Dashboard{% endblock %}
{% block content %}
<h2> Admin Dashboard</h2>

<!--
    Stats Cards: Show system-wide counts.
    The 'stats' dictionary is passed from the route:
    stats = {'total_treks': 5, 'total_users': 23, ...}
    
    {{ stats.total_treks }} → Python dict access in Jinja2
-->
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

<!-- Quick Navigation Links -->
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

<!-- Global Search Bar -->
<div class="mt-4">
    <form method="GET" action="/admin/search">
        <!--
            method="GET" for search: search queries belong in the URL.
            /admin/search?q=himalaya
            This makes search results bookmarkable and shareable.
        -->
        <div class="input-group">
            <input type="text" name="q" class="form-control"
                   placeholder="Search trek, user, or staff by name/ID...">
            <button class="btn btn-dark" type="submit">Search</button>
        </div>
    </form>
</div>
{% endblock %}
```

---

## Step 4.2 — Add Admin Routes to `app.py`

```python
# ───────────────────────────────────────────────────────────────────
# All of the following goes BELOW app = create_app() in app.py.
# Decorators first, then routes.
# ───────────────────────────────────────────────────────────────────

# ═══════════════════════════════════════════════════════
# ADMIN ROUTES
# All admin routes are protected by TWO decorators:
# @login_required → must be authenticated
# @admin_required → must have role='admin'
# ═══════════════════════════════════════════════════════

@app.route('/admin/dashboard')
@login_required
@admin_required
def admin_dashboard():
    """
    Dashboard home: shows aggregate statistics.
    
    .count() is much more efficient than:
        len(Trek.query.all())  ← loads all rows into memory first!
    .count() runs: SELECT COUNT(*) FROM trek
    """
    stats = {
        'total_treks'   : Trek.query.count(),
        'total_users'   : User.query.filter_by(role='user').count(),
        'total_staff'   : User.query.filter_by(role='staff').count(),
        'total_bookings': Booking.query.count()
    }
    return render_template('admin/dashboard.html', stats=stats)


# ── Trek CRUD ──────────────────────────────────────────

@app.route('/admin/treks')
@login_required
@admin_required
def admin_treks():
    """Read: List all treks."""
    treks = Trek.query.all()
    return render_template('admin/manage_treks.html', treks=treks)


@app.route('/admin/trek/add', methods=['GET', 'POST'])
@login_required
@admin_required
def add_trek():
    """
    Create: Add a new trek.
    
    KEY INSIGHT about max_slots vs available_slot:
    When creating a trek, BOTH start at the same value.
    Example: Admin creates trek with max_slots=30
             available_slot is also set to 30 (0 bookings yet)
    
    As users book: available_slot decreases.
    max_slots NEVER changes unless admin edits it.
    """
    if request.method == 'POST':
        # datetime is already imported at top of app.py
        max_slots = int(request.form['max_slots'])

        trek = Trek(
            name         = request.form['name'],
            location     = request.form['location'],
            difficulty   = request.form['difficulty'],
            stay_duration= int(request.form['stay_duration']),
            max_slots    = max_slots,
            available_slot = max_slots,  # Initially all slots are available
            start_date   = datetime.strptime(request.form['start_date'], '%Y-%m-%d').date(),
            end_date     = datetime.strptime(request.form['end_date'], '%Y-%m-%d').date(),
            description  = request.form.get('description', ''),
            status       = 'pending'  # Newly created treks start in pending state
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
    """
    Update: Edit an existing trek.
    
    SLOT RECONCILIATION:
    When admin changes max_slots, we must adjust available_slot proportionally.
    Formula: new_available = old_available + (new_max - old_max)
    
    max(0, ...) prevents negative available slots if admin reduces capacity
    below the number of existing bookings.
    """
    # get_or_404: fetch by PK, or automatically return HTTP 404 response
    trek = Trek.query.get_or_404(trek_id)

    if request.method == 'POST':
        # datetime is already imported at top of app.py
        trek.name         = request.form['name']
        trek.location     = request.form['location']
        trek.difficulty   = request.form['difficulty']
        trek.stay_duration= int(request.form['stay_duration'])
        trek.start_date   = datetime.strptime(request.form['start_date'], '%Y-%m-%d').date()
        trek.end_date     = datetime.strptime(request.form['end_date'], '%Y-%m-%d').date()
        trek.status       = request.form['status']
        trek.description  = request.form.get('description', '')

        # Slot reconciliation
        old_max = trek.max_slots
        new_max = int(request.form['max_slots'])
        trek.max_slots     = new_max
        trek.available_slot = max(0, trek.available_slot + (new_max - old_max))

        db.session.commit()  # SQLAlchemy auto-detects changes → runs UPDATE
        flash('Trek updated!', 'success')
        return redirect('/admin/treks')

    return render_template('admin/edit_trek.html', trek=trek)


@app.route('/admin/trek/delete/<int:trek_id>')
@login_required
@admin_required
def delete_trek(trek_id):
    """
    Delete: Remove a trek from the database.
    
    WARNING: This also cascade-deletes all associated bookings
    (if cascade is configured), or raises an IntegrityError if users
    have active bookings for this trek.
    """
    trek = Trek.query.get_or_404(trek_id)
    db.session.delete(trek)
    db.session.commit()
    flash('Trek deleted.', 'info')
    return redirect('/admin/treks')


# ── Staff Management ───────────────────────────────────

@app.route('/admin/staff')
@login_required
@admin_required
def admin_staff():
    """Read: List all users with role='staff'."""
    staff_list = User.query.filter_by(role='staff').all()
    return render_template('admin/manage_staff.html', staff_list=staff_list)


@app.route('/admin/staff/approve/<int:user_id>')
@login_required
@admin_required
def approve_staff(user_id):
    """
    Update: Set a staff member's approval_status to 'Approved'.
    
    After this, the staff member can log in (the login route checks this).
    Uses .first() — returns None if no profile (safer than get_or_404).
    """
    profile = Staff.query.filter_by(user_id=user_id).first()
    if profile:
        profile.approval_status = 'Approved'
        db.session.commit()
        flash('Staff approved!', 'success')
    return redirect('/admin/staff')


@app.route('/admin/staff/blacklist/<int:user_id>')
@login_required
@admin_required
def blacklist_staff(user_id):
    """
    Update: Mark a user as blacklisted.
    This is a soft ban — the record stays in DB but login is blocked.
    The login route checks user.is_blacklisted before allowing access.
    """
    user = User.query.get_or_404(user_id)
    user.is_blacklisted = True
    db.session.commit()
    flash(f'{user.name} has been blacklisted.', 'warning')
    return redirect('/admin/staff')


# ── Staff-Trek Assignment ──────────────────────────────

@app.route('/admin/trek/assign/<int:trek_id>', methods=['GET', 'POST'])
@login_required
@admin_required
def assign_staff(trek_id):
    """
    Assign an approved staff member to guide a trek.
    Also updates trek status to 'approved'.
    
    SQL JOIN explained:
    db.session.query(User)     → SELECT from user table
    .join(Staff)               → INNER JOIN staff ON user.id = staff.user_id
    .filter(...)               → WHERE role='staff' AND approval_status='Approved'
    
    Only approved staff show in the dropdown.
    """
    trek = Trek.query.get_or_404(trek_id)

    approved_staff = db.session.query(User).join(Staff).filter(
        User.role == 'staff',
        Staff.approval_status == 'Approved'
    ).all()

    if request.method == 'POST':
        staff_id = request.form.get('staff_id')
        trek.assigned_staff_id = int(staff_id) if staff_id else None
        trek.status = 'approved'
        db.session.commit()
        flash('Staff assigned and trek approved!', 'success')
        return redirect('/admin/treks')

    return render_template('admin/assign_staff.html', trek=trek, staff_list=approved_staff)


# ── User Management ────────────────────────────────────

@app.route('/admin/users')
@login_required
@admin_required
def admin_users():
    """Read: List all regular users (role='user')."""
    users = User.query.filter_by(role='user').all()
    return render_template('admin/manage_users.html', users=users)


@app.route('/admin/user/blacklist/<int:user_id>')
@login_required
@admin_required
def blacklist_user(user_id):
    """Blacklist a regular user — same logic as blacklisting staff."""
    user = User.query.get_or_404(user_id)
    user.is_blacklisted = True
    db.session.commit()
    flash(f'{user.name} has been blacklisted.', 'warning')
    return redirect('/admin/users')


# ── Global Search ──────────────────────────────────────

@app.route('/admin/search')
@login_required
@admin_required
def admin_search():
    """
    Multi-entity search across Treks, Users, and Staff.
    
    LOGIC BREAKDOWN:
    - request.args.get('q') reads from URL: /admin/search?q=himalaya
    - q.isdigit() → if the search term is a number, also try matching by ID
    - .contains(q) → SQL LIKE '%q%' (substring match, case-insensitive on SQLite)
    - The | operator → SQL OR
    - The & operator → SQL AND
    
    Example query for q='5':
        SELECT * FROM trek WHERE name LIKE '%5%' OR id = 5
    """
    q = request.args.get('q', '')

    if q.isdigit():
        treks = Trek.query.filter(
            (Trek.name.contains(q)) | (Trek.id == int(q))
        ).all()
        users = User.query.filter(
            (User.role == 'user') & ((User.name.contains(q)) | (User.id == int(q)))
        ).all()
        staff = User.query.filter(
            (User.role == 'staff') & ((User.name.contains(q)) | (User.id == int(q)))
        ).all()
    else:
        treks = Trek.query.filter(Trek.name.contains(q)).all()
        users = User.query.filter(User.name.contains(q), User.role == 'user').all()
        staff = User.query.filter(User.name.contains(q), User.role == 'staff').all()

    return render_template('admin/search_results.html', q=q, treks=treks, users=users, staff=staff)
```

---

## Step 4.3 — Create Trek Management Template

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
        <!--
            {% for trek in treks %}: Jinja2 loop over the list of Trek objects
            passed from the route via render_template(..., treks=treks)
        -->
        {% for trek in treks %}
        <tr>
            <td>{{ trek.id }}</td>
            <td>{{ trek.name }}</td>
            <td>{{ trek.location }}</td>
            <td>
                <!--
                    Conditional badge colors based on difficulty level.
                    bg-success=green, bg-warning=yellow, bg-danger=red
                -->
                <span class="badge 
                    {% if trek.difficulty == 'Easy' %}bg-success
                    {% elif trek.difficulty == 'Moderate' %}bg-warning text-dark
                    {% else %}bg-danger{% endif %}">
                    {{ trek.difficulty }}
                </span>
            </td>
            <!-- available_slot / max_slots: shows "13/20" format -->
            <td>{{ trek.available_slot }}/{{ trek.max_slots }}</td>
            <td><span class="badge bg-info">{{ trek.status }}</span></td>
            <td>
                <!--
                    trek.assigned_staff uses the SQLAlchemy relationship we defined:
                    assigned_treks = db.relationship(..., backref='assigned_staff')
                    So trek.assigned_staff gives the User object who is the guide.
                -->
                {% if trek.assigned_staff %}
                    {{ trek.assigned_staff.name }}
                {% else %}
                    <span class="text-muted">Unassigned</span>
                {% endif %}
            </td>
            <td>
                <a href="/admin/trek/edit/{{ trek.id }}" class="btn btn-sm btn-warning">Edit</a>
                <a href="/admin/trek/assign/{{ trek.id }}" class="btn btn-sm btn-info">Assign</a>
                <!--
                    onclick="return confirm(...)": Native browser dialog box.
                    If user clicks Cancel → returns false → the link doesn't navigate.
                    Prevents accidental deletions.
                -->
                <a href="/admin/trek/delete/{{ trek.id }}" class="btn btn-sm btn-danger"
                   onclick="return confirm('Delete this trek?')">Delete</a>
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endblock %}
```