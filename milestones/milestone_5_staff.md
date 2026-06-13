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