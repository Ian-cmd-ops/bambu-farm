# 🖨️ Bambu Farm — Bellingham Makerspace Fork

A web-based dashboard to monitor and manage multiple Bambu Lab printers (A1, P1, X1 series) using MQTT / FTP / RTSP.

**Forked from [TFyre/bambu-farm](https://github.com/TFyre/bambu-farm) v1.8.0**

---

> [!IMPORTANT]
> **Firmware Blockade:** Bambu Lab has started blocking printing via MQTT unless LAN Mode is enabled. If you cannot print, consider downgrading your firmware or enabling Cloud Mode in the config.

> [!WARNING]
> **X1C Compatibility:** FTPS connections for the X1C require SSL Session Reuse. Set `bambu.use-bouncy-castle=true` in your `.env` and use the JVM flag `-Djdk.tls.useExtendedMasterSecret=false`.

---

## What This Fork Adds

This fork adds a **database-backed user management system** so Makerspace admins can create, edit, and remove user accounts from the web UI — no SSH or service restarts required.

### Implemented Features

| Feature | Status | Description |
|---------|--------|-------------|
| User Management UI | ✅ Done | Admin-only Vaadin page at `/admin/users` for CRUD operations |
| H2 Database Storage | ✅ Done | Users stored in `bambu-users.mv.db` via Hibernate ORM Panache |
| Auto-Migration | ✅ Done | Existing `.env` users imported to DB on first boot |
| Bcrypt Password Hashing | ✅ Done | All passwords stored as bcrypt hashes, never plaintext |
| Password Policy | ✅ Done | Minimum 10 characters, requires uppercase, lowercase, digit, special character |
| Username Validation | ✅ Done | 3-32 chars, alphanumeric plus `.` `-` `_` only |
| Login Rate Limiting | ✅ Done | 5 failed attempts triggers 5-minute lockout per account |
| Audit Logging | ✅ Done | All auth events and user changes logged with `AUDIT:` / `SECURITY:` prefixes |
| Backward Compatibility | ✅ Done | All existing `.env` configurations continue to work unchanged |

### Not Implemented (Yet)

These features are **not present** in this fork despite what previous documentation may have claimed:

- ❌ H2 database encryption at rest (AES cipher)
- ❌ SSL/TLS termination (HTTPS on port 8443)
- ❌ `start-farm.sh` atomic launcher script
- ❌ Obico / AI failure detection integration
- ❌ SMS / Twilio notifications
- ❌ Double opt-in consent workflow
- ❌ Progressive tarpitting (exponential backoff) — current implementation uses flat lockout

---

## Quick Start

### Prerequisites

- **Java 21 LTS** (OpenJDK, Zulu, or Temurin)
- **Maven** (for building from source)

### Build

```bash
git clone https://github.com/BellinghamMakerspace/bambu-farm.git
cd bambu-farm
mvn clean install -Pproduction
```

### Configure

Create a `.env` file in the same directory as the JAR:

```properties
quarkus.http.host=0.0.0.0
quarkus.http.port=8080

# Admin account (will be auto-migrated to database on first boot)
bambu.users.admin.password=CHANGE_ME_TO_A_STRONG_PASSWORD
bambu.users.admin.role=admin

# Printers
bambu.printers.p1p.device-id=SERIAL_NUMBER
bambu.printers.p1p.access-code=ACCESS_CODE
bambu.printers.p1p.ip=192.168.1.XXX
bambu.printers.p1p.model=p1p
```

### Run

```bash
java -jar bambu-web-1.8.0-runner.jar
```

Open `http://YOUR_IP:8080` and log in. Admin users can access **User Management** from the navigation drawer.

### Alpine Linux / OpenRC Service

See the upstream [README.service.md](https://github.com/TFyre/bambu-farm/blob/main/README.service.md) for service setup. Key addition for this fork: ensure the init script includes `directory="/opt/bambu-farm"` so Quarkus can find the `.env` file.

---

## User Management

### Adding Users via Web UI

1. Log in as an admin
2. Open the hamburger menu → **User Management**
3. Click **Add User**
4. Set username, password (must meet policy), and role

### Password Policy

Passwords must meet **all** of the following:
- At least 10 characters
- At least one uppercase letter (A-Z)
- At least one lowercase letter (a-z)
- At least one digit (0-9)
- At least one special character

### Rate Limiting

After 5 failed login attempts, the account is locked for 5 minutes. This resets on successful login. Lockout state is tracked in memory (resets on service restart).

### Database Backup

User data is stored in `bambu-users.mv.db` in the working directory. Back this file up regularly. If you lose it, users will be re-migrated from `.env` on next boot (but any users created via the web UI will be lost).

---

## Configuration Reference

See the upstream [TFyre/bambu-farm README](https://github.com/TFyre/bambu-farm#readme) for the full configuration reference including printer options, AMS settings, cloud mode, and MQTT configuration.

### Additional Properties (This Fork)

These are added to `application.properties` inside the JAR and generally don't need to be changed:

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:file:./bambu-users;AUTO_RECONNECT=TRUE
quarkus.hibernate-orm.database.generation=update
```

---

## Troubleshooting

### "Port 8080 in use"

A previous instance is still running:

```bash
# Linux
sudo fuser -k 8080/tcp
# or
pkill -9 -f bambu-web
```

### Service starts but .env not loaded

Quarkus reads `.env` from the **current working directory** only. If running as a service, ensure the init script sets `directory="/opt/bambu-farm"` (OpenRC) or `WorkingDirectory=/opt/bambu-farm` (systemd).

### Users not showing in User Management

Users are migrated from `.env` to the database on first startup. If you added users to `.env` after the initial boot, they won't appear until you either restart the service or use the web UI to create them.

---

## Architecture

```
Login Request
    → DatabaseIdentityProvider (checks H2 database)
        → Success? → Authenticated
        → Fail? → throws AuthenticationFailedException
    → TFyreIdentityProvider (checks .env users)
        → Success? → Authenticated  
        → Fail? → 401 Unauthorized
```

### New Files (vs upstream)

```
bambu/src/main/java/com/tfyre/bambu/user/
├── DatabaseIdentityProvider.java   # Quarkus IdentityProvider for DB auth
├── UserEntity.java                 # JPA entity
├── UserRepository.java             # Panache repository
└── UserService.java                # Business logic, password policy, rate limiting

bambu/src/main/java/com/tfyre/bambu/view/admin/
└── UserManagementView.java         # Vaadin admin UI
```

### Modified Files (vs upstream)

- `bambu/pom.xml` — added `quarkus-hibernate-orm-panache` and `quarkus-jdbc-h2` dependencies
- `bambu/src/main/resources/application.properties` — added H2 datasource config
- `bambu/src/main/java/com/tfyre/bambu/MainLayout.java` — added User Management to navigation

---

## Contributing

1. Fork this repository
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Test your changes locally (build + run + verify)
4. Open a Pull Request with a clear description of what you changed and why

---

## License

[Apache-2.0](LICENSE.txt) — same as the upstream project.

## Acknowledgments

- [TFyre/bambu-farm](https://github.com/TFyre/bambu-farm) — the original project
- Bellingham Makerspace members for testing and feedback
