## PHASE 1: DESIGN ER DIAGRAM (Day 2-3)

>  **Do this BEFORE writing any code.** The ER diagram is the blueprint of your database.

### What is an ER Diagram?

An ER (Entity-Relationship) diagram shows:
- What **tables** (entities) you have
- What **columns** (attributes) each table has
- How tables are **connected** (relationships)

### Your Tables

#### Table 1: `User`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key, Auto |
| name | String | Full name |
| email | String | Unique |
| password | String | Hashed |
| role | String | 'admin', 'staff', 'user' |
| is_active | Boolean | Default True |
| is_blacklisted | Boolean | Default False |
| created_at | DateTime | Auto |

#### Table 2: `Trek`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key, Auto |
| name | String | Trek name |
| location | String | e.g. "Manali" |
| difficulty | String | 'Easy', 'Moderate', 'Hard' |
| duration_days | Integer | Number of days |
| available_slots | Integer | Max bookings allowed |
| start_date | Date | |
| end_date | Date | |
| status | String | 'Pending','Approved','Open','Closed','Completed' |
| assigned_staff_id | Integer | Foreign Key → User.id |
| description | Text | Optional |

#### Table 3: `Booking`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key, Auto |
| user_id | Integer | Foreign Key → User.id |
| trek_id | Integer | Foreign Key → Trek.id |
| booking_date | DateTime | Auto |
| status | String | 'Booked', 'Cancelled', 'Completed' |

#### Table 4: `StaffProfile` (Optional but Recommended)
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key |
| user_id | Integer | Foreign Key → User.id |
| contact | String | Phone number |
| experience | String | Experience details |
| approval_status | String | 'Pending', 'Approved', 'Rejected' |

### Relationships
```
User (staff) ─────< Trek         [One staff can have many treks]
User (trekker) ───< Booking       [One user can have many bookings]
Trek ──────────────< Booking      [One trek can have many bookings]
User (staff) ──────── StaffProfile [One-to-one]
```

### How to Draw ER Diagram
Use **draw.io** (free online): https://app.diagrams.net/
- Create boxes for each table
- Add columns inside each box
- Draw arrows for relationships (crow's foot notation)
- Export as PNG for your report

---