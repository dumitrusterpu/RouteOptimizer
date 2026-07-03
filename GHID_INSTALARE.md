# Ghid de instalare a mediului de dezvoltare — RouteOptimizer

Acest ghid descrie pas cu pas cum să instalezi toate uneltele necesare și cum să
rulezi aplicația **RouteOptimizer** local, pe Windows. Aplicația are două componente:

| Componentă | Tehnologie | Port implicit |
|------------|------------|---------------|
| **RouteOptimizer.Web** (frontend) | React 19 + TypeScript + Vite 7 | `3000` |
| **RouteOptimizer.API** (backend) | ASP.NET Core (.NET 8) | `7135` (HTTPS) / `5253` (HTTP) |
| PostgreSQL + PostGIS | bază de date geospațială | `5432` |
| Redis | cache / Hangfire | `6379` |
| Keycloak | autentificare (opțional în dev) | `8080` |

---

## 1. Cerințe software (prezentare generală)

| Unealtă | Versiune recomandată | Necesară pentru |
|---------|----------------------|-----------------|
| Node.js | **20.19+** sau **22.12+** (LTS) | frontend (Vite 7) |
| .NET SDK | **8.0** | backend |
| PostgreSQL + PostGIS | **15** (imaginea `postgis/postgis:15-3.3`) | bază de date |
| Redis | **7** | cache & job-uri Hangfire |
| Docker Desktop | ultima versiune | rularea PostgreSQL + Redis (recomandat) |
| Git | ultima versiune | clonare / control versiune |
| Keycloak | **26** | autentificare (opțional local) |

> **Cel mai simplu drum:** instalează Node.js + .NET SDK + Docker Desktop, iar
> PostgreSQL și Redis le pornești prin `docker compose`. Nu trebuie să le
> instalezi manual.

---

## 2. Instalarea uneltelor de bază

### 2.1 Git
Descarcă și instalează de la <https://git-scm.com/download/win>.
Verificare:
```powershell
git --version
```

### 2.2 Node.js (frontend)
Instalează versiunea **LTS** de la <https://nodejs.org> (minim 20.19).

Recomandare: folosește **nvm-windows** ca să poți comuta ușor între versiuni
(<https://github.com/coreybutler/nvm-windows/releases>):
```powershell
nvm install 22.12.0
nvm use 22.12.0
```
Verificare:
```powershell
node --version   # v22.12.0 sau v20.19+
npm --version
```

### 2.3 .NET 8 SDK (backend)
Descarcă **.NET SDK 8.0** de la
<https://dotnet.microsoft.com/download/dotnet/8.0> (nu doar Runtime — ai nevoie de SDK).
Verificare:
```powershell
dotnet --version        # 8.0.x
dotnet --list-sdks
```

### 2.4 Docker Desktop (recomandat pentru DB + Redis)
Descarcă de la <https://www.docker.com/products/docker-desktop/>, instalează și
pornește-l. Verificare:
```powershell
docker --version
docker compose version
```

---

## 3. Clonarea proiectului

```powershell
git clone <URL_REPO> RouteOptimizer
cd RouteOptimizer
```

Structura relevantă:
```
RouteOptimizer/
├─ RouteOptimizer.API/            # backend .NET 8 (+ docker-compose.yml)
├─ RouteOptimizer.Core/
├─ RouteOptimizer.Infrastructure/
├─ RouteOptimizer.Web/            # frontend React/Vite
└─ RouteOptimizer.sln
```

---

## 4. Pornirea bazei de date și Redis (Docker)

Fișierul `RouteOptimizer.API/docker-compose.yml` conține deja serviciile
`postgres` (cu PostGIS) și `redis`. Pornește doar aceste două servicii:

```powershell
cd RouteOptimizer.API
docker compose up -d postgres redis
```

Verifică:
```powershell
docker compose ps
```

Datele de conexiune configurate implicit (vezi `appsettings.Development.json`):

| Parametru | Valoare |
|-----------|---------|
| Host / Port | `localhost:5432` |
| Database | `BusRouteOptimizer` |
| User | `busroute_user` |
| Password | `YourSecurePassword123!` |
| Redis | `localhost:6379` |

> **Alternativă fără Docker:** instalează PostgreSQL 15 + extensia PostGIS și
> Redis manual, apoi creează baza de date și utilizatorul cu credențialele de mai
> sus. Docker este însă calea recomandată.

---

## 5. Configurarea și rularea backend-ului (RouteOptimizer.API)

### 5.1 Verifică `appsettings.Development.json`
Fișierul există deja. Confirmă / actualizează secțiunile:
- `ConnectionStrings:DefaultConnection` → trebuie să corespundă cu Docker (vezi tabelul de mai sus).
- `ConnectionStrings:Redis` → `localhost:6379`.
- `ExternalAPIs:Mapbox:AccessToken` și `OpenWeatherMap:ApiKey` → înlocuiește
  valorile placeholder cu chei reale **doar dacă** ai nevoie de funcționalitățile
  respective (rutare Mapbox / vreme). Pentru pornire simplă poți lăsa placeholder-ele.

### 5.2 Restaurează pachetele și aplică migrările EF Core
```powershell
cd RouteOptimizer.API
dotnet restore
dotnet tool install --global dotnet-ef      # o singură dată, dacă nu e instalat
dotnet ef database update                    # creează schema în PostgreSQL
```

### 5.3 Pornește API-ul
```powershell
dotnet run --launch-profile https
```
API-ul rulează pe **https://localhost:7135**. Swagger UI:
<https://localhost:7135/swagger>.

> La prima rulare HTTPS local, acceptă certificatul de dezvoltare:
> ```powershell
> dotnet dev-certs https --trust
> ```

---

## 6. Configurarea și rularea frontend-ului (RouteOptimizer.Web)

### 6.1 Creează fișierul `.env` (sau `.env.local`)
În folderul `RouteOptimizer.Web`, creează un fișier `.env` cu variabilele
folosite de aplicație:

```dotenv
# URL-ul API-ului backend
VITE_API_BASE_URL=https://localhost:7135/api

# Nume aplicație afișat în UI
VITE_APP_NAME=Bus Route Optimizer

# Token Mapbox (opțional — necesar pentru hărți/rutare Mapbox)
VITE_MAPBOX_TOKEN=

# Keycloak (necesar doar dacă folosești autentificarea Keycloak)
VITE_KEYCLOAK_URL=http://localhost:8080
VITE_KEYCLOAK_REALM=route-optimizer
VITE_KEYCLOAK_CLIENT_ID=route-optimizer-web

# Debug
VITE_DEBUG=true
```

> Variabilele trebuie să înceapă cu prefixul `VITE_` ca să fie expuse în client.
> Dacă lași `VITE_API_BASE_URL` gol, aplicația folosește implicit
> `https://localhost:7135/api`. Serverul de dev proxy-ază oricum `/api` către
> `https://localhost:7135` (vezi `vite.config.ts`).

### 6.2 Instalează dependențele și pornește
```powershell
cd RouteOptimizer.Web
npm install
npm run dev
```
Frontend-ul se deschide automat pe **http://localhost:3000**.

Alte comenzi utile:
```powershell
npm run build        # build de producție (tsc + vite build)
npm run preview      # previzualizează build-ul
npm run lint         # verificare ESLint
npm run type-check   # verificare tipuri TypeScript
```

---

## 7. Autentificare cu Keycloak (opțional)

Aplicația folosește Keycloak pentru login. Pentru dezvoltare rapidă poți porni
un Keycloak local cu Docker:

```powershell
docker run -d --name keycloak -p 8080:8080 `
  -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin `
  quay.io/keycloak/keycloak:26.2 start-dev
```

Apoi, în consola de admin (<http://localhost:8080>):
1. Creează un **realm** (ex.: `route-optimizer`).
2. Creează un **client** public (ex.: `route-optimizer-web`) cu
   `Valid redirect URIs` = `http://localhost:3000/*` și `Web origins` = `http://localhost:3000`.
3. Creează cel puțin un utilizator cu parolă și rolurile necesare.
4. Actualizează variabilele `VITE_KEYCLOAK_*` din `.env` cu aceste valori.

---

## 8. Ordinea de pornire (rezumat zilnic)

1. Pornește Docker Desktop.
2. Bază de date + cache:
   ```powershell
   cd RouteOptimizer.API
   docker compose up -d postgres redis
   ```
3. (Opțional) Keycloak — vezi secțiunea 7.
4. Backend:
   ```powershell
   cd RouteOptimizer.API
   dotnet run --launch-profile https
   ```
5. Frontend:
   ```powershell
   cd RouteOptimizer.Web
   npm run dev
   ```
6. Deschide <http://localhost:3000>.

---

## 9. Depanare (probleme frecvente)

| Problemă | Soluție |
|----------|---------|
| `npm run dev` eșuează cu eroare de versiune Node | Actualizează la Node 20.19+ / 22.12+ (Vite 7 nu suportă versiuni mai vechi). |
| Frontend-ul nu ajunge la API / erori CORS | Verifică că backend-ul rulează pe `https://localhost:7135` și că `VITE_API_BASE_URL` e corect. |
| Eroare de certificat HTTPS local | Rulează `dotnet dev-certs https --trust`. |
| `dotnet ef` nu e recunoscut | `dotnet tool install --global dotnet-ef` și redeschide terminalul. |
| Conexiune eșuată la PostgreSQL | Verifică `docker compose ps` și că portul 5432 nu e ocupat de altă instanță. |
| Portul 3000 / 5432 / 6379 ocupat | Oprește procesul care îl folosește sau schimbă portul în config. |
| Login Keycloak eșuează | Verifică realm, clientId, redirect URIs și variabilele `VITE_KEYCLOAK_*`. |

---

## 10. Referințe rapide

- Frontend dev: <http://localhost:3000>
- API: <https://localhost:7135> · Swagger: <https://localhost:7135/swagger>
- PostgreSQL: `localhost:5432` (DB `BusRouteOptimizer`)
- Redis: `localhost:6379`
- Keycloak: <http://localhost:8080>
