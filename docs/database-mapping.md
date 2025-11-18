# Sanad HRMS Database Field Mapping

This document maps the Sanad HRMS UI screens to underlying database structures. It covers the core tables needed for login, HRMS, CRM, sales, and admin settings, plus optional role and incentive extensions.

## Core Tables Overview

- `users`: authentication, authorization, and optional linkage to employee records.
- `employees`: master data for staff including designation and salary.
- `attendance`: daily and monthly attendance data.
- `customers`: CRM records and visit context for sales linkage.
- `services`: service catalog for sales and reporting.
- `sales`: daily transactions connecting customers, services, and staff.
- `settings`: key-value configuration for company, attendance, and incentive rules.
- Optional: `roles`, `user_roles`, `incentives` for finer-grained access and incentive tracking.

## Screen-to-Table Mapping

### 1. Login Page
**Tables:** `users` (required), `employees` (optional link)

| UI Field | Table | Column | Notes |
| --- | --- | --- | --- |
| Username / Email | `users` | `username`, `email` | Authenticate against either field. |
| Password | `users` | `password_hash` | Store hashed passwords. |
| Remember me | — | — | Handle via auth/session layer. |

**Key columns:** `role` (`admin`, `hr`, `staff`, `accountant`), `employee_id` (FK → `employees.id`, nullable), `status`, `last_login_at`, timestamps.

### 2. Employees List / Add Employee
**Table:** `employees`

| UI Field | Column | Type / Notes |
| --- | --- | --- |
| Employee ID | `employee_code` | Varchar (e.g., `SND001`, auto-generated). |
| Full Name | `full_name` | Varchar. |
| Mobile | `mobile` | Varchar. |
| Email | `email` | Varchar. |
| Designation | `designation` | Varchar or FK to `designations` if added. |
| Civil / National ID | `civil_id` | Varchar. |
| Joining Date | `joining_date` | Date. |
| Basic Salary | `basic_salary` | Decimal(10,3). |
| Status | `status` | Enum: `active`, `inactive`. |

Use `created_at` / `updated_at` for audit; set `status='inactive'` when deactivating.

### 3. Attendance Page
**Table:** `attendance` (FK to `employees`)

| UI Field | Column | Type / Notes |
| --- | --- | --- |
| Date | `date` | Date. |
| Employee | `employee_id` | FK → `employees.id`. |
| In Time | `in_time` | Time, nullable. |
| Out Time | `out_time` | Time, nullable. |
| Status | `status` | Enum: `present`, `absent`, `leave`. |
| Late? | `is_late` | Boolean, auto based on settings. |
| Late Minutes | `late_minutes` | Int, computed. |
| Remarks | `remarks` | Varchar, nullable. |

**Monthly view:** query by `employee_id` and date range. Late rules derive from `settings` (`office_start_time`, `late_grace_minutes`, `late_penalty_per_minute`).

### 4. Customers (CRM)
**Table:** `customers`

| UI Field | Column | Type / Notes |
| --- | --- | --- |
| Customer ID | `customer_code` | Optional varchar (e.g., `CUST0001`). |
| Full Name | `full_name` | Varchar. |
| Mobile | `mobile` | Varchar. |
| WhatsApp | `whatsapp` | Varchar. |
| Email | `email` | Varchar, nullable. |
| Nationality | `nationality` | Varchar, nullable. |
| Civil ID / Passport | `id_number` | Varchar, nullable. |
| Purpose | `purpose` | Varchar or FK to `services`. |
| Source | `source` | Enum: `walk_in`, `whatsapp`, `reference`, `social_media`, `other`. |
| Notes | `notes` | Text, nullable. |

Customer visit history uses `sales` records filtered by `customer_id`.

### 5. Services Master
**Table:** `services`

| UI Field | Column | Type / Notes |
| --- | --- | --- |
| Service Name | `service_name` | Varchar. |
| Code / Category | `service_code`, `category` | Varchar, `category` nullable. |
| Default Commission % | `default_commission_percent` | Decimal(5,2), nullable. |
| Active | `is_active` | Boolean. |

### 6. Sales → New Sale & Reports
**Table:** `sales` (FKs to `customers`, `employees`, `services`)

| UI Field | Column | Type / Notes |
| --- | --- | --- |
| Invoice / Receipt No. | `invoice_no` | Varchar. |
| Date | `date` | Date (default today). |
| Time | `time` | Time (default now). |
| Customer | `customer_id` | FK → `customers.id`. |
| Service Type | `service_id` | FK → `services.id`. |
| Service Description | `service_description` | Varchar or text. |
| Amount (OMR) | `amount` | Decimal(10,3). |
| Payment Mode | `payment_mode` | Enum: `cash`, `card`, `online`, `bank_transfer`. |
| Staff / Employee | `employee_id` | FK → `employees.id`. |

**Reports:**
- Daily: filter by date/employee/service/payment, joining customers and employees.
- Monthly: group by `service_id` and `employee_id` for totals and counts.

### 7. Settings / Admin
**Table:** `settings` (key-value)

| UI Field | Key | Value Example |
| --- | --- | --- |
| Company Name | `company_name` | `"Israh For Trade & Investment – Sanad Services"` |
| Logo Upload | `logo_url` | URL/path string. |
| Address / Contact | `company_address`, `company_contact` | Strings. |
| Default Currency | `default_currency` | `"OMR"` |
| Office Start Time | `office_start_time` | `"08:30"` |
| Grace Minutes | `late_grace_minutes` | `"0"` |
| Late Mark Rule | `late_penalty_per_minute` | `"0.010"` |
| Working Hours | `working_hours_per_day` | `"8"` |
| Attendance Incentive | `attendance_incentive_per_day` | `"1.000"` |
| Sales Incentive % | `sales_incentive_percent` | `"5.0"` |
| Target Bonus Rules | `target_bonus_rules` | JSON or text. |
| Late / Absence Deductions | `late_deduction`, `absence_deduction` | Decimals as strings. |

### 8. User Roles (Optional Extension)
- Simple: `users.role` enum (`admin`, `hr`, `staff`, `accountant`).
- Flexible: add `roles` (id, name, description) and `user_roles` (user_id, role_id).

### 9. Incentives (Optional Extension)
**Table:** `incentives`

| Column | Type / Notes |
| --- | --- |
| `employee_id` | FK → `employees.id`. |
| `month` / `year` | Ints for period. |
| `attendance_incentive` | Decimal(10,3). |
| `sales_incentive` | Decimal(10,3). |
| `target_bonus` | Decimal(10,3). |
| `late_deduction` | Decimal(10,3). |
| `absence_deduction` | Decimal(10,3). |
| `total_incentive` | Decimal(10,3). |
| Timestamps | `created_at`, `updated_at`. |

**Net salary example (future payroll):**
```
net_salary = basic_salary + total_incentive - other_deductions
```

## Summary for the Development Team
Start with `users`, `employees`, `attendance`, `customers`, `services`, `sales`, and `settings`. Layer in `roles`/`user_roles` and `incentives` as needed for advanced permissions and payroll.
