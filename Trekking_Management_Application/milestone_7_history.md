## PHASE 7: BOOKING HISTORY & STATUS TRACKING (Day 24–26)

---

## 🧠 Big Picture: Why Does Booking History Matter?

Users need to be able to:
1. **See** what they've booked (current and past)
2. **Track** the status of their treks (is it still open? completed? cancelled?)
3. **Cancel** active bookings directly from the history view

This page also demonstrates one of SQLAlchemy's most powerful features: **navigating relationships** to fetch data across multiple tables without writing complex SQL JOINs yourself.

---

## 📖 Understanding Lazy Loading — What Happens When You Access `b.trek.name`?

When the route runs:
```python
bookings = Booking.query.filter_by(user_id=user_id).all()
```

SQLAlchemy fetches all `Booking` rows. At this point, it does **NOT** fetch the associated `Trek` or `User` data. Those tables haven't been touched yet.

But when the template accesses `b.trek.name`:
```html
<td>{{ b.trek.name }}</td>
```

Jinja2 evaluates `b.trek`, which triggers SQLAlchemy to run an additional query:
```sql
SELECT * FROM trek WHERE id = <b.trek_id>
```

This is **lazy loading** — the data is fetched "lazily" (only when needed, on demand).

**The N+1 Query Problem:**

If you have 50 bookings, each accessing `b.trek.name` could trigger 50 separate queries:
- 1 query to get all bookings
- 50 queries to get each trek (one per booking)
= 51 total queries!

For a learning project with few records, this is fine. For production apps with thousands of rows, you'd use **eager loading** with `.options(joinedload(...))` to fetch everything in a single JOIN query.

---

## 📖 Understanding `strftime` — Formatting Dates for Humans

Dates stored in databases look like: `2024-03-15 09:23:41.503921`

Users don't want to see that. They want: `15 Mar 2024`

Python's `datetime.strftime()` formats datetime objects into human-readable strings:

| Code | Meaning | Example |
|------|---------|---------|
| `%d` | Day of month (zero-padded) | `05`, `15`, `31` |
| `%b` | Abbreviated month name | `Jan`, `Feb`, `Mar` |
| `%B` | Full month name | `January`, `February` |
| `%Y` | Full year | `2024` |
| `%H` | Hour (24-hour) | `09`, `14`, `23` |
| `%M` | Minutes | `05`, `45` |

So `b.booking_date.strftime('%d %b %Y')` converts:
```
2024-03-15 09:23:41  →  "15 Mar 2024"
```

In Jinja2, you call Python methods directly: `{{ b.booking_date.strftime('%d %b %Y') }}`

---

## 📖 Using `.order_by()` for Sorted Results

Users want to see their **most recent** bookings first. The `.order_by(Booking.booking_date.desc())` clause sorts results in **descending** order (newest first):

```python
# Newest booking appears first
Booking.query.filter_by(user_id=user_id).order_by(Booking.booking_date.desc()).all()

# Oldest booking appears first (ascending = default)
Booking.query.filter_by(user_id=user_id).order_by(Booking.booking_date.asc()).all()
```

This translates to SQL:
```sql
SELECT * FROM bookings WHERE user_id = 2 ORDER BY booking_date DESC
```

---

## Step 7.1 — Create `templates/user/booking_history.html`

```html
<!-- templates/user/booking_history.html -->
{% extends "base.html" %}
{% block title %}My Trek History{% endblock %}
{% block content %}
<h3> My Trekking History</h3>

{% if bookings %}
<table class="table table-hover">
    <thead class="table-dark">
        <tr>
            <th>#</th>
            <th>Trek Name</th>
            <th>Location</th>
            <th>Difficulty</th>
            <th>Date Booked</th>
            <th>Trek Status</th>
            <th>Booking Status</th>
            <th>Action</th>
        </tr>
    </thead>
    <tbody>
        {% for b in bookings %}
        <tr>
            <td>{{ b.id }}</td>

            <!--
                RELATIONSHIP TRAVERSAL:
                b is a Booking object. It has trek_id = some number.
                b.trek → SQLAlchemy looks up that trek_id in Trek table → returns Trek object
                b.trek.name → access the name attribute of that Trek object
                
                This is powered by the backref='trek' in models.py:
                    bookings = db.relationship('Booking', backref='trek', lazy=True)
                
                Without this relationship, you'd need to manually query:
                    Trek.query.get(b.trek_id).name
            -->
            <td>{{ b.trek.name }}</td>
            <td>{{ b.trek.location }}</td>
            <td>{{ b.trek.difficulty }}</td>

            <!--
                strftime: Format the datetime object into a readable string.
                b.booking_date is a Python datetime object from the DB.
                .strftime('%d %b %Y') → "15 Mar 2024"
            -->
            <td>{{ b.booking_date.strftime('%d %b %Y') }}</td>
            <td>{{ b.trek.status | capitalize }}</td>

            <td>
                <!--
                    Conditional badge styling based on booking status:
                    - Booked → green (active)
                    - Cancelled → grey (inactive)
                    - Completed → blue (done)
                -->
                <span class="badge 
                    {% if b.status == 'Booked' %}bg-success
                    {% elif b.status == 'Cancelled' %}bg-secondary
                    {% else %}bg-primary{% endif %}">
                    {{ b.status }}
                </span>
            </td>
            <td>
                <!--
                    Only show Cancel button if booking is active.
                    No button for already-cancelled or completed bookings.
                    
                    onclick="return confirm(...)": Shows a browser dialog.
                    If user clicks OK → returns true → link navigates.
                    If user clicks Cancel → returns false → link does nothing.
                -->
                {% if b.status == 'Booked' %}
                    <a href="/user/cancel_booking/{{ b.id }}"
                       class="btn btn-sm btn-outline-danger"
                       onclick="return confirm('Cancel this booking?')">
                        Cancel
                    </a>
                {% endif %}
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>

{% else %}
    <!--
        {% if bookings %} ... {% else %}: What to show when the list is empty.
        Good UX: don't show a blank page, guide the user to take action.
    -->
    <div class="alert alert-info">
        No treks booked yet.
        <a href="/user/treks">Explore available treks!</a>
    </div>
{% endif %}
{% endblock %}
```

---

## Step 7.2 — Add Booking History Route to `app.py`

```python
# ═══════════════════════════════════════════════════════
# BOOKING HISTORY ROUTE
# ═══════════════════════════════════════════════════════

@app.route('/user/bookings')
@login_required
def user_bookings():
    """
    Fetch and display all bookings for the current user.
    
    QUERY EXPLAINED:
    Booking.query              → FROM bookings
    .filter_by(user_id=...)    → WHERE user_id = <session user>
    .order_by(...desc())       → ORDER BY booking_date DESC (newest first)
    .all()                     → fetch all matching rows as Booking objects
    
    The template then traverses relationships:
    b.trek → lazy-loads the Trek record for each booking
    b.trek.name, b.trek.location, etc.
    b.trekker → lazy-loads the User record (not needed here, but available)
    """
    user_id  = session['user_id']
    bookings = Booking.query.filter_by(user_id=user_id).order_by(
        Booking.booking_date.desc()
    ).all()

    return render_template('user/booking_history.html', bookings=bookings)
```