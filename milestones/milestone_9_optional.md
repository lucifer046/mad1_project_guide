## PHASE 9: OPTIONAL FEATURES (Day 26-32 — Do These for Better Score)

### 9.1 — Add Flask-Login (Recommended)

```bash
pip install flask-login
```

```python
# In app.py
from flask_login import LoginManager, UserMixin, login_user, logout_user, current_user

login_manager = LoginManager()
login_manager.init_app(app)

# Make User model inherit UserMixin
class User(db.Model, UserMixin):
    # ... (rest of model stays same)
    pass
```

### 9.2 — Add Charts (Chart.js — Optional)

In your admin dashboard template, add at the bottom:

```html
<canvas id="bookingChart" width="400" height="200"></canvas>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
const ctx = document.getElementById('bookingChart').getContext('2d');
new Chart(ctx, {
    type: 'bar',
    data: {
        labels: {{ trek_names | tojson }},
        datasets: [{
            label: 'Bookings per Trek',
            data: {{ booking_counts | tojson }},
            backgroundColor: 'rgba(54, 162, 235, 0.6)'
        }]
    }
});
</script>
```

### 9.3 — Add API Endpoints (Optional)

```python
# Simple JSON API example
@app.route('/api/treks')
def api_treks():
    from flask import jsonify
    treks = Trek.query.filter_by(status='Open').all()
    result = [{
        'id': t.id,
        'name': t.name,
        'location': t.location,
        'difficulty': t.difficulty,
        'available_slots': t.get_remaining_slots()
    } for t in treks]
    return jsonify(result)

@app.route('/api/treks/<int:trek_id>')
def api_trek_detail(trek_id):
    from flask import jsonify
    trek = Trek.query.get_or_404(trek_id)
    return jsonify({
        'id': trek.id,
        'name': trek.name,
        'location': trek.location,
        'difficulty': trek.difficulty,
        'available_slots': trek.get_remaining_slots(),
        'status': trek.status
    })
```

---