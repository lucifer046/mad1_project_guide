## PHASE 9: OPTIONAL FEATURES (Day 26–32 — Do These for a Better Score)

---

## 9.1 — Flask-Login (Recommended)

### 📖 What is Flask-Login and Why Use It?

Right now, our auth system works — but it's manual. We wrote custom decorators (`login_required`), manually store `user_id` in session, and manually check it everywhere.

**Flask-Login** is an official Flask extension that handles all of this for you:
- Manages user sessions automatically
- Provides the `current_user` object anywhere in your app
- Handles the "remember me" cookie feature
- Provides `@login_required` decorator built-in
- Integrates with Flask's `unauthorized_handler`

### 📖 What is a Mixin?

`UserMixin` is a **mixin class** — a class that provides method implementations to be inherited by other classes.

```python
class UserMixin:
    # Flask-Login checks these properties on every request
    @property
    def is_active(self):      return True    # Is the account active?
    @property
    def is_authenticated(self): return True  # Is the user logged in?
    @property
    def is_anonymous(self):   return False   # Is this an anonymous visitor?
    def get_id(self):
        return str(self.id)                  # Returns user's primary key as string
```

By inheriting `UserMixin`, our `User` model gets all these properties automatically. Flask-Login reads them to manage sessions.

### 📖 The User Loader Callback

Flask-Login needs to know how to load a user from their stored ID. We provide a **callback function**:

```python
@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))
```

On **every request**, Flask-Login:
1. Reads the session cookie
2. Extracts the stored `user_id`
3. Calls `load_user(user_id)` to fetch the full User object
4. Sets `current_user` to that object

If the session is invalid or expired, `current_user` is set to an `AnonymousUser` object.

```bash
pip install flask-login
```

> ⚠️ **Important Before You Start:**
> Flask-Login is a **replacement** for the manual session/decorator system built in Phase 3.
> If you add Flask-Login, you must:
> 1. Remove or stop using the custom `login_required` decorator (replace it with Flask-Login's built-in one)
> 2. Use `login_user(user)` instead of manually setting `session['user_id'] = user.id`
> 3. Use `logout_user()` instead of `session.clear()`
> 4. Use `current_user.id` instead of `session['user_id']`
>
> **Your project works perfectly WITHOUT this.** Only add it if you want the bonus marks.

```python
# In app.py — add Flask-Login integration

from flask_login import (
    LoginManager,
    UserMixin,
    login_user,
    logout_user,
    current_user,
    login_required   # Flask-Login's built-in decorator (replaces our custom one)
)

# Initialize the LoginManager
login_manager = LoginManager()
login_manager.login_view = 'login'   # Which route to redirect to if not logged in
login_manager.init_app(app)

# Tell Flask-Login HOW to load a user from their stored ID
@login_manager.user_loader
def load_user(user_id):
    """
    Called automatically on every request.
    user_id comes from the session cookie (stored as string).
    We convert to int for the database query.
    """
    return User.query.get(int(user_id))

# In models.py — make User inherit UserMixin
from flask_login import UserMixin

class User(db.Model, UserMixin):
    # UserMixin provides: is_authenticated, is_anonymous, get_id()
    # IMPORTANT: UserMixin's is_active always returns True.
    # We must override it to respect our is_blacklisted column:
    __tablename__ = 'user'

    # ... all existing columns stay the same ...

    @property
    def is_active(self):
        # User is active only if not blacklisted
        return not self.is_blacklisted
```

---

## 9.2 — Add Charts with Chart.js (Optional Visualization)

### 📖 What is Chart.js?

Chart.js is a JavaScript library that draws beautiful, interactive charts in the browser using the HTML `<canvas>` element.

**How data flows from Flask to Chart.js:**

```
Python (Flask route)          Jinja2 Template             JavaScript (Chart.js)
trek_names = ['Trek A', ...]  {{ trek_names|tojson }}     ["Trek A", ...]
booking_counts = [12, 7, ...]  {{ booking_counts|tojson }} [12, 7, ...]
```

The `|tojson` Jinja2 filter converts a Python list into a JSON array string that JavaScript can directly understand.

### In your admin dashboard route, add:

```python
@app.route('/admin/dashboard')
@login_required
@admin_required
def admin_dashboard():
    stats = { ... }  # your existing stats code

    # Gather chart data: names and booking counts per trek
    all_treks     = Trek.query.all()
    trek_names    = [t.name for t in all_treks]
    booking_counts= [t.get_booked_slots() for t in all_treks]

    return render_template(
        'admin/dashboard.html',
        stats=stats,
        trek_names=trek_names,
        booking_counts=booking_counts
    )
```

### In `templates/admin/dashboard.html`, add at the bottom of `{% block content %}`:

```html
<!--
    <canvas>: An HTML element that JavaScript draws graphics onto.
    Chart.js targets this element by its id.
-->
<h4 class="mt-5">Bookings per Trek</h4>
<canvas id="bookingChart" width="400" height="200"></canvas>

<!-- Load Chart.js from CDN -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<script>
/*
    {{ trek_names | tojson }}:
    Jinja2 renders this as a JSON array: ["Himalaya Trek", "Valley Trek", ...]
    JavaScript reads it directly as an array.
*/
const ctx = document.getElementById('bookingChart').getContext('2d');
new Chart(ctx, {
    type: 'bar',
    data: {
        labels  : {{ trek_names    | tojson }},
        datasets: [{
            label          : 'Bookings per Trek',
            data           : {{ booking_counts | tojson }},
            backgroundColor: 'rgba(54, 162, 235, 0.6)',
            borderColor    : 'rgba(54, 162, 235, 1)',
            borderWidth    : 1
        }]
    },
    options: {
        responsive: true,
        scales: {
            y: { beginAtZero: true }
        }
    }
});
</script>
```

---

## 9.3 — Add API Endpoints (JSON Serialization)

### 📖 What is a REST API?

So far, our routes return **HTML** (complete web pages for browsers). But what if a mobile app or another service wants the same data? They don't need full HTML — they just need the raw data.

**REST API** (Representational State Transfer) routes return **JSON** (JavaScript Object Notation) instead of HTML:

```json
[
    {"id": 1, "name": "Himalaya Trek", "location": "Uttarakhand", "available_slot": 13},
    {"id": 2, "name": "Valley Route", "location": "Kashmir", "available_slot": 5}
]
```

Any application — React, Vue, Android, iOS, or another server — can consume this.

### 📖 What is `jsonify()`?

`jsonify()` is a Flask function that:
1. Takes a Python dict or list
2. Serializes it to a JSON-formatted string
3. Returns an HTTP Response with `Content-Type: application/json` header

The `Content-Type` header tells the client: *"Parse this response body as JSON, not HTML."*

Without `Content-Type: application/json`, a browser would try to display the JSON as plain text.

```python
# ── Optional JSON REST API Endpoints ──

@app.route('/api/treks')
def api_treks():
    """
    Public API endpoint: returns all open treks as JSON.
    No authentication required — public information.
    
    Process:
    1. Query open treks from DB
    2. Convert each Trek object to a Python dict manually
       (SQLAlchemy objects can't be directly JSON-serialized)
    3. jsonify() converts the list of dicts to a JSON response
    """
    from flask import jsonify

    treks = Trek.query.filter_by(status='open').all()

    # List comprehension: build a list of dicts from Trek objects
    result = [
        {
            'id'            : t.id,
            'name'          : t.name,
            'location'      : t.location,
            'difficulty'    : t.difficulty,
            'available_slot': t.available_slot,
            'stay_duration' : t.stay_duration,
            'start_date'    : str(t.start_date),  # date objects need str() for JSON
            'end_date'      : str(t.end_date)
        }
        for t in treks
    ]

    return jsonify(result)


@app.route('/api/treks/<int:trek_id>')
def api_trek_detail(trek_id):
    """
    Returns details for a single trek.
    
    Access: /api/treks/5 → returns JSON for trek with id=5
    If trek doesn't exist → get_or_404 returns HTTP 404 automatically.
    """
    from flask import jsonify

    trek = Trek.query.get_or_404(trek_id)

    return jsonify({
        'id'            : trek.id,
        'name'          : trek.name,
        'location'      : trek.location,
        'difficulty'    : trek.difficulty,
        'available_slot': trek.available_slot,
        'max_slots'     : trek.max_slots,
        'stay_duration' : trek.stay_duration,
        'start_date'    : str(trek.start_date),
        'end_date'      : str(trek.end_date),
        'status'        : trek.status,
        'description'   : trek.description
    })
```

### Testing Your API

You can test API endpoints directly in your browser or with `curl`:

```bash
# In a new terminal while app is running:
curl http://127.0.0.1:5000/api/treks
# Returns JSON array of open treks

curl http://127.0.0.1:5000/api/treks/1
# Returns JSON for trek with id=1
```

Or open `http://127.0.0.1:5000/api/treks` in your browser — it will display the raw JSON.