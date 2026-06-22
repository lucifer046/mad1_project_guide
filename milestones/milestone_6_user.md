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