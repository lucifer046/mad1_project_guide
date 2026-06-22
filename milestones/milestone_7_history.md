## PHASE 7: BOOKING HISTORY & STATUS TRACKING (Day 24-26)

### Step 7.1 — Create `templates/user/booking_history.html`

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
            <th>#</th><th>Trek Name</th><th>Location</th><th>Difficulty</th>
            <th>Date Booked</th><th>Trek Status</th><th>Booking Status</th><th>Action</th>
        </tr>
    </thead>
    <tbody>
        {% for b in bookings %}
        <tr>
            <td>{{ b.id }}</td>
            <td>{{ b.trek.name }}</td>
            <td>{{ b.trek.location }}</td>
            <td>{{ b.trek.difficulty }}</td>
            <td>{{ b.booking_date.strftime('%d %b %Y') }}</td>
            <td>{{ b.trek.status }}</td>
            <td>
                <span class="badge 
                    {% if b.status == 'Booked' %}bg-success
                    {% elif b.status == 'Cancelled' %}bg-secondary
                    {% else %}bg-primary{% endif %}">
                    {{ b.status }}
                </span>
            </td>
            <td>
                {% if b.status == 'Booked' %}
                    <a href="/user/cancel_booking/{{ b.id }}" class="btn btn-sm btn-outline-danger"
                       onclick="return confirm('Cancel booking?')">Cancel</a>
                {% endif %}
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% else %}
    <div class="alert alert-info">No treks booked yet. <a href="/user/treks">Explore now!</a></div>
{% endif %}
{% endblock %}
```

---