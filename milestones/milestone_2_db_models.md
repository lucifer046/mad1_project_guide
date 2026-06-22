```

```

## PHASE 2: DATABASE MODELS (Day 3–5)

> 📺 Watch: [SQLAlchemy Models Hindi](https://www.youtube.com/watch?v=cYWiDiIUxQc)

---

## 🧠 Big Picture: What is a Database Model?

Before writing a single line of code, you need to understand **why** we have models.

Every web application stores data — user accounts, products, orders, bookings. This data lives in a **database**. A database is like a spreadsheet full of tables. Each table has **rows** (records) and **columns** (fields).

**The problem**: Python doesn't naturally talk to databases. Python works with objects (classes, instances), but databases understand SQL (Structured Query Language).

**The solution**: An **ORM — Object-Relational Mapper**.

> 💡 **Think of an ORM as a translator between Python and your Database.**
> Instead of writing: `INSERT INTO user (name, email) VALUES ('Raj', 'raj@gmail.com')`
> You write: `user = User(name='Raj', email='raj@gmail.com'); db.session.add(user)`

We use **Flask-SQLAlchemy**, which wraps the powerful SQLAlchemy ORM library and integrates it seamlessly with Flask.

---

## Step 2.1 — Create `config.py`

### 📖 Why Separate Config from App Code?

Imagine you are building a house. The **blueprint** (design) is separate from the **construction materials** (bricks, wood). Similarly, web apps have:

- **Configuration** → What database to use? What's the secret key? Are we in debug mode?
- **Application Logic** → What happens when user visits `/login`? How do we create a trek?

Mixing these together creates problems:

1. You need to change database paths when deploying to production — if it's scattered everywhere, you break things.
2. Teammates can't safely share code without exposing secrets.
3. Testing becomes impossible because you can't swap the test DB for the real one.

**The golden rule**: Keep config in one place.

### 🔑 Understanding Each Config Variable

#### `SECRET_KEY`

Flask needs to **sign** (cryptographically protect) its session cookies. When a user logs in, Flask creates a small encrypted token that identifies them. Without a secret key, anyone could fake a login session.

```
User logs in → Flask creates session cookie → Signs it with SECRET_KEY → Sends to browser
Browser sends cookie → Flask verifies signature with SECRET_KEY → Trusts the session
```

If someone knows your `SECRET_KEY`, they can forge any user's session. **Never commit this to GitHub.**

Using `os.environ.get('SECRET_KEY')` first means: *"Check if there's an environment variable set. If not, fall back to the hardcoded value."* This is the standard professional pattern.

#### `SQLALCHEMY_DATABASE_URI`

This is the **address** of your database. The format is:

```
dialect+driver://username:password@host:port/database_name
```

For SQLite (a simple file-based database, perfect for development):

```
sqlite:///trekking.db
```

- `sqlite:///` = use SQLite with a relative file path
- `trekking.db` = the file will be created in your project folder

SQLite is perfect for learning because it's **zero-configuration** — no server to install, no ports to configure.

#### `SQLALCHEMY_TRACK_MODIFICATIONS = False`

By default, SQLAlchemy tracks every change made to any object and fires "signals". This feature consumes **extra memory and CPU** for something most apps don't need. Setting it to `False` silences a deprecation warning and improves performance.

```python
# config.py
import os

class Config:
    # SECRET_KEY: Try environment variable first, then use fallback
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'your-super-secret-key-change-this'
  
    # DATABASE_URI: Where is our database file?
    SQLALCHEMY_DATABASE_URI = 'sqlite:///trekking.db'
  
    # Disable modification tracking to save memory
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

---

## Step 2.2 — Create `models.py`

### 📖 Understanding Database Tables, Rows, and Columns

Picture this spreadsheet — this is what the `user` table looks like in the database:

| id | name      | email          | role  | is_blacklisted |
| -- | --------- | -------------- | ----- | -------------- |
| 1  | Admin     | admin@trek.com | admin | False          |
| 2  | Raj Kumar | raj@gmail.com  | user  | False          |
| 3  | Priya     | priya@trek.com | staff | False          |

In SQLAlchemy:

- **The class** = the table structure (columns)
- **An instance of the class** = one row in the table
- **Each attribute** = one column value for that row

### 📖 How SQLAlchemy Column Types Map to Real Data

| SQLAlchemy Type  | What it stores            | SQL Equivalent |
| ---------------- | ------------------------- | -------------- |
| `db.Integer`   | Whole numbers (1, 42, -5) | `INTEGER`    |
| `db.String(n)` | Text up to n characters   | `VARCHAR(n)` |
| `db.Text`      | Long text (no limit)      | `TEXT`       |
| `db.Boolean`   | True or False             | `BOOLEAN`    |
| `db.DateTime`  | Date + Time               | `DATETIME`   |
| `db.Date`      | Date only (no time)       | `DATE`       |

### 📖 Understanding Relationships Between Tables

In our app, tables are **related** to each other:

- One **User** can have many **Bookings** → One-to-Many
- One **Trek** can have many **Bookings** → One-to-Many
- One **User** (if staff) has exactly one **Staff** profile → One-to-One

```
User ──(makes many)──> Bookings <──(belongs to one)── Trek
User ──(has one)──────> Staff Profile
User ──(assigned to many)──> Treks
```

**How do databases link tables?** With **Foreign Keys**.

A Foreign Key is a column in one table that references the Primary Key of another table. In the `bookings` table:

- `user_id` references `user.id` (which user made this booking?)
- `trek_id` references `trek.id` (which trek was booked?)

### 📖 What is `backref`?

When you define `db.relationship('Booking', backref='trekker')` on the `User` model, SQLAlchemy does two things automatically:

1. Adds `user.bookings` → gives you all bookings for a user
2. Adds `booking.trekker` → gives you the user who made that booking (the "back reference")

Without `backref`, you'd have to define relationships explicitly on both sides.

### 📖 What is `lazy=True`?

When you access `user.bookings`, SQLAlchemy has to **query the database** to fetch them. **When** does it do this?

- `lazy=True` (default): Only fetches from DB when you actually access `user.bookings`. Called **Lazy Loading**.
- `lazy='eager'`: Fetches everything upfront in one big JOIN query.

**Lazy loading** is usually better for performance — don't load data you don't need.

### 📖 Password Security — Why We NEVER Store Plain Passwords

If someone hacks your database and steals the `user` table, what do they see?

❌ **Without hashing**: They see everyone's actual passwords → Disaster
✅ **With hashing**: They see scrambled text like `pbkdf2:sha256:260000$...` → Useless to them

**Hashing** is a one-way mathematical function:

```
"mysecret123" → hash function → "pbkdf2:sha256:abc123xyz..."
```

You can NEVER reverse this — you can't go from the hash back to "mysecret123".

**Then how does login work?** When a user logs in:

1. They enter their password ("mysecret123")
2. We hash what they entered
3. We compare that hash against the stored hash
4. If they match → valid password

This is why `generate_password_hash()` and `check_password_hash()` from Werkzeug are crucial.

### 📖 What is a Composite Unique Constraint?

For the `Booking` model, we use:

```python
__table_args__ = (db.UniqueConstraint('user_id', 'trek_id', name='unique_user_trek_booking'),)
```

This tells the database: **"The combination of user_id + trek_id must be unique across all rows."**

So user #2 can book trek #5 **once**. But they cannot have TWO rows where both `user_id=2` and `trek_id=5`.

Without this, a user could accidentally book the same trek multiple times by clicking the button twice.

---

```python
# models.py
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash

# ─────────────────────────────────────────────────────────
# db = SQLAlchemy()
#
# We create a single, shared SQLAlchemy instance here.
# We do NOT pass the Flask app to it yet — that happens in app.py.
# This pattern is called "late binding" and prevents circular imports.
# ─────────────────────────────────────────────────────────
db = SQLAlchemy()


# ═══════════════════════════════════════════════════════
# MODEL 1: User
# Represents EVERY person who has an account in our system.
# Admins, Staff, and regular Users all live in this ONE table.
# Role-Based Access Control (RBAC) is handled via the `role` column.
# ═══════════════════════════════════════════════════════
class User(db.Model):
    __tablename__ = 'user'  # The actual SQL table name

    # ── Primary Key ──
    # autoincrement=True: DB automatically assigns 1, 2, 3... to new users
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)

    # ── Core Identity Fields ──
    name     = db.Column(db.String(96), nullable=False)           # Max 96 chars
    email    = db.Column(db.String(128), unique=True, nullable=False)  # Must be unique!
    password = db.Column(db.String(256), nullable=False)          # Stores HASH not plain text

    # ── Role Column ──
    # This single column controls what the user can access.
    # Values: 'admin' | 'staff' | 'user'
    role = db.Column(db.String(48), nullable=False)

    # ── Status Flags ──
    is_active      = db.Column(db.Boolean, default=True)   # Can they log in?
    is_blacklisted = db.Column(db.Boolean, default=False)  # Are they banned?

    # ── Audit Timestamp ──
    # db.func.current_timestamp() = use the DATABASE's current time, not Python's
    # This is more reliable than datetime.utcnow() because it's DB-server time
    created_at = db.Column(db.DateTime, default=db.func.current_timestamp())

    # ──────────────────────────────────────────────────
    # RELATIONSHIPS
    # These don't create columns. They let Python navigate between related objects.
    # ──────────────────────────────────────────────────

    # One User → Many Bookings
    # backref='trekker' means: on any Booking object, booking.trekker gives back this User
    bookings = db.relationship('Booking', backref='trekker', lazy=True)

    # One Staff User → Many Assigned Treks
    # foreign_keys is required because Trek has TWO FKs pointing to User
    # (one for assigned_staff_id). We must tell SQLAlchemy which FK this relationship uses.
    assigned_treks = db.relationship(
        'Trek',
        backref='assigned_staff',
        lazy=True,
        foreign_keys='Trek.assigned_staff_id'
    )

    # One User → One Staff Profile (if staff)
    # uselist=False = don't return a list, return a single object (One-to-One)
    staff_profile = db.relationship('Staff', backref='user', uselist=False)

    # ──────────────────────────────────────────────────
    # METHODS
    # ──────────────────────────────────────────────────

    def set_password(self, password):
        """
        NEVER store passwords as plain text.
        generate_password_hash() uses PBKDF2-SHA256 algorithm with a random salt.
        Each call produces a DIFFERENT hash even for the same password!
        """
        self.password = generate_password_hash(password)

    def check_password(self, password):
        """
        To verify login: hash what the user typed, compare with stored hash.
        Returns True if correct, False otherwise.
        """
        return check_password_hash(self.password, password)

    def __repr__(self):
        # How this object appears in Python console / debug output
        return f'<User {self.email}>'


# ═══════════════════════════════════════════════════════
# MODEL 2: Trek
# Represents a single trekking route/event.
# Has capacity management via max_slots and available_slot.
# ═══════════════════════════════════════════════════════
class Trek(db.Model):
    __tablename__ = 'trek'

    id       = db.Column(db.Integer, primary_key=True, autoincrement=True)
    name     = db.Column(db.String(128), nullable=False)
    location = db.Column(db.String(56), nullable=False)

    # difficulty: 'Easy', 'Moderate', or 'Hard'
    difficulty = db.Column(db.String(20), nullable=False)

    # ── Capacity Management ──
    # max_slots     = The TOTAL number of people allowed on this trek
    # available_slot = How many slots are STILL free (starts equal to max_slots)
    # Example: max_slots=20, available_slot=17 means 3 people have booked.
    max_slots      = db.Column(db.Integer, nullable=False)
    available_slot = db.Column(db.Integer, nullable=False)

    # stay_duration: How many days the trek lasts (e.g., 3 days)
    stay_duration = db.Column(db.Integer, nullable=False)

    # Date range for the trek
    start_date = db.Column(db.Date, nullable=False)
    end_date   = db.Column(db.Date, nullable=False)

    # ── Status Lifecycle ──
    # A trek moves through these states:
    # pending → approved → open → ongoing → completed
    #                    → closed (if cancelled or full)
    status = db.Column(db.String(32), default='pending')

    # ── Staff Assignment ──
    # Foreign Key: which User (role='staff') is guiding this trek?
    # nullable=True because a trek may not have a staff guide yet
    assigned_staff_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=True)

    # Long description of the trek route
    description = db.Column(db.Text, default=' ')

    # One Trek → Many Bookings
    # backref='trek' means: on any Booking object, booking.trek gives back this Trek
    bookings = db.relationship('Booking', backref='trek', lazy=True)

    def get_booked_slots(self):
        """Count how many active bookings exist for this trek."""
        return Booking.query.filter_by(trek_id=self.id, status='Booked').count()

    def get_remaining_slots(self):
        """Return current available slot count."""
        return self.available_slot

    def __repr__(self):
        return f'<Trek {self.name}>'


# ═══════════════════════════════════════════════════════
# MODEL 3: Booking
# Junction/Association table linking Users ↔ Treks.
# One row = one user's registration to one trek.
# ═══════════════════════════════════════════════════════
class Booking(db.Model):
    __tablename__ = 'bookings'

    # ── Composite Unique Constraint ──
    # A user can only book a given trek ONCE.
    # The DB enforces this at the storage level — no duplicate (user_id, trek_id) pairs allowed.
    __table_args__ = (
        db.UniqueConstraint('user_id', 'trek_id', name='unique_user_trek_booking'),
    )

    id = db.Column(db.Integer, primary_key=True, autoincrement=True)

    # Foreign Keys linking to User and Trek tables
    user_id  = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    trek_id  = db.Column(db.Integer, db.ForeignKey('trek.id'), nullable=False)

    # When was this booking made?
    booking_date = db.Column(db.DateTime, default=db.func.current_timestamp())

    # Booking lifecycle: 'Booked' → 'Cancelled' or 'Completed'
    status = db.Column(db.String(48), default='Booked')

    def __repr__(self):
        return f'<Booking User:{self.user_id} Trek:{self.trek_id}>'


# ═══════════════════════════════════════════════════════
# MODEL 4: Staff
# Extended profile for users with role='staff'.
# Linked to User via One-to-One relationship.
# Holds professional details and admin approval status.
# ═══════════════════════════════════════════════════════
class Staff(db.Model):
    __tablename__ = 'staff'

    id = db.Column(db.Integer, primary_key=True, autoincrement=True)

    # One-to-One with User. unique=True enforces: one staff profile per user account.
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), unique=True, nullable=False)

    contact    = db.Column(db.String(20))   # Phone number
    experience = db.Column(db.Text)         # Bio / years of experience

    # Admin must approve before staff can log in.
    # Values: 'Pending' | 'Approved' | 'Rejected'
    approval_status = db.Column(db.String(20), default='Pending')

    def __repr__(self):
        return f'<Staff user_id:{self.user_id}>'
```

---

## Step 2.3 — Create `app.py` (Application Entry Point)

### 📖 Why Use an Application Factory?

The **simplest** Flask app looks like this:

```python
from flask import Flask
app = Flask(__name__)  # global app object
```

This works for tiny projects, but creates **problems** in larger ones:

- **Circular imports**: `models.py` needs to import `db` from `app.py`, but `app.py` imports from `models.py`
- **Testing**: You can't run two different app configurations (e.g., test DB vs real DB) simultaneously
- **Multiple workers**: Production servers spawn multiple app instances

**The Factory Pattern** solves this by wrapping everything in a function:

```python
def create_app():
    app = Flask(__name__)
    # configure, initialize extensions, register routes...
    return app
```

Now the app object only exists when you **call** `create_app()`.

### 📖 What is an Application Context?

Flask has a concept of "context" — it needs to know **which app** is currently active before running certain operations.

During a web request, Flask automatically pushes an **application context** (it knows which app is handling this request). But when we run `db.create_all()` at startup — before any request exists — we need to push the context manually.

```python
with app.app_context():
    db.create_all()  # ✅ Works because context is active
```

Without `app.app_context()`, SQLAlchemy would raise:

```
RuntimeError: No application found. Either work inside a view function or push an application context.
```

### 📖 What does `db.create_all()` actually do?

It scans **every class** that inherits from `db.Model` in the currently imported modules and generates the SQL to create their tables if they don't exist:

```sql
CREATE TABLE IF NOT EXISTS user (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(96) NOT NULL,
    ...
);
```

This is "Database Migration" at its simplest. You never have to manually write SQL `CREATE TABLE` statements.

### 📖 The Seeding Pattern (create_admin)

"Seeding" means inserting **initial data** into the database when the app first runs. Our app needs at least one admin user to be functional. The `create_admin()` function handles this:

```
App starts → check if admin@trek.com exists → if NO → create it → commit
```

The `.flush()` vs `.commit()` distinction:

- `db.session.flush()` → sends the SQL to DB but doesn't finalize the transaction (still rollback-able)
- `db.session.commit()` → finalizes the transaction permanently

```python
# app.py
# ─────────────────────────────────────────────────────────────────────────
# IMPORTANT: All imports go at the TOP of this file.
# As you complete each Phase, you add more imports here.
# By the end of the project, this section will look like:
# ─────────────────────────────────────────────────────────────────────────
from flask import Flask, render_template, request, redirect, session, flash
from functools import wraps
from datetime import datetime
from config import Config
from models import db, User, Staff, Trek, Booking


# ─────────────────────────────────────────────────────────────────────────
# Step 1: Define helper functions that don't need the app object yet.
# create_admin() is called inside create_app(), so it's defined BEFORE it.
# ─────────────────────────────────────────────────────────────────────────

def create_admin():
    """
    Seed function: Creates a default administrator account on first run.
    Uses .first() to check existence — if admin exists, does nothing.
    """
    admin = User.query.filter_by(email='admin@trek.com').first()

    if not admin:
        admin = User(
            name='Admin',
            email='admin@trek.com',
            role='admin',
            is_active=True
        )
        # set_password() hashes the string before assigning to admin.password
        admin.set_password('admin123')

        db.session.add(admin)    # Stage the new record
        db.session.commit()      # Persist to database file
        print('★ Default Admin created: admin@trek.com / admin123')


def create_app():
    """
    Application Factory: Creates and configures the Flask application.
    Returns the configured app object.
    """
    # Flask(__name__) creates the app.
    # __name__ tells Flask where to look for templates and static files.
    app = Flask(__name__)

    # Load configuration values from our Config class
    app.config.from_object(Config)

    # "Bind" the SQLAlchemy instance to THIS Flask app.
    # db was created in models.py without an app. Now we attach it.
    db.init_app(app)

    # Push application context so DB operations can run at startup
    with app.app_context():
        # Scan all models and create their SQL tables if not already there
        db.create_all()

        # Seed the database with a default admin user
        create_admin()

    return app


# ─────────────────────────────────────────────────────────────────────────
# Step 2: Create the app object.
# ⚠️  ALL @app.route() functions MUST be defined AFTER this line!
#     If you put a route before create_app() is called, Python raises:
#     NameError: name 'app' is not defined
# ─────────────────────────────────────────────────────────────────────────
app = create_app()


# ─────────────────────────────────────────────────────────────────────────
# Step 3: All route functions go here (after app is created).
# Phases 3–7 add routes below this line.
# Example structure:
#
#   @app.route('/login', methods=['GET', 'POST'])
#   def login(): ...
#
#   @app.route('/admin/dashboard')
#   def admin_dashboard(): ...
#
# ─────────────────────────────────────────────────────────────────────────


if __name__ == '__main__':
    # debug=True: auto-reloads server on code changes (NEVER use in production!)
    app.run(debug=True)
```

---

## Step 2.4 — Test Database Creation

### What to Expect

When you run `python app.py` for the first time, SQLAlchemy creates `trekking.db` in your project folder, builds all four tables, and seeds the admin user.

```bash
# From your project root directory
python app.py

# Expected terminal output:
# ★ Default Admin created: admin@trek.com / admin123
# * Running on http://127.0.0.1:5000 (Press CTRL+C to quit)
```

You can verify the database was created:

```bash
ls -la *.db
# Should show: trekking.db
```
