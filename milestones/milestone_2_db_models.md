## PHASE 2: DATABASE MODELS (Day 3-5)

>  Watch: [SQLAlchemy Models Hindi](https://www.youtube.com/watch?v=cYWiDiIUxQc)

### Step 2.1 — Create `config.py`

```python
# config.py
import os

class Config:
    SECRET_KEY = 'your-super-secret-key-change-this'
    SQLALCHEMY_DATABASE_URI = 'sqlite:///trekking.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

### Step 2.2 — Create `models.py`

```python
# models.py
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime
from werkzeug.security import generate_password_hash, check_password_hash

db = SQLAlchemy()

class User(db.Model):
    __tablename__ = 'user'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password = db.Column(db.String(200), nullable=False)
    role = db.Column(db.String(20), nullable=False)  # 'admin', 'staff', 'user'
    is_active = db.Column(db.Boolean, default=True)
    is_blacklisted = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    # Relationships
    bookings = db.relationship('Booking', backref='trekker', lazy=True)
    assigned_treks = db.relationship('Trek', backref='assigned_staff', lazy=True,
                                      foreign_keys='Trek.assigned_staff_id')
    staff_profile = db.relationship('StaffProfile', backref='user', uselist=False)
    
    def set_password(self, password):
        self.password = generate_password_hash(password)
    
    def check_password(self, password):
        return check_password_hash(self.password, password)
    
    def __repr__(self):
        return f'<User {self.email}>'


class Trek(db.Model):
    __tablename__ = 'trek'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(150), nullable=False)
    location = db.Column(db.String(100), nullable=False)
    difficulty = db.Column(db.String(20), nullable=False)  # Easy/Moderate/Hard
    duration_days = db.Column(db.Integer, nullable=False)
    available_slots = db.Column(db.Integer, nullable=False)
    start_date = db.Column(db.Date, nullable=False)
    end_date = db.Column(db.Date, nullable=False)
    status = db.Column(db.String(20), default='Pending')  # Pending/Approved/Open/Closed/Ongoing/Completed
    assigned_staff_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=True)
    description = db.Column(db.Text, default='')
    
    # Relationships
    bookings = db.relationship('Booking', backref='trek', lazy=True)
    
    def get_booked_slots(self):
        return Booking.query.filter_by(trek_id=self.id, status='Booked').count()
    
    def get_remaining_slots(self):
        return self.available_slots - self.get_booked_slots()
    
    def __repr__(self):
        return f'<Trek {self.name}>'


class Booking(db.Model):
    __tablename__ = 'booking'
    
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    trek_id = db.Column(db.Integer, db.ForeignKey('trek.id'), nullable=False)
    booking_date = db.Column(db.DateTime, default=datetime.utcnow)
    status = db.Column(db.String(20), default='Booked')  # Booked/Cancelled/Completed
    
    def __repr__(self):
        return f'<Booking User:{self.user_id} Trek:{self.trek_id}>'


class StaffProfile(db.Model):
    __tablename__ = 'staff_profile'
    
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    contact = db.Column(db.String(20))
    experience = db.Column(db.String(200))
    approval_status = db.Column(db.String(20), default='Pending')  # Pending/Approved/Rejected
    
    def __repr__(self):
        return f'<StaffProfile user_id:{self.user_id}>'
```

### Step 2.3 — Create `app.py` (Basic Setup)

```python
# app.py
from flask import Flask
from config import Config
from models import db, User, StaffProfile, Trek, Booking
from werkzeug.security import generate_password_hash

def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)
    
    db.init_app(app)
    
    with app.app_context():
        db.create_all()
        create_admin()  # Create default admin
    
    return app

def create_admin():
    """Create admin user if doesn't exist"""
    admin = User.query.filter_by(email='admin@trek.com').first()
    if not admin:
        admin = User(
            name='Admin',
            email='admin@trek.com',
            role='admin',
            is_active=True
        )
        admin.set_password('admin123')  # Change this password!
        db.session.add(admin)
        db.session.commit()
        print(" Admin created: admin@trek.com / admin123")

app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
```

### Step 2.4 — Test Database Creation

```bash
# Run this in terminal from your project folder
python app.py
# You should see: * Running on http://127.0.0.1:5000
# Check that trekking.db file was created
```

---