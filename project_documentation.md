# Trekking Management Application - Complete Project Documentation

Welcome to the comprehensive guide for your Trekking Management Application! I've analyzed your project structure and requirements, and I've put together this detailed explanation. We will focus purely on the **core, mandatory requirements** and delve deeply into the database creation and relationships. 

Think of this document as a guided tour through your project's architecture. 

---

## 1. Project Overview

We are building a web-based **Trekking Management Application** designed to streamline operations for adventure organizations. Instead of using messy spreadsheets, this application brings Admins, Trek Staff, and Trekkers into a single, cohesive system.

### Tech Stack (Core Restrictions)
- **Backend:** Flask (Python)
- **Frontend:** Jinja2 templating, HTML, CSS, Bootstrap 5
- **Database:** SQLite (using SQLAlchemy ORM)
- **Rule:** No JavaScript for core functionalities.
- **Rule:** Database must be created programmatically (no manual DB browser tools).

### User Roles
1. **Admin:** The pre-existing superuser. Manages everything (treks, staff approvals, staff assignments, user blacklisting).
2. **Trek Staff:** Guides who manage the actual treks. They need to register, get approved by the Admin, and can then manage their assigned treks (updating slots, viewing trekkers).
3. **User (Trekker):** The customers. They register, browse available treks, book slots, and view their history.

---

## 2. Database Design & Relationships (The Core Engine)

The database is the heart of this application. Since we are using SQLAlchemy, we define our database tables as Python classes (models). Let's break down the schema and understand how these tables talk to each other.

### The Entities (Tables)

We need four primary tables:
1. `User`: Stores all people in the system (Admin, Staff, Trekkers).
2. `Trek`: Stores details about the trekking events.
3. `Booking`: The bridge table mapping users to the treks they booked.
4. `StaffProfile`: Extra details specifically for staff members (like approval status).

### Detailed Model Breakdown

#### 1. The `User` Model
This table holds login credentials and role information for everyone.

*   **Primary Key:** `id`
*   **Fields:** `name`, `email`, `password` (hashed), `role` ('admin', 'staff', 'user'), `is_active`, `is_blacklisted`.
*   **Why one table?** It's easier for authentication. We differentiate what a person can do using the `role` column.

#### 2. The `Trek` Model
This table holds the catalog of treks.

*   **Primary Key:** `id`
*   **Fields:** `name`, `location`, `difficulty`, `duration_days`, `available_slots`, `status` (Pending/Open/Closed/Completed), `start_date`, `end_date`.
*   **Foreign Key:** `assigned_staff_id`. This points back to the `User` table (specifically a user with the 'staff' role).

#### 3. The `Booking` Model
This is a **transactional table**. It records the event of a Trekker booking a Trek.

*   **Primary Key:** `id`
*   **Foreign Key 1:** `user_id` (Points to the Trekker).
*   **Foreign Key 2:** `trek_id` (Points to the Trek).
*   **Fields:** `booking_date`, `status` (Booked/Cancelled/Completed).

#### 4. The `StaffProfile` Model
Since staff members have extra data that regular users don't need (like experience and approval status), we separate it to keep the `User` table clean.

*   **Primary Key:** `id`
*   **Foreign Key:** `user_id` (Points to the User).
*   **Fields:** `contact`, `experience`, `approval_status` (Pending/Approved/Rejected).

### The Relationships Explained

Here is how SQLAlchemy links them together logically:

1. **One-to-Many: User (Admin) to Trek (Staff Assignment)**
   *   *Logic:* One Staff member can be assigned to *many* Treks. 
   *   *Implementation:* The `Trek` table has an `assigned_staff_id`. In the `User` model, we define a relationship `assigned_treks = db.relationship('Trek', backref='assigned_staff')`.

2. **Many-to-Many: User (Trekker) to Trek (via Booking)**
   *   *Logic:* A User can book many Treks. A Trek can have many Users booked.
   *   *Implementation:* The `Booking` table acts as the middleman (association table). A User has many Bookings, and a Trek has many Bookings.

3. **One-to-One: User to StaffProfile**
   *   *Logic:* One User (if they are staff) has exactly one Staff Profile.
   *   *Implementation:* `StaffProfile` has `user_id`. The `User` model has `staff_profile = db.relationship('StaffProfile', backref='user', uselist=False)`.

---

## 3. Programmatic Database Creation

One of the strict rules is that the DB must be created via code. Here is how that flow works when the application starts:

1.  **Flask App Context:** We initialize SQLAlchemy with our Flask app.
2.  **`db.create_all()`:** Inside the application context, we run `db.create_all()`. SQLAlchemy looks at all our defined models and translates them into SQL `CREATE TABLE` commands.
3.  **Auto-creating the Admin:** The prompt states "Admin must pre-exist". Immediately after creating the tables, we write a script that checks if an admin exists. If not, it programmatically injects the Admin user into the database using `db.session.add(admin)`.

---

## 4. Core Application Workflows

Let's walk through how the application operates step-by-step based purely on the core requirements.

### Workflow 1: The Initial State
When the app launches for the first time, the database is generated. The pre-defined Admin exists. No treks exist. No other users exist.

### Workflow 2: Staff Registration & Approval
1.  A person registers as "Trek Staff".
2.  They are added to the `User` table and a `StaffProfile` is created with a status of 'Pending'.
3.  *Crucially*, they **cannot** access the staff dashboard yet.
4.  The Admin logs in, sees the pending staff request, and clicks "Approve". The `StaffProfile` status becomes 'Approved'. Now the staff can access their dashboard.

### Workflow 3: Trek Creation & Assignment
1.  The Admin creates a new Trek route (Name, Location, Slots, etc.). The trek's initial status is likely 'Pending' or 'Open'.
2.  The Admin assigns an approved Trek Staff member to this trek.

### Workflow 4: User Registration & Booking
1.  A standard User (Trekker) registers. They gain immediate access to the User Dashboard.
2.  The User browses treks that have the status 'Open'.
3.  The User books a trek. The system checks:
    *   *Validation:* Is the trek 'Open'?
    *   *Validation:* Are `available_slots` > booked slots?
    *   *Validation:* Has this user already booked this exact trek?
4.  If valid, a new row is added to the `Booking` table.

### Workflow 5: Trek Management (Staff side)
1.  The assigned Trek Staff logs in. They ONLY see treks assigned to them by the Admin.
2.  They monitor how many people have booked.
3.  When the trek is about to start, the Staff changes the Trek status from 'Open' to 'Closed' (so no one else can book).
4.  After the trek, they mark it as 'Completed'.

### Workflow 6: Admin Oversight
1.  Throughout this, the Admin can view all users, all staff, and all treks.
2.  The Admin can search for specific users or treks.
3.  If a user misbehaves, the Admin can set `is_blacklisted = True` in the `User` table, preventing that user from logging in.

---

## 5. Summary of Development Priorities

Since we are focusing *only* on the core milestones right now, here is exactly what you should concentrate on building, in order:

1.  **Model Building:** Write the `models.py` exactly as described above. Ensure `db.create_all()` and the admin injection work flawlessly.
2.  **Auth System:** Build the login/register routes. Ensure role-based redirection (Admin goes to Admin dashboard, User to User dashboard, etc.).
3.  **Admin Core:** Build the pages for Admin to create treks, approve staff, and assign staff.
4.  **Staff Core:** Build the page for Staff to view their assigned treks and update statuses.
5.  **User Core:** Build the page for Users to list open treks, book them, and view their booking history.

*Note: Skip APIs, Chart.js, and advanced Flask-Login security features until these core pillars are functioning perfectly.*
