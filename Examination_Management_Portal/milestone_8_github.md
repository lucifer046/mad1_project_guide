## PHASE 8: GITHUB SETUP & COLLABORATION (Day 1 — Required Before Submission)

> ⚠️ **Mandatory Course Requirement**: The Git repository must be initialized on Day 1, pushed to GitHub as a **Private** repository, and the course evaluator account must be invited as a collaborator.

---

## 🛠️ Step 8.1 — Initialize Git Locally

In your project root directory (`emp/`):

```bash
# 1. Initialize local repository
git init

# 2. Add all initial project files
git add .

# 3. Create initial commit
git commit -m "Milestone-0 EMP-MAD-1"
```

---

## 🌐 Step 8.2 — Create GitHub Repository & Push

1. Open [github.com](https://github.com) and click **New repository**.
2. Name the repository: `examination-management-portal` (or your assigned naming convention).
3. Select **Private**.
4. Leave "Add a README file" unchecked (you already have one locally).
5. Link and push your local repo:

```bash
# Set default branch name to main
git branch -M main

# Add remote origin URL (replace with your GitHub username)
git remote add origin https://github.com/YOUR_USERNAME/examination-management-portal.git

# Push your initial commit
git push -u origin main
```

---

## 👥 Step 8.3 — Add Course Evaluator as Collaborator

1. In GitHub, go to your repository page.
2. Click **Settings** (tab at the top).
3. In the left sidebar, click **Collaborators**.
4. Click **Add people**.
5. Enter the course evaluation handle: `MADI-cs2003` (or the evaluator handle specified in your portal instructions).
6. Send the invitation.

---

## 📅 Step 8.4 — Milestone Commit Schedule

Commit your work progressively as each milestone is completed:

```bash
# After Milestone 2 (Models & Skeleton):
git add .
git commit -m "Milestone: Database Models and Schema Setup"
git push

# After Milestone 3 (Authentication & Base Layout):
git add .
git commit -m "Milestone: Authentication and Role-Based Access"
git push

# After Milestone 4 (Admin Dashboard & Management):
git add .
git commit -m "Milestone: Admin Dashboard and Management"
git push

# After Milestone 5 (Examiner Dashboard & Slots):
git add .
git commit -m "Milestone: Examiner Dashboard and Slot Management"
git push

# After Milestone 6 (Student Dashboard & Booking):
git add .
git commit -m "Milestone: Student Dashboard and Exam Booking System"
git push

# After Milestone 7 (History & Results Tracking):
git add .
git commit -m "Milestone: Booking History and Evaluation Tracking"
git push

# Final Submission Commit:
git add .
git commit -m "Milestone-EMP Final-Submission"
git push
```
