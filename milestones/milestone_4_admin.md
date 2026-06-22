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
    if q.isdigit():
        treks = Trek.query.filter((Trek.name.contains(q)) | (Trek.id == int(q))).all()
        users = User.query.filter((User.role == 'user') & ((User.name.contains(q)) | (User.id == int(q)))).all()
        staff = User.query.filter((User.role == 'staff') & ((User.name.contains(q)) | (User.id == int(q)))).all()
    else:
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