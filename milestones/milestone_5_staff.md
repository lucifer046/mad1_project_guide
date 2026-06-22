## PHASE 5: TREK STAFF DASHBOARD (Day 14–18)

---

## 🧠 Big Picture: What Can Staff Do?

A trek staff member (guide) has a limited, focused dashboard. They can:
1. See only the treks **assigned to them** (not all treks)
2. Update **slot availability** and **trek status** for their trek
3. View the **list of registered trekkers** for their trek

They **cannot**: see other staff's treks, manage users, create new treks, or access admin features.

This is **Role-Based Access Control (RBAC)** in action: different roles get different permissions.

---

## 📖 Understanding Horizontal Privilege Escalation (IDOR Attack)

This is a critical security concept every web developer must know.

**The Problem:**

Imagine staff member "Priya" is assigned to Trek #5. She knows the URL for updating her trek:
```
POST /staff/update_trek/5
```

But what if she changes the URL to:
```
POST /staff/update_trek/7
```

Trek #7 belongs to another staff member "Kiran". Without a security check, Priya could modify Kiran's trek — even though she's not assigned to it!

This attack is called **IDOR — Insecure Direct Object Reference**.

**The simple fix:** After fetching the trek, verify ownership:
```python
if trek.assigned_staff_id != session['user_id']:
    flash('You are not assigned to this trek.', 'danger')
    return redirect('/staff/dashboard')
```

This is called a **horizontal privilege check** because:
- "Vertical" privilege = admin vs. user (different levels)
- "Horizontal" privilege = User A vs. User B (same level, different data)

The decorator `@staff_required` only does vertical checking (are they staff?). The ownership check does horizontal checking (is THIS their trek?).

---

## 📖 Jinja2 Filters for Counting Relationships

In the dashboard, we show a count of registered trekkers for each trek:

```html
{{ trek.bookings|selectattr('status', 'eq', 'Booked')|list|length }}
```

This is a **Jinja2 filter pipeline** — each `|` passes the result to the next operation:

```
trek.bookings              → All booking objects for this trek (list)
|selectattr('status', 'eq', 'Booked')  → Filter: keep only where booking.status == 'Booked'
|list                      → Convert the filter result to an actual list
|length                    → Count the items
```

Without this, you'd need a Python helper function or a separate database query.

---

## 📖 The `capitalize` Filter in Jinja2

We store statuses in lowercase (`'open'`, `'closed'`), but we want to display them capitalized to the user.

The Jinja2 `|capitalize` filter does exactly this:
```
{{ 'open' | capitalize }}   → "Open"
{{ 'completed' | capitalize }} → "Completed"
```

This keeps storage and display concerns separated — storing lowercase makes comparisons simpler in Python code (`trek.status == 'open'` is unambiguous).

---

## Step 5.1 — Create `templates/staff/dashboard.html`

```html
<!-- templates/staff/dashboard.html -->
{% extends "base.html" %}
{% block title %}Staff Dashboard{% endblock %}
{% block content %}
<h2> Staff Dashboard — Welcome, {{ session.user_name }}</h2>

<!--
    assigned_treks: a list of Trek objects passed from the route.
    Each trek in this list has assigned_staff_id = session['user_id'].
-->
{% if assigned_treks %}
    {% for trek in assigned_treks %}
    <div class="card mb-3">
        <div class="card-header d-flex justify-content-between">
            <strong>{{ trek.name }}</strong>
            <!--
                Ternary expression in Jinja2:
                'success' if condition else 'secondary'
                → bg-success (green) for open, bg-secondary (grey) for all others
            -->
            <span class="badge bg-{{ 'success' if trek.status == 'open' else 'secondary' }}">
                {{ trek.status | capitalize }}
            </span>
        </div>
        <div class="card-body">
            <!-- Quick Trek Info Summary -->
            <p>
                📍 {{ trek.location }} &nbsp;|&nbsp;
                🕐 {{ trek.stay_duration }} days &nbsp;|&nbsp;
                🪑 {{ trek.available_slot }}/{{ trek.max_slots }} slots left
            </p>

            <!--
                Trek Update Form:
                Staff can update:
                  1. available_slot (how many seats are free right now)
                  2. status (open/closed/ongoing/completed)
                
                action="/staff/update_trek/{{ trek.id }}"
                → Each trek has its own submit URL with its ID embedded.
            -->
            <form method="POST" action="/staff/update_trek/{{ trek.id }}" class="row g-2">
                <div class="col-md-3">
                    <label class="form-label">Available Slots</label>
                    <!--
                        value="{{ trek.available_slot }}": Pre-fills the input
                        with current value so staff sees what it is before editing.
                        min="0": Browser validation — can't enter negative numbers.
                    -->
                    <input type="number" name="available_slot" class="form-control"
                           value="{{ trek.available_slot }}" min="0">
                </div>
                <div class="col-md-3">
                    <label class="form-label">Status</label>
                    <select name="status" class="form-select">
                        <!--
                            Loop over valid status options.
                            'selected' is added to the option that matches current trek status
                            so the dropdown shows the current state.
                        -->
                        {% for s in ['open', 'closed', 'ongoing', 'completed'] %}
                            <option value="{{ s }}" {{ 'selected' if trek.status == s }}>
                                {{ s | capitalize }}
                            </option>
                        {% endfor %}
                    </select>
                </div>
                <div class="col-md-2 d-flex align-items-end">
                    <button type="submit" class="btn btn-primary">Update</button>
                </div>
            </form>

            <!-- Trekker List for this Trek -->
            <h6 class="mt-3">
                Registered Trekkers
                <!--
                    Jinja2 filter pipeline to count active bookings:
                    1. trek.bookings → all booking objects (lazy-loaded from DB)
                    2. selectattr('status', 'eq', 'Booked') → filter active bookings
                    3. list → convert to list
                    4. length → count
                -->
                ({{ trek.bookings|selectattr('status', 'eq', 'Booked')|list|length }}):
            </h6>
            <ul class="list-group">
                {% for booking in trek.bookings %}
                    {% if booking.status == 'Booked' %}
                    <li class="list-group-item d-flex justify-content-between">
                        <!--
                            booking.trekker: backref relationship.
                            In models.py: db.relationship('Booking', backref='trekker')
                            So booking.trekker gives us the User who made this booking.
                        -->
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

---

## Step 5.2 — Add Staff Routes to `app.py`

```python
# ═══════════════════════════════════════════════════════
# STAFF ROUTES
# Protected by @login_required AND @staff_required.
# Additionally uses manual ownership check for security.
# ═══════════════════════════════════════════════════════

@app.route('/staff/dashboard')
@login_required
@staff_required
def staff_dashboard():
    """
    Show the staff member's assigned treks.
    
    KEY: We use session['user_id'] to know WHO is logged in.
    Then filter treks WHERE assigned_staff_id = that user's ID.
    
    A staff member with no assigned treks sees a friendly "nothing assigned" message.
    """
    user_id = session['user_id']
    assigned_treks = Trek.query.filter_by(assigned_staff_id=user_id).all()
    return render_template('staff/dashboard.html', assigned_treks=assigned_treks)


@app.route('/staff/update_trek/<int:trek_id>', methods=['POST'])
@login_required
@staff_required
def staff_update_trek(trek_id):
    """
    Update trek's available slots and status.
    
    SECURITY LAYERS:
    Layer 1: @login_required → must be logged in (session exists)
    Layer 2: @staff_required → must have role='staff'
    Layer 3: Ownership check → must be ASSIGNED to THIS specific trek
    
    All three must pass. Without Layer 3, any staff member could edit any trek.
    This is the IDOR protection.
    """
    trek = Trek.query.get_or_404(trek_id)

    # ── IDOR Protection: Horizontal Privilege Check ──
    if trek.assigned_staff_id != session['user_id']:
        flash('You are not assigned to this trek.', 'danger')
        return redirect('/staff/dashboard')

    # Update the trek fields with submitted form data
    trek.available_slot = int(request.form['available_slot'])
    trek.status         = request.form['status']

    db.session.commit()
    flash('Trek updated successfully!', 'success')
    return redirect('/staff/dashboard')
```