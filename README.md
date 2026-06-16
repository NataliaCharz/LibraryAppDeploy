# Library App — OAuth 2.0 z Keycloak

Aplikacja webowa do zarządzania biblioteką osobistą, zabezpieczona standardem **OAuth 2.0 + PKCE** z wykorzystaniem Keycloak jako Authorization Servera.

---

## Spełnione wymagania projektowe

| Wymaganie | Realizacja |
|-----------|------------|
| Backend zabezpieczony OAuth 2.0 | Spring Boot jako Resource Server — waliduje JWT wystawione przez Keycloak |
| Min. 1 endpoint z rolami | `/api/admin/**` — tylko `ADMIN` &nbsp;/&nbsp; `/api/user/**` — tylko `USER` |
| Min. 4 zabezpieczone endpointy | `/api/books/**`, `/api/authors/**`, `/api/user/**`, `/api/admin/**`, `/api/contact/**` |
| Min. 1 niezabezpieczony endpoint | `/actuator/health`, `/api/library/**`, `/v3/api-docs/**` |
| Frontend korzystający z backendu | Next.js — komunikacja przez REST z Bearer tokenem |
| Baza danych | PostgreSQL 16 |
| Skonfigurowany Authorization Server | Keycloak 26, realm `bookcase` |
| PKCE włączone | S256, Authorization Code Flow |

---

## Wymagania wstępne

- **Docker** i **Docker Compose** v2+
- Wolne porty: `80`, `8081`

---

## Uruchomienie

```bash
git clone <adres-repo>
cd LibraryAppDeploy
docker compose up -d
```

Pierwsze uruchomienie pobiera obrazy i zajmuje ~2–3 minuty. Keycloak potrzebuje ~60 sekund na pełny start.

### Sprawdzenie stanu kontenerów

```bash
docker compose ps
```

Oczekiwany wynik — wszystkie serwisy `running`:

```
NAME        STATUS
keycloak    running
postgres    running (healthy)
backend     running (healthy)
frontend    running
nginx       running
redis       running (healthy)
```

---

## Adresy

| Serwis | URL |
|--------|-----|
| **Aplikacja (frontend)** | http://localhost |
| **Keycloak Admin Console** | http://localhost:8081 |
| **Swagger UI** | http://localhost/swagger-ui/index.html |
| **Health check** | http://localhost/actuator/health |

---

## Konta testowe

### Aplikacja

| Użytkownik | Hasło | Rola |
|------------|-------|------|
| `admin` | `admin` | ADMIN + USER |
| `user1` | `user1` | USER |

> Konta są automatycznie importowane z `keycloak/bookcase-realm.json` przy każdym świeżym starcie.

### Keycloak Admin Console

| Login | Hasło |
|-------|-------|
| `admin` | `admin` |

---

## Jak sprawdzić projekt

### 1. Logowanie — weryfikacja PKCE w przeglądarce

1. Otwórz http://localhost
2. Otwórz DevTools → zakładka **Network**
3. Kliknij **Login** w menu bocznym
4. Obserwuj pierwszy request do Keycloak — zawiera:
   - `response_type=code` — Authorization Code Flow
   - `code_challenge=<hash>` — wygenerowany przez frontend
   - `code_challenge_method=S256` — SHA-256
5. Po wpisaniu hasła na stronie Keycloak następuje redirect do `/callback?code=<jednorazowy_kod>`
6. Ostatni request to wymiana kodu na token — zawiera `code_verifier` (oryginalny losowy ciąg)
7. Zaloguj się jako `user1` / `user1`

### 2. Weryfikacja tokena JWT

Po zalogowaniu otwórz DevTools → **Application** → **Local Storage** → `http://localhost`:

- Skopiuj wartość klucza `token`
- Wklej na https://jwt.io

W sekcji **Payload** widoczne m.in.:

```json
{
  "preferred_username": "user1",
  "realm_access": {
    "roles": ["USER", "offline_access", "uma_authorization"]
  },
  "iss": "http://keycloak:8080/realms/bookcase"
}
```

### 3. Weryfikacja zabezpieczenia endpointów przez curl

#### Niezabezpieczone — 200 bez tokena:

```bash
curl -s http://localhost/actuator/health
curl -s "http://localhost/api/library/author/Tolkien"
```

#### Zabezpieczone — 401 bez tokena:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost/api/books/
# → 401

curl -s -o /dev/null -w "%{http_code}" http://localhost/api/user/books
# → 401
```

#### Rola USER — dostęp do `/api/books/`, brak do `/api/admin/`:

```bash
TOKEN=$(curl -s -X POST "http://localhost:8081/realms/bookcase/protocol/openid-connect/token" \
  -d "grant_type=password&client_id=bookcase-app&username=user1&password=user1" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" http://localhost/api/books/
# → 200

curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" http://localhost/api/admin/
# → 403
```

#### Rola ADMIN — dostęp do `/api/admin/`:

```bash
ADMIN_TOKEN=$(curl -s -X POST "http://localhost:8081/realms/bookcase/protocol/openid-connect/token" \
  -d "grant_type=password&client_id=bookcase-app&username=admin&password=admin" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $ADMIN_TOKEN" http://localhost/api/admin/
# → 200
```

### 4. Konfiguracja Keycloak

Otwórz http://localhost:8081 → zaloguj jako `admin`/`admin` → przejdź do realm `bookcase`:

- **Clients** → `bookcase-app` → zakładka **Settings**: `publicClient: true`, `Standard flow: ON`
- **Clients** → `bookcase-app` → zakładka **Advanced**: `PKCE Code Challenge Method: S256`
- **Realm roles**: widoczne role `USER` i `ADMIN`

### 5. Swagger UI

http://localhost/swagger-ui/index.html — dostępny bez logowania (niezabezpieczony endpoint).

---

## Architektura

```
Przeglądarka
     │
     ▼
  Nginx :80  ─── reverse proxy ──────────────────────
  │                                                   │
  │  /          → frontend:3000 (Next.js)             │
  │  /api/      → backend:8080 (Spring Boot)          │
  └───────────────────────────────────────────────────┘
                       │
          ┌────────────┼───────────┐
          ▼            ▼           ▼
     PostgreSQL      Redis     Keycloak :8081
     (dane app)     (cache)    Authorization Server
                               realm: bookcase
                               client: bookcase-app
                               PKCE S256
```

---

## Jak działa PKCE

PKCE (Proof Key for Code Exchange) to rozszerzenie OAuth 2.0 dla aplikacji, które nie mogą bezpiecznie przechowywać client secret (np. SPA, aplikacje mobilne).

```
Frontend (przeglądarka)              Keycloak                Backend
        │                               │                        │
        │ 1. Generuje code_verifier     │                        │
        │    (64 losowe bajty)          │                        │
        │                               │                        │
        │ 2. code_challenge =           │                        │
        │    BASE64URL(SHA256(verifier))│                        │
        │                               │                        │
        │──── 3. GET /auth?             │                        │
        │    response_type=code         │                        │
        │    code_challenge=<hash> ────>│                        │
        │    code_challenge_method=S256 │                        │
        │                               │                        │
        │<─── 4. Strona logowania ──────│                        │
        │                               │                        │
        │──── 5. Login/hasło ──────────>│                        │
        │                               │                        │
        │<─── 6. redirect /callback     │                        │
        │         ?code=<jednorazowy> ──│                        │
        │                               │                        │
        │──── 7. POST /token            │                        │
        │    code=<kod>                 │                        │
        │    code_verifier=<oryginał> ─>│                        │
        │    (Keycloak weryfikuje:       │                        │
        │     SHA256(verifier)==hash)    │                        │
        │<─── 8. access_token ──────────│                        │
        │                               │                        │
        │──────────── 9. GET /api/books/ + Bearer token ────────>│
        │                               │   (walidacja podpisu   │
        │<──────────────────────────────│────JWT lokalnie) ──────│
```

**Dlaczego PKCE chroni:** atakujący może przechwycić `code` z URL (przez logi, referrer header). Bez PKCE mógłby go wymienić na token. Z PKCE potrzebuje `code_verifier` — losowego ciągu wygenerowanego w przeglądarce użytkownika, który **nigdy nie jest przesyłany przez URL**. Przechwycony `code` bez `code_verifier` jest bezużyteczny.

---

## Reset aplikacji

```bash
# Zatrzymanie — dane zostają w volumes
docker compose down

# Pełny reset — usuwa volumes (PostgreSQL, Keycloak, Redis)
docker compose down -v

# Następny start automatycznie odtworzy Keycloak z keycloak/bookcase-realm.json
docker compose up -d
```

---

## Struktura repozytorium

```
LibraryAppDeploy/
├── docker-compose.yaml          # Orchestracja serwisów
├── nginx/
│   └── nginx.conf               # Reverse proxy, wsparcie SSE
├── keycloak/
│   └── bookcase-realm.json      # Konfiguracja realm (auto-import przy starcie)
├── secrets/
│   ├── db_user.txt              # Login PostgreSQL
│   └── db_password.txt          # Hasło PostgreSQL
└── README.md
```

---

## Stack technologiczny

| Warstwa | Technologia | Wersja |
|---------|-------------|--------|
| Authorization Server | Keycloak | 26 |
| Backend | Spring Boot + Spring Security OAuth2 Resource Server | 3.5 / Java 21 |
| Frontend | Next.js + React | 16 / 19 |
| Baza danych | PostgreSQL | 16 |
| Cache | Redis | 7 |
| Reverse Proxy | Nginx | Alpine |
| Konteneryzacja | Docker Compose | v2 |
