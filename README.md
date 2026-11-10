# Employee Management System (EMS)

EMS is an employee management application with an Angular frontend and a Spring Boot REST backend. It supports JWT login, role-based permissions, employee management, user listings, table search/filtering, and onboarding requests. Onboarding uses a separate OR service that is not included in this repository.

## Repository structure

The root repository tracks `backend/` and `frontend/` as Git submodules.

```text
EMS-Mono/
├── README.md
├── .gitmodules                         # Backend and frontend repository links
├── backend/                            # Java 17 / Spring Boot 4.0.6 / Maven
│   ├── pom.xml                         # Dependencies and build configuration
│   ├── mvnw, mvnw.cmd, .mvn/            # Maven wrapper
│   ├── src/main/java/org/example/ems/
│   │   ├── EmsApplication.java         # Application entry point
│   │   ├── controller/                 # Auth, employee, user, onboarding APIs
│   │   ├── service/                    # Business logic and approval processing
│   │   ├── repository/                 # Spring Data JPA database access
│   │   ├── model/                      # Employee, User, Role, Permission entities
│   │   ├── dto/                        # API request and response objects
│   │   ├── security/                   # JWT, BCrypt, authorization, CORS
│   │   ├── config/                     # Role/demo seeding and HTTP client setup
│   │   └── client/                     # HTTP integration with the OR service
│   ├── src/main/resources/
│   │   └── application.properties      # Database, JWT, port, CORS, OR settings
│   ├── src/test/java/                  # Backend regression tests
│   ├── db/approval-idempotency.sql     # Migration for externally managed schemas
│   └── Dockerfile, Buildfile, Procfile # Deployment support
└── frontend/                           # Angular 21 / TypeScript / PrimeNG
    ├── package.json                    # Dependencies and npm commands
    ├── angular.json                    # Angular build and test configuration
    ├── public/config.json              # Runtime backend API URL
    └── src/
        ├── main.ts                     # Frontend entry point
        ├── styles.css                  # Global styles
        └── app/
            ├── app.config.ts           # Application providers and initialization
            ├── app.routes.ts           # Employee, login, and user routes
            ├── components/             # Login, employees, users, onboarding, filters
            ├── services/               # API calls, authentication, HTTP interceptor
            ├── models/                 # TypeScript data models
            ├── guards/                 # Login and protected-route navigation
            └── config/                 # Runtime configuration loader
```

The browser calls EMS; EMS accesses PostgreSQL and forwards onboarding requests to OR. The browser does not need the OR internal API key.

## Prerequisites

- Git with access to this repository and both submodule repositories.
- JDK 17 or newer, with `JAVA_HOME` set to your JDK installation.
- Node.js compatible with the checked-in Angular 21 packages: `^20.19.0`, `^22.12.0`, or `>=24.0.0`. Node.js 22.12+ within the 22.x release line is one supported option.
- npm; `frontend/package.json` declares npm `11.13.0`.
- A running PostgreSQL server and `psql` (or a database GUI).

Maven is downloaded by the included wrapper, so a separate Maven installation and a global Angular CLI installation are unnecessary. Initial dependency installation requires internet access.

## Starting the application

The commands below use PowerShell on Windows. Run them from the indicated directories. On macOS/Linux, use `./mvnw` instead of `.\mvnw.cmd` and `export NAME=value` instead of `$env:NAME = "value"`.

### 1. Clone the repository and its submodules

```powershell
git clone --recurse-submodules https://github.com/ShrutiSBaghel/EMS.git EMS-Mono
cd EMS-Mono
```

For an existing clone with empty submodule directories, run from the repository root:

```powershell
git submodule update --init --recursive
```

### 2. Create a local PostgreSQL database

Connect as your PostgreSQL administrator (replace `postgres` if necessary):

```powershell
psql -h localhost -U postgres -d postgres
```

Run the following SQL, replacing the example password with your own:

```sql
CREATE USER ems_user WITH PASSWORD 'replace-with-your-local-db-password';
CREATE DATABASE ems OWNER ems_user;
```

Exit `psql` with `\q`. The backend uses Hibernate `ddl-auto=update` to create/update tables on startup; the database itself must already exist.

### 3. Configure and start the backend

In a terminal at the repository root:

```powershell
cd backend
$env:DATABASE_URL = "jdbc:postgresql://localhost:5432/ems"
$env:DATABASE_USERNAME = "ems_user"
$env:DATABASE_PASSWORD = "replace-with-your-local-db-password"

# Generate a random local JWT signing secret (at least 32 UTF-8 bytes required).
$jwtBytes = New-Object byte[] 48
$jwtRng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
$jwtRng.GetBytes($jwtBytes)
$jwtRng.Dispose()
$env:JWT_KEY = [Convert]::ToBase64String($jwtBytes)

$env:PORT = "8080"
$env:CORS_ALLOWED_ORIGINS = "http://localhost:4200"

# For a fresh, disposable local database, enable demo accounts and employees.
$env:SPRING_PROFILES_ACTIVE = "demo"
$env:APP_DEMO_PASSWORD = "replace-with-your-local-demo-password"

.\mvnw.cmd spring-boot:run
```

Keep this terminal running. EMS listens at `http://localhost:8080`. Environment variables set here apply to this terminal and its child processes; set them again when starting from a new terminal. Spring Boot does not automatically load a root `.env` file in this setup. Reuse the same `JWT_KEY` between restarts if you want existing tokens to remain valid.

| Variable | Purpose | Default / requirement |
| --- | --- | --- |
| `DATABASE_URL` | PostgreSQL JDBC URL | Required; use the `jdbc:postgresql://` format |
| `DATABASE_USERNAME` | PostgreSQL login | Required |
| `DATABASE_PASSWORD` | PostgreSQL password | Required |
| `JWT_KEY` | JWT signing secret | Required; at least 32 UTF-8 bytes |
| `PORT` | EMS HTTP port | `8080` |
| `CORS_ALLOWED_ORIGINS` | Comma-separated browser origins | `http://localhost:4200` |
| `SPRING_PROFILES_ACTIVE` | Optional Spring profile | Set to `demo` for sample data |
| `APP_DEMO_PASSWORD` | Password for newly created demo users | Required when `demo` is active |
| `OR_SERVICE_BASE_URL` | Separate onboarding service URL | `http://localhost:8081` |
| `OR_SERVICE_INTERNAL_API_KEY` | Shared key for EMS-to-OR calls | Match the OR service configuration |

Normal startup creates missing roles only: `ADMIN`, `READ`, `WRITE`, `READ_WRITE`, and `DELETE`. The demo profile also creates missing demo users and inserts 1,000 employees only when the employee table is empty. It preserves existing demo users' passwords/roles and does not top up a partially populated employee table.

### 4. Configure and start the frontend

In a second terminal at the repository root:

```powershell
cd frontend
npm ci
```

Edit `frontend/public/config.json` to point to your local backend. The checked-in value currently targets a hosted backend, so this change is necessary for local development:

```json
{
  "apiBaseUrl": "http://localhost:8080"
}
```

Then start Angular from `frontend/`:

```powershell
npm start
```

Open [http://localhost:4200](http://localhost:4200). The app loads `public/config.json` before startup; use this file to configure API access. If you change the backend port, update both this URL and `PORT`. If you change the frontend origin, update `CORS_ALLOWED_ORIGINS` on the backend.

### 5. Log in and optionally create a local administrator

For the demo setup above, sign in with `editor.user` and the password you set in `APP_DEMO_PASSWORD`. This account can read, create, and update records.

| Demo username | Role |
| --- | --- |
| `reader.user` | `READ` |
| `viewer.user` | `READ` |
| `writer.user` | `WRITE` |
| `editor.user` | `READ_WRITE` |
| `delete.user` | `DELETE` |

No administrator is created automatically. Registration and onboarding approval/rejection require an `ADMIN` account. For a fresh disposable local demo database, you can promote the seeded `editor.user` after backend startup. Connect to the application database:

```powershell
psql -h localhost -U ems_user -d ems
```

```sql
UPDATE users
SET role_id = (SELECT id FROM roles WHERE name = 'ADMIN')
WHERE username = 'editor.user';
```

Confirm that the command reports `UPDATE 1`, exit with `\q`, and log in again. For a non-demo installation, provision the first administrator through controlled database setup with a BCrypt-encoded password and an `ADMIN` role; plain-text passwords will not authenticate. There is no public self-registration flow.

### 6. Enable onboarding (requires the separate OR service)

Employee/user functionality can run with PostgreSQL, EMS, and Angular alone. The onboarding tab requires the external OR service, whose source and database setup are not included here. Obtain that service and follow its own setup instructions, then set these variables in the backend terminal before starting/restarting EMS:

```powershell
$env:OR_SERVICE_BASE_URL = "http://localhost:8081"
$env:OR_SERVICE_INTERNAL_API_KEY = "replace-with-the-shared-or-service-key"
```

OR must accept the matching `X-Internal-Service-Key` header and implement the onboarding API, including `/onboarding-requests/{id}/prepare-approval` and `/onboarding-requests/{id}/complete`. Without a running, compatible OR service, onboarding requests fail. See `backend/README.md` for approval recovery behavior and schema migration details.

## Build and test

From `backend/`, with the required environment configured:

```powershell
.\mvnw.cmd test
.\mvnw.cmd clean package
```

The packaged backend is `backend/target/EMS-0.0.1-SNAPSHOT.jar`. Run it with the same backend environment variables:

```powershell
java -jar target/EMS-0.0.1-SNAPSHOT.jar
```

From `frontend/`:

```powershell
npm run build
npm run test:guards
npm run test:onboarding
npm run test:filters
```

The Angular build writes to `frontend/dist/`. `npm test` runs the standard test suite; the frontend's existing README documents an unrelated `employee.spec.ts` import error in that suite. The focused test commands use their own TypeScript configurations.

## Common setup issues

- **Empty `backend/` or `frontend/`:** initialize the Git submodules as shown above.
- **Database connection or authentication failure:** confirm PostgreSQL is running, the `ems` database exists, and the three `DATABASE_*` values match your local setup.
- **Missing configuration or JWT key errors:** set all required backend variables in the terminal that launches Java; use a JWT secret of at least 32 UTF-8 bytes.
- **Login fails on a fresh database:** normal startup creates no users. Use the explicit local demo setup or provision an administrator. Existing demo users keep their original passwords.
- **Requests go to the hosted API:** change `frontend/public/config.json` to the local backend URL and reload the page.
- **Browser CORS errors:** ensure `CORS_ALLOWED_ORIGINS` exactly matches the browser origin, including its port, then restart EMS.
- **HTTP 403:** the signed-in account lacks the role required for that action. Use a suitable demo account or provision an administrator.
- **Onboarding errors:** verify the separate OR service, shared internal key, URL, and compatible approval endpoints.

Stop each development server with `Ctrl+C` in its terminal.
