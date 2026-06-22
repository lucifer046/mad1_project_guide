## PHASE 6: USER DASHBOARD & BOOKINGS (Day 18–24)

---

## 🧠 Big Picture: The User's Journey

A regular user (trekker) interacts with the app in a clear flow:

```
Register → Login → Browse Open Treks → Search/Filter → Book a Trek → View History → Cancel if needed
```

This phase implements all of these steps. The most important concept here is **transactional integrity** — when a user books a trek, the slot count must decrease by exactly 1, always.

---

## 📖 Defense-in-Depth: Multi-Layer Validation

Security and data integrity are never achieved by a single check. We use **defense-in-depth** — multiple validation layers, each acting as a safety net for the ones above.

**For booking a trek:**

```
Layer 1 (UI/Frontend):
    → "Book Now" button hidden/disabled if already booked or trek is full
    → Purely cosmetic — a savvy user can bypass this with direct HTTP requests

Layer 2 (Server/Application Logic):
    → Python code re-validates: status == 'open', available_slot > 0, not already booked
    → This runs even if the user bypassed Layer 1

Layer 3 (Database/Storage):
    → UniqueConstraint('user_id', 'trek_id') prevents duplicate rows at DB level
    → Even if Layers 1 and 2 somehow fail, the database itself refuses the duplicate
```

**Why multiple layers?** Because each layer can fail independently:
- Frontend-only: easily bypassed with tools like curl or Postman
- Backend-only: if a bug exists in code, DB has no protection
- Database-only: raises uncaught exceptions that crash the app

All three together create a robust, professional system.

---

## 📖 Understanding Race Conditions in Slot Booking

A **race condition** happens when two operations happen simultaneously and their results interfere with each other.

**Scenario:**
- Trek has `available_slot = 1` (last seat)
- User A and User B click "Book Now" at the exact same millisecond
- Both their requests arrive at the server simultaneously

```
Thread A: reads available_slot = 1 → passes check → prepares to book
Thread B: reads available_slot = 1 → passes check → prepares to book
Thread A: decrements → available_slot = 0 → commits
Thread B: decrements → available_slot = -1 → commits ← OVERBOOKING!
```

**Our protection:** The database-level `available_slot -= 1` combined with the UniqueConstraint creates a safeguard. The UniqueConstraint will reject the second booking attempt at the DB level, causing an error that we catch.

For a truly race-condition-proof solution, you'd use database transactions with `SELECT FOR UPDATE` (row-level locking), but for a learning project, our current approach is sufficient.

---

## 📖 Passing Functions to Templates

In the browse treks route, we need a way for the template to check if the current user has already booked each trek. We do this by **passing a function itself** as a template variable:

```python
def already_booked(trek_id):
    existing = Booking.query.filter_by(user_id=user_id, trek_id=trek_id, status='Booked').first()
    return existing is not None

return render_template('user/browse_treks.html', treks=treks, already_booked=already_booked)
```

Then in the template:
```html
{% if already_booked(trek.id) %}   ← Calling the Python function from Jinja2!
```

Jinja2 allows passing callable functions as template variables. This is a clean way to do per-item logic in templates without making dozens of database calls in the route.

---

## 📖 Slot Management During Booking/Cancellation

Booking lifecycle:
```
Trek created: available_slot = max_slots (e.g., 20)
User A books:  available_slot = 19  (decrement by 1)
User B books:  available_slot = 18  (decrement by 1)
User A cancels: available_slot = 19 (increment by 1, slot released)
```

This is maintained **at the application level** (Python code updates the column). The `available_slot` column is like a real-time counter of free seats.

**Why check `booking.status == 'Booked'` before cancelling?**

A booking can only be cancelled if it's currently active. Attempting to cancel an already-cancelled booking should be rejected, otherwise the slot count becomes corrupted:

```
booking.status = 'Cancelled'   # User already cancelled
Clicks cancel again:
  booking.available_slot += 1  # WRONG! Slot count goes up incorrectly
```

Our guard `if booking.status == 'Booked':` prevents this.

---

## Step 6.1 — Create `templates/user/dashboard.html`

```html
<!-- templates/user/dashboard.html -->
{% extends "base.html" %}
{% block title %}User Dashboard{% endblock %}
{% block content %}
<h2>Welcome, {{ session.user_name }}! </h2>

<div class="row mt-4">
    <div class="col-md-4">
        <div class="card p-3 mb-3 text-center bg-light">
            <h4> Find Treks</h4>
            <p>Explore available trekking routes and book your next adventure.</p>
            <a href="/user/treks" class="btn btn-primary">Browse Treks</a>
        </div>
    </div>
    <div class="col-md-4">
        <div class="card p-3 mb-3 text-center bg-light">
            <h4> My Bookings</h4>
            <p>View your current registrations and past trekking history.</p>
            <a href="/user/bookings" class="btn btn-success">View History</a>
        </div>
    </div>
    <div class="col-md-4">
        <div class="card p-3 mb-3 text-center bg-light">
            <h4> My Profile</h4>
            <p>Update your name and contact details.</p>
            <a href="/user/profile" class="btn btn-warning">Edit Profile</a>
        </div>
    </div>
</div>
{% endblock %}
```

---

## Step 6.2 — Create `templates/user/browse_treks.html`

```html
<!-- templates/user/browse_treks.html -->
{% extends "base.html" %}
{% block title %}Browse Treks{% endblock %}
{% block content %}
<h3> Available Treks</h3>

<!--
    Search Filter Form:
    method="GET" because search is idempotent (same search = same results).
    GET parameters appear in URL: /user/treks?q=himalaya&difficulty=Hard
    This makes searches bookmarkable and shareable.
    
    request.args.get('q','') reads the current URL parameter
    to pre-fill the search box with the user's last query.
-->
<form method="GET" action="/user/treks" class="row g-2 mb-4">
    <div class="col-md-6">
        <input type="text" name="q" value="{{ request.args.get('q','') }}"
               class="form-control" placeholder="Search by name or location...">
    </div>
    <div class="col-md-4">
        <select name="difficulty" class="form-select">
            <option value="">All Difficulties</option>
            <option value="Easy"     {{ 'selected' if request.args.get('difficulty') == 'Easy' }}>Easy</option>
            <option value="Moderate" {{ 'selected' if request.args.get('difficulty') == 'Moderate' }}>Moderate</option>
            <option value="Hard"     {{ 'selected' if request.args.get('difficulty') == 'Hard' }}>Hard</option>
        </select>
    </div>
    <div class="col-md-2">
        <button type="submit" class="btn btn-dark w-100">Filter</button>
    </div>
</form>

<!-- Trek Cards Grid -->
<div class="row">
    {% for trek in treks %}
    <div class="col-md-4 mb-4">
        <div class="card h-100 shadow-sm">
            <div class="card-body">
                <h5 class="card-title">{{ trek.name }}</h5>
                <h6 class="text-muted">📍 {{ trek.location }}</h6>
                <p class="card-text">{{ trek.description[:100] }}...</p>
                <p class="mb-1">
                    <!-- Difficulty badge with conditional color -->
                    <span class="badge
                        {% if trek.difficulty == 'Easy' %}bg-success
                        {% elif trek.difficulty == 'Moderate' %}bg-warning text-dark
                        {% else %}bg-danger{% endif %}">
                        {{ trek.difficulty }}
                    </span>
                    <span class="ms-2">🕐 {{ trek.stay_duration }} days</span>
                </p>
                <p>🪑 {{ trek.available_slot }} slots left</p>
                <p>📅 {{ trek.start_date }} → {{ trek.end_date }}</p>
            </div>
            <div class="card-footer">
                <!--
                    Priority order of button states:
                    1. Already booked → disabled "Already Booked"
                    2. No slots left → disabled "Fully Booked"
                    3. Available → active "Book Now" link
                    
                    already_booked() is a Python function we passed to the template.
                    It queries the DB for this user + this specific trek.
                -->
                {% if already_booked(trek.id) %}
                    <span class="btn btn-secondary w-100 disabled">Already Booked</span>
                {% elif trek.available_slot <= 0 %}
                    <span class="btn btn-secondary w-100 disabled">Fully Booked</span>
                {% else %}
                    <a href="/user/book/{{ trek.id }}" class="btn btn-success w-100">Book Now</a>
                {% endif %}
            </div>
        </div>
    </div>
    {% else %}
    <!-- {% else %} on a for loop: runs if the list is empty -->
    <div class="col-12">
        <div class="alert alert-warning">No treks match your search. <a href="/user/treks">Clear filters</a></div>
    </div>
    {% endfor %}
</div>
{% endblock %}
```

---

## Step 6.3 — Add User Routes to `app.py`

```python
# ═══════════════════════════════════════════════════════
# USER ROUTES
# ═══════════════════════════════════════════════════════

@app.route('/user/dashboard')
@login_required
def user_dashboard():
    return render_template('user/dashboard.html')


@app.route('/user/treks')
@login_required
def browse_treks():
    """
    Display all open treks with optional search filtering.
    
    QUERY BUILDING PATTERN:
    Instead of multiple separate queries, we build one query progressively.
    Start with the base filter, then ADD conditions based on what the user submitted.
    
    This is called the "Builder Pattern" for queries.
    """
    q          = request.args.get('q', '')
    difficulty = request.args.get('difficulty', '')

    # Base query: only 'open' treks visible to users
    query = Trek.query.filter_by(status='open')

    # Conditionally add more filters if user provided them
    if q:
        query = query.filter(
            (Trek.name.contains(q)) | (Trek.location.contains(q))
        )
    if difficulty:
        query = query.filter_by(difficulty=difficulty)

    treks = query.all()

    # ── Template Helper Function ──
    # We define this INSIDE the route so it has access to user_id from session.
    # Closures in Python: inner functions can "capture" variables from outer scope.
    user_id = session['user_id']

    def already_booked(trek_id):
        """Check if current user has an active booking for this specific trek."""
        booking = Booking.query.filter_by(
            user_id=user_id,
            trek_id=trek_id,
            status='Booked'
        ).first()
        return booking is not None

    # Pass the FUNCTION ITSELF (not its result) as a template variable
    return render_template('user/browse_treks.html', treks=treks, already_booked=already_booked)


@app.route('/user/book/<int:trek_id>')
@login_required
def book_trek(trek_id):
    """
    Book a trek for the logged-in user.
    
    COMPLETE VALIDATION CHAIN:
    1. Trek status must be 'open' (server-side, not just UI)
    2. Slots must be available (server-side check)
    3. User must not have already booked this trek
    4. Create booking + decrement slot count ATOMICALLY
    """
    trek    = Trek.query.get_or_404(trek_id)
    user_id = session['user_id']

    # Validation 1: Status check
    if trek.status != 'open':
        flash('This trek is not open for booking.', 'danger')
        return redirect('/user/treks')

    # Validation 2: Capacity check
    if trek.available_slot <= 0:
        flash('No slots available for this trek.', 'danger')
        return redirect('/user/treks')

    # Validation 3: Duplicate booking check
    existing = Booking.query.filter_by(
        user_id=user_id,
        trek_id=trek_id,
        status='Booked'
    ).first()
    if existing:
        flash('You have already booked this trek.', 'warning')
        return redirect('/user/treks')

    # All checks passed — create the booking atomically
    booking = Booking(user_id=user_id, trek_id=trek_id, status='Booked')
    trek.available_slot -= 1   # Decrement slot count

    db.session.add(booking)    # Stage the new booking
    db.session.commit()        # Commit both: booking INSERT + trek UPDATE together

    flash(f'Successfully booked {trek.name}!', 'success')
    return redirect('/user/bookings')


@app.route('/user/cancel_booking/<int:booking_id>')
@login_required
def cancel_booking(booking_id):
    """
    Cancel an active booking.
    
    SECURITY: Users can only cancel their OWN bookings.
    Without this check, User A could cancel User B's booking by guessing the ID.
    This is another horizontal privilege (IDOR) protection.
    
    STATE VALIDATION: Only 'Booked' bookings can be cancelled.
    Prevents slot corruption from double-cancellation.
    """
    booking = Booking.query.get_or_404(booking_id)

    # Security: Ownership check
    if booking.user_id != session['user_id']:
        flash('Unauthorized.', 'danger')
        return redirect('/user/bookings')

    # State check: Only cancel if currently active
    if booking.status == 'Booked':
        booking.status = 'Cancelled'
        booking.trek.available_slot += 1   # Release slot back to the pool
        db.session.commit()
        flash('Booking cancelled.', 'info')
    else:
        flash('This booking cannot be cancelled.', 'warning')

    return redirect('/user/bookings')


@app.route('/user/profile', methods=['GET', 'POST'])
@login_required
def user_profile():
    """
    View and update user profile.
    
    After updating the name, we also update session['user_name']
    so the navbar shows the new name immediately without logout/login.
    """
    user = User.query.get_or_404(session['user_id'])

    if request.method == 'POST':
        user.name = request.form['name']
        db.session.commit()
        session['user_name'] = user.name   # Sync session with DB change
        flash('Profile updated!', 'success')
        return redirect('/user/dashboard')

    return render_template('user/profile.html', user=user)
```