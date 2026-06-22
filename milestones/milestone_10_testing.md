## PHASE 10: TESTING CHECKLIST (Day 32-34)

Go through every feature and test it:

```
Auth:
[ ] Register as User → Login → See user dashboard
[ ] Register as Staff → Login → See "pending approval" message
[ ] Admin login → See admin dashboard
[ ] Blacklisted user cannot login

Admin:
[ ] Create a trek
[ ] Edit a trek
[ ] Delete a trek
[ ] Approve staff → staff can now login
[ ] Assign staff to trek
[ ] Blacklist user
[ ] Search for trek/user/staff

Staff:
[ ] See only assigned treks
[ ] Update trek slots
[ ] Change trek status to Open/Closed/Completed
[ ] View registered trekkers

User:
[ ] See only Open treks
[ ] Filter by difficulty/location
[ ] Book a trek → slot decreases by 1
[ ] Can't book same trek twice
[ ] Can't book fully booked trek
[ ] Cancel booking
[ ] View booking history
```

---