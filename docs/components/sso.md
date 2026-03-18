# SSO & Authentication

Med-SEAL Suite includes a **single sign-on (SSO) system** that provides unified authentication across all services, with automatic user synchronisation to OpenEMR.

## Role in Med-SEAL

The SSO system ensures that:

- Users have **one set of credentials** across the platform
- User accounts are **automatically synced** to OpenEMR
- **Department/facility assignments** are managed centrally
- An **admin panel** allows user management (CRUD + role assignment)

## Architecture

```{uml}
@startuml
skinparam componentStyle uml2

component "AI Frontend\n(Admin Panel)" as AIUI
component "AI Service\n/api/auth/*" as AISvc

database "SSO DB\n(PostgreSQL)" as SSODB
database "OpenEMR DB\n(MariaDB)" as OEDB
database "Medplum\n(FHIR)" as FHIR

AIUI --> AISvc
AISvc --> SSODB
AISvc --> OEDB : user sync
AISvc --> FHIR : Practitioner sync
@enduml
```

### Entity-Relationship Diagram

```{uml}
@startuml
skinparam linetype ortho
hide circle

entity "sso.users" as SSOUser {
  * id : SERIAL <<PK>>
  --
  * username : VARCHAR(64) <<UNIQUE>>
  * password_hash : VARCHAR(255)
  * role : VARCHAR(32)
  full_name : VARCHAR(128)
  email : VARCHAR(128)
  facility_id : INTEGER <<FK>>
  created_at : TIMESTAMP
  updated_at : TIMESTAMP
}

entity "openemr.users" as OEUser {
  * id : BIGINT <<PK>>
  --
  * username : VARCHAR(255) <<UNIQUE>>
  * password : VARCHAR(255)
  abook_type : VARCHAR(30)
  authorized : TINYINT
  facility_id : INT
}

entity "openemr.users_secure" as OESecure {
  * id : BIGINT <<PK>>
  --
  * username : VARCHAR(255) <<UNIQUE>>
  * password : VARCHAR(255)
  last_update_password : DATETIME
}

entity "openemr.facility" as Facility {
  * id : INT <<PK>>
  --
  * name : VARCHAR(255)
  phone : VARCHAR(30)
  street : VARCHAR(255)
  city : VARCHAR(255)
}

SSOUser ||--o{ OEUser : "sync (username)"
SSOUser ||--o{ OESecure : "sync (password)"
SSOUser }o--|| Facility : "facility_id"
OEUser }o--|| Facility : "facility_id"
@enduml
```

### Authentication Flow (Sequence Diagram)

```{uml}
@startuml
actor "User" as User
participant "Client App" as Client
participant "AI Service\n(/api/auth)" as Auth
database "SSO DB" as SSODB
database "OpenEMR DB" as OEDB

User -> Client : Enter credentials
Client -> Auth : POST /api/auth/login\n{username, password}
Auth -> SSODB : SELECT user\nWHERE username = ?
SSODB --> Auth : user record

alt Password matches (bcrypt verify)
  Auth -> Auth : Generate session token
  Auth -> OEDB : Sync user record\n(if changed)
  OEDB --> Auth : OK
  Auth --> Client : 200 {token, user, role}
  Client --> User : Login success
else Password mismatch
  Auth --> Client : 401 Unauthorized
  Client --> User : Invalid credentials
end

note over Auth, SSODB
  Session tokens expire after 60 min.
  Failed attempts are logged as
  AuditEvent resources.
end note
@enduml
```

## SSO Database

The SSO system uses a dedicated PostgreSQL database:

| Property | Value |
|---|---|
| Container | `medseal-sso-db` |
| Database | `medseal_sso` |
| User | `sso` |
| Port | `5434` |

### Schema

The core `users` table stores:

| Column | Description |
|---|---|
| `id` | Auto-incrementing primary key |
| `username` | Unique login name |
| `password_hash` | bcrypt-hashed password |
| `role` | User role (`admin`, `clinician`, `nurse`, etc.) |
| `full_name` | Display name |
| `email` | Email address |
| `facility_id` | Department/facility assignment |
| `created_at` | Account creation timestamp |
| `updated_at` | Last modification timestamp |

## OpenEMR Sync

When a user is created or updated in the SSO system, their account is **automatically synchronised** to OpenEMR's `users_secure` and `users` tables via direct MariaDB queries.

The sync maintains:
- Username and password alignment
- Role mapping (SSO role → OpenEMR user type)
- Facility/department assignment

### Sync Configuration

Environment variables for OpenEMR database connectivity:

```bash
OPENEMR_DB_HOST=openemr-db
OPENEMR_DB_PORT=3306
OPENEMR_DB_USER=openemr
OPENEMR_DB_PASS=openemr
OPENEMR_DB_NAME=openemr
```

## API Endpoints

The AI Service exposes authentication endpoints:

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/login` | Authenticate user, return session |
| `POST` | `/api/auth/register` | Create new user account |
| `GET` | `/api/auth/users` | List all users (admin only) |
| `PUT` | `/api/auth/users/:id` | Update user details |
| `DELETE` | `/api/auth/users/:id` | Delete user account |
| `GET` | `/api/facilities` | List available departments/facilities |

## Admin Panel

The AI Frontend includes an admin page for user management:

- Create, edit, and delete user accounts
- Assign roles and departments
- View user list with search and filter
- Password reset

## Security Notes

```{warning}
The default SSO configuration uses simple credentials suitable for **development only**. For production:
- Use strong, unique passwords
- Enable HTTPS on all endpoints
- Implement rate limiting on auth endpoints
- Add session expiry and refresh token rotation
- Consider integrating with an external identity provider (OIDC/SAML)
```
