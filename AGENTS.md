# Streama — repository notes

Streama is a Grails 3.2.6 / Groovy application (AngularJS frontend, REST backend).
It needs JDK 8: the checked-in Gradle wrapper is 3.1, which does not run on newer JDKs.

## Running it here

```bash
docker compose -f docker-compose.base44.yml up -d --build
```

- `web` runs `./gradlew bootRun -Dgrails.env=mysql` from the bind-mounted working tree
  (host port 3000 → container 8080). `db` is MariaDB.
- The `mysql` Grails environment in `grails-app/conf/application.yml` reads its connection
  from `MYSQL_HOST` / `MYSQL_PORT` / `MYSQL_DB` / `MYSQL_USER` / `MYSQL_PASSWORD`, which is
  why no application code or config had to be modified for this environment. Hibernate
  `dbCreate: update` creates the schema on first boot; `BootStrap` then seeds roles, the
  `admin` user and default settings.
- The first boot downloads Gradle 3.1 plus all Gradle/Maven dependencies into the
  `streama-gradle` volume; later boots take about a minute. The app is ready when the log
  prints `Grails application running at http://localhost:8080`.
- Healthcheck and `healthPath` both probe `/login/auth` (the login form) — `/` is protected
  and redirects there.

## Things that are not obvious

- Login: `/login/auth`, default credentials `admin` / `admin` (announced on the login screen).
  `/` requires authentication and redirects to the login page.
- Video uploads: the "Upload Directory" setting is empty in a fresh database and must be set
  (as admin, under Settings) to `/data/streama`, which is the persistent `streama-upload`
  volume. Uploads fail until it is set.
- External metadata uses TheMovieDB, and subtitles use OpenSubtitles. Both are stored as
  **database settings entered in the admin Settings page**, not environment variables, so the
  app boots and runs fully without them. Metadata/subtitle fetching stays empty until a key is
  entered there.
- Frontend assets (JS/CSS/translations under `grails-app/assets/`) are read live from the
  working tree — editing them shows up on a browser refresh. Groovy classes, controllers,
  services and domain classes are compiled at boot, so those changes need
  `docker compose -f docker-compose.base44.yml restart web`.
- `streama-remember-me.key` is generated into the working tree on first boot and is gitignored.

## Tests

`./gradlew test` (Grails Spock specs in `src/test/groovy/streama`) — run inside the container,
because the sandbox host has no JDK:

```bash
docker compose -f docker-compose.base44.yml exec -T web ./gradlew test
```
