# Library App — OAuth 2.0 z Keycloak

**Autor:** Natalia Charzyńska  
**Grupa:** 3

Aplikacja webowa do zarządzania biblioteką osobistą, zabezpieczona standardem **OAuth 2.0 + PKCE** z wykorzystaniem Keycloak jako Authorization Servera.

---

## Wymagania projektowe

| Wymaganie | Realizacja |
|-----------|------------|
| Backend zabezpieczony OAuth 2.0 | Spring Boot jako Resource Server — waliduje JWT wystawione przez Keycloak |
| Min. 1 endpoint uwzględniający role | `/api/contact/send` — tylko `USER` / `/api/contact/**` — tylko `ADMIN` |
| Min. 4 zabezpieczone endpointy | `/api/books/**`, `/api/authors/**`, `/api/user/**`, `/api/contact/**` |
| Min. 1 niezabezpieczony endpoint | `/actuator/health`, `/api/library/**`, `/v3/api-docs/**` |
| Frontend korzystający z backendu | Next.js — komunikacja przez REST z Bearer tokenem w nagłówku |
| Baza danych | PostgreSQL 16 |
| Skonfigurowany Authorization Server | Keycloak 26, realm `bookcase`, klient `bookcase-app` |
| PKCE włączone | S256, Authorization Code Flow |

---

## Funkcjonalności aplikacji

### Dostępne bez logowania

| Funkcja | Opis |
|---------|------|
| Przeglądanie biblioteki | Wyszukiwanie książek po nazwisku autora przez zewnętrzne API OpenLibrary |
| Rejestracja | Przekierowanie na stronę rejestracji Keycloak — tworzy konto z rolą USER |
| Logowanie | Przekierowanie na stronę logowania Keycloak (OAuth 2.0 + PKCE S256) |

### Panel użytkownika (rola USER)

| Funkcja | Opis |
|---------|------|
| Przeglądanie książek | Lista wszystkich książek w systemie z wyszukiwarką po tytule |
| Lista lektur | Dodawanie/usuwanie książek do osobistej listy przeczytanych |
| Lista życzeń | Dodawanie/usuwanie książek do wishlisty |
| Przeglądanie autorów | Lista wszystkich autorów z ich szczegółami |
| Szczegóły książki | Podgląd informacji o wybranej książce |
| Kontakt z adminem | Formularz wysyłania wiadomości do administratora (imię, e-mail, treść) |

### Panel administratora (rola ADMIN)

| Funkcja | Opis |
|---------|------|
| Dodawanie książek | Formularz dodawania nowej książki do systemu |
| Dodawanie autorów | Formularz dodawania nowego autora |
| Przeglądanie książek i autorów | Dostęp do pełnej listy |
| Wiadomości od użytkowników | Podgląd i usuwanie wiadomości kontaktowych |
| Powiadomienia SSE | Powiadomienia w czasie rzeczywistym o nowych wiadomościach (Server-Sent Events) |

---

## Uruchomienie

```bash
git clone https://github.com/NataliaCharz/LibraryAppDeploy.git
cd LibraryAppDeploy
cp .env.example .env          # ustaw własne hasło KEYCLOAK_ADMIN_PASSWORD
docker compose up -d
```

Keycloak automatycznie importuje realm `bookcase` z pliku `keycloak/bookcase-realm.json` (konta testowe, role, konfiguracja PKCE).

## Adresy

| Serwis | URL |
|--------|-----|
| **Aplikacja (frontend)** | http://localhost |
| **Keycloak Admin Console** | http://localhost:8081 |

---

## Konta testowe

### Aplikacja

| Użytkownik | Hasło | Rola |
|------------|-------|------|
| `admin` | `admin` | ADMIN + USER |
| `user1` | `user1` | USER |

> Konta są automatycznie importowane z `keycloak/bookcase-realm.json` przy każdym świeżym starcie.

### Keycloak Admin Console

Credentiale są pobierane z pliku `.env` (nie commitowanego do repozytorium).  
Skopiuj `.env.example` jako `.env` i ustaw własne wartości przed uruchomieniem:

```bash
cp .env.example .env
```

| Zmienna | Domyślna wartość w `.env.example` |
|---------|----------------------------------|
| `KEYCLOAK_ADMIN` | `admin` |
| `KEYCLOAK_ADMIN_PASSWORD` | `change_me` |

---

## Diagram komunikacji w systemie

```
Przeglądarka (localhost)
        │
        ▼
   Nginx :80  ──────────── reverse proxy ─────────────────────
        │                                                      │
        │  /                 frontend:3000  (Next.js)          │
        │  /api/*            backend:8080   (Spring Boot)      │
        └──────────────────────────────────────────────────────┘
                                │
               ┌────────────────┼──────────────┐
               ▼                ▼              ▼
          PostgreSQL           Redis        Keycloak :8081
          (dane aplikacji)    (cache)       Authorization Server
                                            realm: bookcase
                                            client: bookcase-app
                                            PKCE S256

Przepływ autoryzacji (OAuth 2.0 Authorization Code + PKCE):

  Przeglądarka                Keycloak               Backend
       │                         │                      │
       │─── 1. Klik Login ──────>│                      │
       │    (z code_challenge)   │                      │
       │<── 2. Formularz logowania                      │
       │─── 3. Login/hasło ─────>│                      │
       │<── 4. /callback?code=X ─│                      │
       │─── 5. POST /token ─────>│                      │
       │    (z code_verifier)    │                      │
       │<── 6. access_token ─────│                      │
       │                         │                      │
       │─── 7. GET /api/books ───────────────────────>  │
       │       Bearer: <JWT>     │     walidacja JWT    │
       │<── 8. dane ─────────────────────────────────── │
```

---

## Jak działa PKCE

PKCE (Proof Key for Code Exchange) to rozszerzenie OAuth 2.0 dla aplikacji, które nie mogą bezpiecznie przechowywać client secret (np. SPA, aplikacje mobilne).

Bez niego przechwycenie kodu autoryzacyjnego wystarczyłoby do uzyskania tokenu. PKCE wiąże kod z konkretną sesją przeglądarki przez kryptograficzny dowód.

**Przebieg:**

1. Frontend generuje losowy `code_verifier` (64 bajty)
2. Oblicza `code_challenge = BASE64URL(SHA256(code_verifier))`
3. Wysyła `code_challenge` do Keycloak przy żądaniu autoryzacji
4. Keycloak zapamiętuje `code_challenge` i wydaje jednorazowy kod
5. Frontend wysyła kod + oryginalny `code_verifier` po token
6. Keycloak weryfikuje: `SHA256(verifier) == challenge` → wydaje `access_token`

Nawet jeśli ktoś przechwyci kod autoryzacyjny, bez `code_verifier` (który nigdy nie opuszcza przeglądarki) nie może wymienić go na token.

**Implementacja w projekcie:** `app/auth/pkce.js` — funkcje `generateCodeVerifier()`, `generateCodeChallenge()`, `buildAuthUrl()`.

---

## Spring Security — konfiguracja Resource Server

Backend (`SecurityConfig.java`) działa jako OAuth 2.0 Resource Server. Nie wystawia tokenów — tylko je weryfikuje. Walidacja JWT odbywa się przez pobranie kluczy publicznych Keycloak z endpointu JWK:

```yaml
spring.security.oauth2.resourceserver.jwt.jwk-set-uri:
  http://keycloak:8080/realms/bookcase/protocol/openid-connect/certs
```

Role użytkownika są wyodrębniane z claimu `realm_access.roles` w JWT i mapowane na Spring Security authorities (`ADMIN`, `USER`).

---

## Struktura repozytorium

```
LibraryAppDeploy/
├── docker-compose.yaml          # Orchestracja serwisów
├── .env.example                 # Szablon zmiennych środowiskowych (commitowany)
├── .env                         # Lokalne credentiale (NIE commitowany, w .gitignore)
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
| Frontend | Next.js + React | 15 / 19 |
| Baza danych | PostgreSQL | 16 |
| Cache | Redis | 7 |
| Reverse Proxy | Nginx | Alpine |
| Konteneryzacja | Docker Compose | v2 |
