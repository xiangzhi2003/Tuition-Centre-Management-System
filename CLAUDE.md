# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

Run the main application:
```bash
python login.py
```

The application is a CLI-based tuition center management system. Login credentials are stored in [txt_databases/main_database.txt](txt_databases/main_database.txt) with the format: `USERNAME: <username>, PASSWORD: <password>, STATUS: <role>`.

## Architecture Overview

This is a role-based access control (RBAC) system with a text file-based persistence layer. The architecture follows a modular design separating concerns between authentication, role-specific functionality, and shared utilities.

### Entry Point and Authentication

[login.py](login.py) is the main entry point that:
- Handles user authentication (3 attempts limit)
- Logs successful logins to `logged_in_users.txt`
- Dynamically imports and executes the appropriate role module based on user type
- Stores current session information that role modules read to identify the logged-in user

### Role-Based Module System

Each role module ([role_py_files/](role_py_files/)) implements a `main()` function containing:
- A menu loop presenting role-specific functions
- Helper functions for each feature
- A switch/dispatch dictionary mapping user input to functions
- Profile update and logout functionality

**Role modules:**
- [admin.py](role_py_files/admin.py) - Manages tutors, receptionists, and views income reports
- [receptionist.py](role_py_files/receptionist.py) - Handles student registration, enrollment, payments, and receipts
- [tutor.py](role_py_files/tutor.py) - Manages class information and views enrolled students
- [student.py](role_py_files/student.py) - Views schedules, payment status, and sends subject change requests

### Shared Utilities ([peripheral_py_files/](peripheral_py_files/))

- [database_absolute_paths.py](peripheral_py_files/database_absolute_paths.py) - Centralizes database file path management via the `databases` dictionary. Import this to access any txt database file.
- [income_report.py](peripheral_py_files/income_report.py) - Defines the `subjects` dictionary mapping subject names to monthly prices
- [user_messages.py](peripheral_py_files/user_messages.py) - Contains menu text strings for each role and the subjects list display

### Data Persistence ([txt_databases/](txt_databases/))

All data is stored in structured text files with a key-value format:

- **main_database.txt** - User credentials and roles
- **student_database.txt** - Student information: `STUDENT NAME: <name>, LEVEL: Form <1-5>, SUBJECT(S): <subjects>, IC/PASSPORT: <id>, EMAIL: <email>, CONTACT NUMBER: <phone>, ADDRESS: <address>, MONTH OF ENROLLMENT: <month>`
- **tutor_database.txt** - Tutor information: `TUTOR NAME: <name>, LEVEL: Form <1-5>, SUBJECT: '<subject>'`
- **receptionist_database.txt** - Receptionist names
- **classes_database.txt** - Class schedules with tutor, subject, charge, level, schedule, and time
- **payment_status.txt** - Student payment status: `STUDENT NAME: <name>, PAYMENT STATUS: <Paid|Unpaid>`
- **pending_requests.txt** - Student requests for subject changes
- **logged_in_users.txt** - Current session information (cleared on logout)

## Key Design Patterns

### Current User Identification
Role modules identify the logged-in user by reading `logged_in_users.txt`, parsing the username from the first line containing "(", and using it to query other databases. This pattern appears in all profile update and user-specific query functions.

### Pricing Calculation
Monthly tuition fees = (level × 1000) + sum of subject prices. Subject prices are defined in [income_report.py:1-10](peripheral_py_files/income_report.py#L1-L10). Per-session charges shown in class information are calculated as `monthly_price / 4`.

### Text File Parsing
Data is extracted using string operations:
- `split()` to separate key-value pairs
- `index()` + slicing to extract specific fields between markers
- Case normalization (`.lower()` for usernames, `.title()` for names)

### Temporary Passwords
When admins/receptionists create new users, 6-digit numeric passwords are randomly generated using `random.choice(string.digits)`.

## Important Implementation Notes

- **Subject validation**: All subject inputs must match keys in the `subjects` dictionary from [income_report.py](peripheral_py_files/income_report.py) (title case)
- **Level validation**: Levels must be integers between 1-5 (Form 1 to Form 5)
- **Day validation**: Class schedules only accept Monday-Friday (tutor.py enforces this)
- **Student enrollment limit**: Students can enroll in exactly 3 subjects
- **Cross-file updates**: Deleting users requires updating multiple files (e.g., deleting a student updates main_database.txt, student_database.txt, and payment_status.txt)
- **Session management**: Always clear `logged_in_users.txt` on logout to prevent session leaks

## Common Operations

### Adding a new subject
1. Add to `subjects` dictionary in [income_report.py](peripheral_py_files/income_report.py)
2. Update `subjects_list` string in [user_messages.py](peripheral_py_files/user_messages.py)

### Reading logged-in user
```python
with open(databases["logged_in_users.txt"], "r") as liu:
    for line in liu:
        if "(" in line:
            username = line.split()[0]
            break
```

### Updating text file records
Standard pattern: read all lines, filter/modify, write back:
```python
with open(filepath, "r") as f:
    lines = f.readlines()
with open(filepath, "w") as f:
    f.writelines(line for line in lines if condition)
```
