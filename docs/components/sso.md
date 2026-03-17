# SSO & Authentication

Med-SEAL Suite includes a **single sign-on (SSO) system** that provides unified authentication across all services, with automatic user synchronisation to OpenEMR.

## Role in Med-SEAL

The SSO system ensures that:

- Users have **one set of credentials** across the platform
- User accounts are **automatically synced** to OpenEMR
- **Department/facility assignments** are managed centrally
- An **admin panel** allows user management (CRUD + role assignment)

## Architecture

```
┌────────────────────┐      ┌─────────────────┐
│   AI Frontend      │─────▶│   AI Service     │
│   (Admin Panel)    │      │   /api/auth/*    │
└────────────────────┘      └────────┬────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
           ┌──────────────┐  ┌─────────────┐  ┌────────────┐
           │  SSO DB       │  │  OpenEMR DB  │  │  Medplum   │
           │  (Postgres)   │  │  (MariaDB)   │  │  (FHIR)    │
           └──────────────┘  └─────────────┘  └────────────┘
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
