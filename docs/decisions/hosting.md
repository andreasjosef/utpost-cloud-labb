# Beslutsdokument: hostingval

*Ett av teamets sex beslutsdokument. Skrivs i M6, hålls levande. Samma mall som `api-stil.md`, `databas.md` och `testing.md`. Det här dokumentet examinerar Cloud 1 (redogöra för hostinglösningar som AWS, Azure, Google Cloud) – skriv så att en läsare utan Azure-konto förstår valet.*

**Datum:**
**Beslut:** *(en eller två meningar: var kör Utpost i staging, var ska den köra i produktion, och vad är skillnaden?)*

## Bakgrund
*Vad har vi när M5 är klar: två images i GHCR taggade med sha, en pipeline som bygger dem, compose lokalt. Vad saknas för att någon utanför teamet ska kunna öppna appen? Vilka krav ställer kursen (M6: automatisk deploy till staging vid merge, secrets ur koden, publik URL)?*

## Vad vi faktiskt hyr
*Fyll i tabellen med era egna värden ur Azure-portalen eller `az containerapp show`. Kolumnen "nivå" = IaaS / PaaS / SaaS / serverless (klippet *Molnet: vad det är och vad man hyr*).*

| Del | Tjänst | Nivå | Vem sköter uppdateringar/skalning | Vad det kostar oss i staging |
|---|---|---|---|---|
| Klient + API | Azure Container Apps (Consumption), appen `utpost-staging` i miljön `utpost-env`, Sweden Central. `client` (nginx, 0,25 vCPU / 0,5 GiB) tar emot trafiken, `api` (0,5 vCPU / 1 GiB) är sidecar på samma localhost | Serverless containrar (PaaS) – vi lämnar en image, Azure kör den | Azure: OS, runtime-värd, TLS-certifikat, skalning 0 → 1 replica på trafik. Vi: imagen och vilken sha som kör (pipelinen) | 0 kr. Appen skalar till noll mellan anropen. Vaken kostar hela appen 1,75 vCPU + 3,5 GiB, och gratiskvoten per prenumeration (180 000 vCPU-s + 360 000 GiB-s per månad) räcker till 180 000 / 1,75 ≈ 102 900 s ≈ **28,6 h vaken app per månad** – för hela klassen (29 st) tillsammans. Cost Management visar 0 kr för resursgruppen hittills (`ActualCost`, MonthToDate) |
| Postgres | `postgres:16` som sidecar-container i samma app (0,5 vCPU / 1 GiB) | IaaS-liknande: vi kör databasen själva i en container, bara på en serverless plattform | Vi – version, backup, data. Ingen volym: databasen töms vid varje kallstart, ny revision och omstart, och seedas om (`SEED_ON_START=1`) | Ingår i appens förbrukning ovan |
| MongoDB | `mongo:8.2` som sidecar-container i samma app (0,5 vCPU / 1 GiB) | Som Postgres | Som Postgres | Ingår i appens förbrukning ovan |
| Images | GHCR | SaaS (container-registry) | GitHub. Vi bestämmer bara vad som pushas (jobbet *Image*, taggat `sha-<7>`) | 0 kr – publika paket är gratis |
| Deploy | GitHub Actions → Azure (OIDC-inloggning, `az containerapp update`) | SaaS (CI/CD) | GitHub sköter runners; vi sköter `ci.yml`. Azure-identiteten (app registration + federerad identitet, Contributor på bara vår resursgrupp) skapades en gång med `identitet.sh` | 0 kr – publikt repo, Actions-minuter är gratis. Deploy-jobbet tar ~1,5 min (88 s) |

## Alternativ vi jämförde
*Minst tre riktiga alternativ, och minst två leverantörer. Förslag: Azure Container Apps (det vi gjorde) · Azure App Service for Containers · Google Cloud Run · AWS ECS på Fargate (App Runner tar inte längre nya kunder) · en VM (IaaS) med docker compose · Render/Railway (PaaS utanför de tre stora). För varje: vad hade ändrats i repot, vad hade kostat, vad hade vi fått/förlorat?*

| Alternativ | Vad ändras i repot | Kostnad (staging) | För | Emot |
|---|---|---|---|---|
| | | | | |

## Staging och produktion är inte samma sak
*I staging kör databaserna som sidecars utan volym och seedas vid varje kallstart. Vad krävs för produktion? Managed databas (vilken, vad kostar den), backup, en app som inte skalar till noll, egen domän och certifikat, loggar. Skriv vad ni skulle byta och vad som INTE behöver bytas (imagen? pipelinen?).*

## Inlåsning
*Om Azure försvann i morgon: vilka filer i repot ändras, och vilka ändras inte? Räkna upp dem. Det är svaret på hur portabel lösningen är – och argumentet för att bygga på image + pipeline i stället för plattformens egna byggverktyg.*

Ingenting i imagerna vet att de kör i Azure. Det som låser oss är fyra filer och några värden – det är det som byts om vi flyttar till Cloud Run, ECS eller en VM.

**Finns bara för Azure (byts vid flytt):**
- `infra/azure/staging.yaml` – appen som fyra sidecars, secrets, skalning 0–1
- `infra/azure/skapa-staging.sh` – skapar miljön och appen, hittar på hemligheterna
- `infra/azure/identitet.sh` – app registration, federerad identitet, rollen Contributor
- jobbet `deploy` i `.github/workflows/ci.yml` – `azure/login` och `az containerapp update`

**Likadant i alla moln (byts inte):**
- `api/Dockerfile` och `client/Dockerfile` – och imagerna i GHCR
- `client/nginx.conf.template` – nginx-mallen, adressen kommer från `API_UPSTREAM`
- `.env.example` – samma variabelnamn överallt
- `api/src/` – koden läser bara miljövariabler
- jobben Kvalitet, Bygg, Integration och Image i `ci.yml`

**Värdena som skiljer compose från Azure:**

| Variabel | Compose | Azure staging |
|---|---|---|
| `API_UPSTREAM` | `api:4000` (egen container, eget namn) | `localhost:4000` (sidecar, delad localhost) |
| `DATABASE_URL` | `…@postgres:5432/utpost` | `…@localhost:5432/utpost`, som secret `database-url` |
| `MONGO_URL` | `…@mongo:27017/…` | `…@localhost:27017/…`, som secret `mongo-url` |
| `JWT_SECRET` | dev-värde i `.env` | slumpat av `skapa-staging.sh`, finns bara som secret `jwt-secret` i Azure |
| `SEED_ON_START` | `0` (volymen behåller datan) | `1` (ingen volym – tom databas vid varje kallstart) |

## Konsekvenser
*Vad kostar valet i tid och pengar? Kallstarten när appen skalat till noll. Gratiskvoten som delas av alla i prenumerationen. Hemligheter som nu lever på två ställen (lokalt `.env`, i molnet secrets) – hur roteras de? Vem i teamet har tillgång till portalen?*

- **Kallstarten:** första anropet efter att appen skalat till noll tog **~21 s** till `"ok":true` (uppmätt 30/10). Under tiden svarar nginx med 502 (efter 16,5 s) – client är uppe före api. api-loggen: väntar på Postgres 3 gånger, seedar en tom databas (204 ms), lyssnar. En besökare som kommer först efter en paus väntar alltså en halv minut.

**Skrivet av:**
