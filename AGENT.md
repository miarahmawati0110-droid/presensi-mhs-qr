# AGENTS.md

## Project Overview

This project is a **Laravel 12 web application for student attendance using QR Code**.

The application has two user roles:

* `admin`
* `mahasiswa`

The application uses:

* Laravel 12
* PHP 8.2+
* MySQL/MariaDB
* Blade
* Tailwind CSS
* Laravel Eloquent ORM
* Laravel Authentication
* QR Code for attendance
* Vite for frontend asset compilation

Do not introduce unnecessary frontend frameworks such as React, Vue, or Angular unless explicitly requested.

---

# 1. Core Requirements

The system must support:

### Admin

Admin can:

* Login
* View dashboard
* Manage mahasiswa
* Manage mata kuliah
* Manage kelas
* Assign mahasiswa to kelas
* Create pertemuan
* Generate QR Code for a pertemuan
* Activate/deactivate QR Code
* Set QR Code expiration
* View attendance
* Change attendance status when necessary
* View attendance reports
* Filter attendance by:

  * mata kuliah
  * kelas
  * mahasiswa
  * date
  * pertemuan
  * attendance status
* Export attendance reports when required

### Mahasiswa

Mahasiswa can:

* Login
* View dashboard
* View profile
* View registered classes
* Scan QR Code
* Submit attendance
* View personal attendance history
* View attendance percentage/statistics

---

# 2. Roles

The application has exactly two roles:

```text
admin
mahasiswa
```

The role is stored in the `users` table.

Example:

```text
users
├── id
├── name
├── email
├── password
├── role
└── timestamps
```

Use an enum or equivalent constrained value:

```php
$table->enum('role', [
    'admin',
    'mahasiswa',
])->default('mahasiswa');
```

Do not create unnecessary role tables or permission packages unless explicitly requested.

---

# 3. Database Structure

The main entities are:

```text
users
    │
    └── mahasiswas
            │
            └── kelas_mahasiswa
                    │
                    ▼
                   kelas
                    │
                    ▼
               matakuliahs
                    │
                    ▼
               pertemuans
                    │
                    ▼
                presensis
```

## users

Laravel's default users table with an additional `role`.

Important fields:

```text
id
name
email
password
role
remember_token
timestamps
```

---

## mahasiswas

Stores student-specific information.

Recommended fields:

```text
id
user_id
nim
nama
prodi
angkatan
timestamps
```

Rules:

* `user_id` references `users.id`
* `user_id` should be unique
* `nim` must be unique

---

## matakuliahs

Stores courses.

Recommended fields:

```text
id
kode
nama
sks
timestamps
```

Rules:

* `kode` must be unique
* `sks` should use an appropriate small integer type

---

## kelas

Stores course classes.

Recommended fields:

```text
id
matakuliah_id
nama_kelas
dosen
ruangan
timestamps
```

A class belongs to one mata kuliah.

---

## kelas_mahasiswa

Pivot table connecting students and classes.

Recommended fields:

```text
id
kelas_id
mahasiswa_id
timestamps
```

Rules:

```php
$table->unique([
    'kelas_id',
    'mahasiswa_id'
]);
```

A student cannot be registered twice in the same class.

---

## pertemuans

Stores individual class meetings.

Recommended fields:

```text
id
kelas_id
pertemuan_ke
tanggal
jam_mulai
jam_selesai
qr_token
qr_aktif
qr_berlaku_mulai
qr_berlaku_sampai
materi
timestamps
```

Rules:

* `qr_token` must be unique
* A class cannot have duplicate `pertemuan_ke`
* QR Code must be associated with a specific meeting
* QR token must be generated securely

Recommended constraint:

```php
$table->unique([
    'kelas_id',
    'pertemuan_ke'
]);
```

---

## presensis

Stores student attendance.

Recommended fields:

```text
id
pertemuan_id
mahasiswa_id
waktu_presensi
status
ip_address
keterangan
timestamps
```

Allowed status:

```text
hadir
terlambat
izin
sakit
alpha
```

A student may only have one attendance record for one meeting:

```php
$table->unique([
    'pertemuan_id',
    'mahasiswa_id'
]);
```

---

# 4. QR Code Attendance

The QR Code represents a **meeting**, not a student.

Never put sensitive information directly inside the QR Code.

Do not use:

```text
mahasiswa_id
password
user_id
```

as the QR token.

Use a cryptographically secure random token.

Example:

```php
use Illuminate\Support\Str;

$pertemuan->qr_token = Str::random(64);
```

The QR Code should resolve to an attendance endpoint containing the token.

Example concept:

```text
/presensi/scan/{qr_token}
```

The actual route structure may differ, but the QR token must identify the meeting securely.

---

# 5. QR Validation

When a student scans a QR Code, the backend MUST validate:

1. User is authenticated.
2. User has role `mahasiswa`.
3. Student profile exists.
4. QR token exists.
5. Meeting exists.
6. QR Code is active.
7. Current time is within the QR validity period.
8. Student is registered in the class.
9. Student has not already attended the meeting.

Pseudo-flow:

```text
Scan QR
   ↓
Authenticate user
   ↓
Check role = mahasiswa
   ↓
Find meeting by qr_token
   ↓
Check qr_aktif
   ↓
Check QR expiration
   ↓
Check student is enrolled in class
   ↓
Check duplicate attendance
   ↓
Create attendance
   ↓
Return success
```

Never trust the frontend to perform these validations.

All important validation must happen server-side.

---

# 6. Attendance Security

Do not rely only on hiding UI elements.

A student must not be able to:

* Submit attendance for another student
* Submit attendance for a class they are not enrolled in
* Reuse an expired QR Code
* Submit attendance twice
* Modify another student's attendance
* Change attendance status through request manipulation
* Submit attendance by changing IDs in the URL or request

Always derive the student from the authenticated user:

```php
$mahasiswa = auth()->user()->mahasiswa;
```

Do not accept `mahasiswa_id` from the attendance request when it can be derived from the authenticated user.

Bad:

```php
$mahasiswaId = $request->mahasiswa_id;
```

Preferred:

```php
$mahasiswaId = auth()->user()->mahasiswa->id;
```

---

# 7. Authorization

Use middleware and policies where appropriate.

Example route separation:

```php
Route::middleware(['auth', 'role:admin'])
    ->prefix('admin')
    ->name('admin.')
    ->group(function () {
        // Admin routes
    });

Route::middleware(['auth', 'role:mahasiswa'])
    ->prefix('mahasiswa')
    ->name('mahasiswa.')
    ->group(function () {
        // Mahasiswa routes
    });
```

Do not rely on Blade conditions alone for authorization.

This is insufficient:

```blade
@if(auth()->user()->role === 'admin')
```

UI visibility is not authorization.

The backend must enforce access.

---

# 8. Middleware

Create a reusable role middleware.

Example:

```php
if (auth()->user()->role !== $role) {
    abort(403);
}
```

Use Laravel's current middleware registration conventions for Laravel 12.

Do not blindly copy middleware registration patterns from older Laravel versions.

---

# 9. Models

Use Eloquent relationships.

Expected relationships:

### User

```php
public function mahasiswa()
{
    return $this->hasOne(Mahasiswa::class);
}
```

### Mahasiswa

```php
public function user()
{
    return $this->belongsTo(User::class);
}

public function kelas()
{
    return $this->belongsToMany(Kelas::class, 'kelas_mahasiswa');
}

public function presensis()
{
    return $this->hasMany(Presensi::class);
}
```

### MataKuliah

```php
public function kelas()
{
    return $this->hasMany(Kelas::class);
}
```

### Kelas

```php
public function matakuliah()
{
    return $this->belongsTo(MataKuliah::class);
}

public function mahasiswa()
{
    return $this->belongsToMany(Mahasiswa::class, 'kelas_mahasiswa');
}

public function pertemuans()
{
    return $this->hasMany(Pertemuan::class);
}
```

### Pertemuan

```php
public function kelas()
{
    return $this->belongsTo(Kelas::class);
}

public function presensis()
{
    return $this->hasMany(Presensi::class);
}
```

### Presensi

```php
public function pertemuan()
{
    return $this->belongsTo(Pertemuan::class);
}

public function mahasiswa()
{
    return $this->belongsTo(Mahasiswa::class);
}
```

Use proper return types when the project coding standard requires them.

---

# 10. Controllers

Controllers should remain thin.

Avoid putting large business logic directly into controllers.

For example, attendance validation can be moved into:

* Form Requests
* Actions
* Services

depending on complexity.

For a simple application, a controller may handle the workflow directly.

For more complex attendance rules, prefer:

```text
app/
├── Actions/
│   └── Attendance/
├── Services/
│   └── Attendance/
```

Do not create abstractions without a real need.

---

# 11. Form Requests

Use Form Request classes for complex validation.

Example:

```bash
php artisan make:request StoreMahasiswaRequest
```

Validation belongs in the request class when appropriate.

Example:

```php
public function rules(): array
{
    return [
        'nim' => ['required', 'string', 'max:50'],
        'nama' => ['required', 'string', 'max:255'],
        'prodi' => ['nullable', 'string', 'max:255'],
        'angkatan' => ['nullable', 'string', 'max:10'],
    ];
}
```

Never trust request data.

---

# 12. Blade UI

The frontend MUST use:

* Blade
* Tailwind CSS

Do not introduce React/Vue unless explicitly requested.

Use Blade components for reusable UI.

Recommended structure:

```text
resources/views/
├── layouts/
│   ├── app.blade.php
│   ├── guest.blade.php
│   └── navigation.blade.php
│
├── components/
│   ├── button.blade.php
│   ├── input.blade.php
│   ├── modal.blade.php
│   ├── badge.blade.php
│   ├── alert.blade.php
│   └── table.blade.php
│
├── admin/
│   ├── dashboard.blade.php
│   ├── mahasiswa/
│   ├── matakuliah/
│   ├── kelas/
│   ├── pertemuan/
│   └── presensi/
│
└── mahasiswa/
    ├── dashboard.blade.php
    ├── scan.blade.php
    ├── presensi/
    └── profile.blade.php
```

---

# 13. Tailwind CSS Rules

Use Tailwind utility classes directly in Blade.

Example:

```blade
<button
    type="submit"
    class="inline-flex items-center rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
>
    Simpan
</button>
```

Prefer reusable Blade components when a UI pattern appears multiple times.

Avoid excessive custom CSS.

Do not use inline `<style>` blocks unless there is a specific reason.

---

# 14. UI Design

The UI should be:

* Clean
* Modern
* Responsive
* Mobile friendly
* Accessible
* Consistent

Primary target:

```text
Desktop
Tablet
Mobile
```

Student QR scanning is especially important on mobile devices.

The scan page should have:

* Large QR scanning area
* Clear camera permission state
* Loading state
* Success state
* Error state
* Clear instructions

Example UI states:

```text
Ready to scan
       ↓
Scanning...
       ↓
QR detected
       ↓
Validating...
       ↓
Attendance successful
```

---

# 15. Dashboard Admin

Admin dashboard should provide summary information such as:

```text
Total Mahasiswa
Total Mata Kuliah
Total Kelas
Pertemuan Hari Ini
Mahasiswa Hadir Hari Ini
Mahasiswa Terlambat
```

Use cards with Blade components.

Example:

```text
┌──────────────┐ ┌──────────────┐
│ Mahasiswa    │ │ Mata Kuliah  │
│ 250          │ │ 32           │
└──────────────┘ └──────────────┘

┌──────────────┐ ┌──────────────┐
│ Kelas        │ │ Hadir Hari Ini│
│ 18           │ │ 212          │
└──────────────┘ └──────────────┘
```

---

# 16. Dashboard Mahasiswa

The student dashboard should show:

* Student name
* NIM
* Current classes
* Today's meetings
* Attendance percentage
* Recent attendance
* Scan attendance button

Example:

```text
Halo, Andi

NIM: 2201001

Kehadiran
92%

Pertemuan Hari Ini
Pemrograman Web
08:00 - 10:00

[ Scan QR ]
```

---

# 17. Attendance Status

Use these statuses:

```text
hadir
terlambat
izin
sakit
alpha
```

Recommended visual labels:

```text
hadir     → green
terlambat → yellow
izin      → blue
sakit     → orange
alpha     → red
```

Do not rely solely on color.

Always include text/iconography where appropriate for accessibility.

---

# 18. Time Handling

Use Laravel/Carbon for time calculations.

Do not manually compare date/time strings when Carbon can be used.

Example:

```php
now()->between(
    $pertemuan->qr_berlaku_mulai,
    $pertemuan->qr_berlaku_sampai
);
```

Use the application's configured timezone consistently.

Do not hardcode timezone values throughout the codebase.

---

# 19. Database Transactions

Use database transactions when multiple related records must be modified atomically.

Example:

```php
DB::transaction(function () {
    // database operations
});
```

Attendance creation should be designed to prevent duplicate submissions.

The database unique constraint is mandatory even if the application checks duplicates beforehand.

Application validation:

```text
Check duplicate
       ↓
Create attendance
```

Database protection:

```text
UNIQUE(pertemuan_id, mahasiswa_id)
```

Both should exist.

---

# 20. Eager Loading

Avoid N+1 queries.

Bad:

```php
$kelas = Kelas::all();

foreach ($kelas as $item) {
    echo $item->matakuliah->nama;
}
```

Preferred:

```php
$kelas = Kelas::with('matakuliah')->get();
```

Use eager loading when rendering related data.

---

# 21. Pagination

Do not load large datasets with `get()` when pagination is appropriate.

Use:

```php
Mahasiswa::paginate(15);
```

Example in Blade:

```blade
{{ $mahasiswa->links() }}
```

Admin tables should generally use pagination.

---

# 22. Routing

Use named routes.

Example:

```php
Route::get('/dashboard', ...)
    ->name('dashboard');
```

For admin:

```text
admin.dashboard
admin.mahasiswa.index
admin.mahasiswa.create
admin.mahasiswa.store
admin.mahasiswa.edit
admin.mahasiswa.update
admin.mahasiswa.destroy
```

For mahasiswa:

```text
mahasiswa.dashboard
mahasiswa.scan
mahasiswa.presensi
mahasiswa.riwayat
```

Use resource controllers where appropriate.

---

# 23. Route Model Binding

Prefer route model binding instead of manually finding IDs.

Preferred:

```php
public function edit(Mahasiswa $mahasiswa)
{
    //
}
```

Instead of:

```php
public function edit($id)
{
    $mahasiswa = Mahasiswa::findOrFail($id);
}
```

---

# 24. Mass Assignment

Define `$fillable` or `$guarded` appropriately.

Example:

```php
protected $fillable = [
    'nim',
    'nama',
    'prodi',
    'angkatan',
];
```

Never blindly use:

```php
protected $guarded = [];
```

for sensitive models unless there is a clear reason.

---

# 25. Validation and Security

Always validate:

* Form input
* Route parameters where necessary
* QR tokens
* Attendance requests

Use CSRF protection for forms.

Blade forms should use:

```blade
@csrf
```

For destructive operations:

```blade
@method('DELETE')
```

Never expose passwords or sensitive tokens in views.

---

# 26. Error Handling

User-facing errors should be clear and understandable.

Example:

```text
QR Code tidak ditemukan.
```

```text
QR Code sudah tidak berlaku.
```

```text
Anda tidak terdaftar pada kelas ini.
```

```text
Anda sudah melakukan presensi untuk pertemuan ini.
```

Do not expose stack traces or internal database errors to users in production.

---

# 27. Flash Messages

Use session flash messages for CRUD operations.

Example:

```php
return redirect()
    ->route('admin.mahasiswa.index')
    ->with('success', 'Data mahasiswa berhasil disimpan.');
```

Blade should have a reusable alert component.

---

# 28. Naming Convention

Use Laravel conventions.

### Models

Singular PascalCase:

```text
User
Mahasiswa
MataKuliah
Kelas
Pertemuan
Presensi
```

### Tables

Plural snake_case:

```text
users
mahasiswas
matakuliahs
kelas
kelas_mahasiswa
pertemuans
presensis
```

### Controllers

```text
MahasiswaController
MataKuliahController
KelasController
PertemuanController
PresensiController
```

### Views

Use lowercase kebab/snake conventions consistently.

---

# 29. Indonesian UI Language

The application's user-facing UI should use Indonesian.

Examples:

```text
Dashboard
Mahasiswa
Mata Kuliah
Kelas
Pertemuan
Presensi
Kehadiran
Simpan
Batal
Edit
Hapus
Detail
Scan QR
Berhasil
Gagal
```

Code identifiers may remain in English or follow the established project convention.

Do not mix Indonesian and English unnecessarily in user-facing UI.

---

# 30. QR Scanner Frontend

The QR scanner may use a suitable JavaScript QR scanning library.

However:

* Keep the application Blade-based.
* Do not introduce SPA architecture.
* Load scanner JavaScript only on pages that require it.
* Stop the camera after successful scanning.
* Handle camera permission denial gracefully.
* Provide a fallback/error message.

Do not assume camera access is always available.

---

# 31. Attendance Endpoint

The endpoint should receive only the QR token necessary to identify the meeting.

Conceptually:

```php
Route::post('/presensi/scan', [PresensiController::class, 'scan'])
    ->middleware(['auth', 'role:mahasiswa'])
    ->name('mahasiswa.presensi.scan');
```

The server determines the authenticated student from the logged-in user.

Do not trust:

```text
mahasiswa_id
kelas_id
pertemuan_id
```

from the client when they can be derived from the QR token and authenticated user.

---

# 32. Duplicate Attendance

A student cannot submit multiple attendance records for the same meeting.

Always protect this at database level:

```php
$table->unique([
    'pertemuan_id',
    'mahasiswa_id',
]);
```

The UI should display a friendly message when duplicate attendance is attempted.

---

# 33. Testing

Every significant feature should have automated tests.

Important test cases:

### Authentication

* Admin can login
* Mahasiswa can login
* Invalid credentials are rejected

### Authorization

* Admin cannot access mahasiswa-only routes
* Mahasiswa cannot access admin routes
* Unauthenticated users are redirected to login

### Attendance

* Valid QR creates attendance
* Invalid QR is rejected
* Expired QR is rejected
* Inactive QR is rejected
* Student not enrolled in class is rejected
* Duplicate attendance is rejected
* Attendance belongs to authenticated student

### Data integrity

* Duplicate NIM is rejected
* Duplicate class enrollment is rejected
* Duplicate meeting number in the same class is rejected

---

# 34. Factories and Seeders

Create realistic seed data for development.

Example:

```text
1 Admin
20 Mahasiswa
5 Mata Kuliah
5 Kelas
Multiple class enrollments
Multiple Pertemuan
Attendance records
```

Use Laravel factories where practical.

Never hardcode production credentials in seeders.

If a development admin account is seeded, clearly mark it as development-only.

---

# 35. Artisan Commands

Prefer Artisan generators when creating Laravel files.

Examples:

```bash
php artisan make:model Mahasiswa -m
php artisan make:model MataKuliah -m
php artisan make:model Kelas -m
php artisan make:model Pertemuan -m
php artisan make:model Presensi -m
```

Controllers:

```bash
php artisan make:controller Admin/MahasiswaController --resource
php artisan make:controller Admin/MataKuliahController --resource
php artisan make:controller Admin/KelasController --resource
php artisan make:controller Admin/PertemuanController --resource
php artisan make:controller Mahasiswa/PresensiController
```

Requests:

```bash
php artisan make:request StoreMahasiswaRequest
```

Do not manually create files if an official Laravel generator is appropriate.

---

# 36. Migration Rules

Migration files must:

* Use foreign keys
* Use appropriate indexes
* Use unique constraints where required
* Define `onDelete` behavior explicitly when appropriate
* Avoid redundant columns
* Preserve referential integrity

Example:

```php
$table->foreignId('kelas_id')
    ->constrained('kelas')
    ->cascadeOnDelete();
```

Do not use raw SQL unless there is a clear technical requirement.

---

# 37. Git Rules

Do not modify unrelated files.

Before changing code:

1. Inspect the existing implementation.
2. Identify the smallest set of files required.
3. Preserve existing project conventions.
4. Make focused changes.
5. Run relevant tests.

Do not rewrite large parts of the application unnecessarily.

---

# 38. Code Quality

Prioritize:

* Readability
* Maintainability
* Security
* Laravel conventions
* DRY where appropriate
* Small focused methods
* Clear naming
* Server-side validation
* Proper authorization

Avoid:

* Huge controllers
* Duplicate Blade markup
* Duplicate business logic
* Hardcoded IDs
* Hardcoded credentials
* Trusting client-side validation
* Unnecessary abstractions
* Unnecessary dependencies

---

# 39. Before Completing a Task

The coding agent should verify:

```text
[ ] Existing code was inspected
[ ] Database relationships are correct
[ ] Authorization is enforced
[ ] Validation is implemented
[ ] CSRF protection is present
[ ] No sensitive data is exposed
[ ] QR attendance cannot be duplicated
[ ] Student can only attend registered classes
[ ] Blade UI is responsive
[ ] Tailwind CSS is used
[ ] N+1 queries are avoided
[ ] Relevant tests pass
[ ] No unrelated files were modified
```

---

# 40. Expected Development Architecture

Preferred architecture:

```text
Laravel 12
│
├── Authentication
│   └── users.role
│
├── Admin
│   ├── Dashboard
│   ├── Mahasiswa
│   ├── Mata Kuliah
│   ├── Kelas
│   ├── Pertemuan
│   └── Presensi
│
├── Mahasiswa
│   ├── Dashboard
│   ├── Scan QR
│   ├── Riwayat Presensi
│   └── Profile
│
├── Blade
│   ├── Layouts
│   ├── Components
│   ├── Admin Views
│   └── Mahasiswa Views
│
├── Tailwind CSS
│
└── MySQL
    ├── users
    ├── mahasiswas
    ├── matakuliahs
    ├── kelas
    ├── kelas_mahasiswa
    ├── pertemuans
    └── presensis
```

---

# 41. Important Principle

The application is an **attendance management system**, not merely a QR Code generator.

QR Code is only one part of the attendance workflow.

The backend must always determine whether an attendance submission is valid based on:

```text
Authenticated User
        +
Student Identity
        +
Class Enrollment
        +
Meeting
        +
QR Token
        +
QR Validity
        +
Duplicate Check
```

Only after all checks pass should attendance be recorded.

---

# 42. Agent Behavior

When implementing a feature:

1. Understand the existing project structure.
2. Check existing migrations/models/controllers/components before creating new ones.
3. Reuse existing components when possible.
4. Follow Laravel 12 conventions.
5. Use Blade + Tailwind CSS.
6. Keep role authorization server-side.
7. Protect attendance data against manipulation.
8. Prefer database constraints for data integrity.
9. Write tests for important business rules.
10. Do not introduce unnecessary packages.
11. Do not modify unrelated functionality.
12. Explain important architectural decisions briefly when completing a task.

When requirements are ambiguous, prefer the simplest implementation consistent with the existing application architecture.
