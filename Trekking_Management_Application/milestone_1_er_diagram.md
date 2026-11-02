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
| stay_duration | Integer | Number of days |
| max_slots | Integer | Max bookings allowed |
| available_slot | Integer | Current available slots |
| start_date | Date | |
| end_date | Date | |
| status | String | 'pending', 'approved', 'open', 'closed', 'completed' |
| assigned_staff_id | Integer | Foreign Key → User.id |
| description | Text | Optional |

#### Table 3: `Booking` (Table name: `bookings`)
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key, Auto |
| user_id | Integer | Foreign Key → User.id |
| trek_id | Integer | Foreign Key → Trek.id |
| booking_date | DateTime | Auto |
| status | String | 'Booked', 'Cancelled', 'Completed' |

*Note: Unique constraint on (user_id, trek_id).*

#### Table 4: `Staff` (Table name: `staff`)
| Column | Type | Notes |
|--------|------|-------|
| id | Integer | Primary Key |
| user_id | Integer | Foreign Key → User.id, Unique |
| contact | String | Phone number |
| experience | Text | Experience details |
| approval_status | String | 'Pending', 'Approved', 'Rejected' |

### Relationships
```
User (staff) ─────< Trek         [One staff can have many treks]
User (trekker) ───< Booking       [One user can have many bookings]
Trek ──────────────< Booking      [One trek can have many bookings]
User (staff) ──────── Staff       [One-to-one]
```

### How to Draw ER Diagram
Use **draw.io** (free online): https://app.diagrams.net/
- Create boxes for each table
- Add columns inside each box
- Draw arrows for relationships (crow's foot notation)
- Export as PNG for your report

---