# Arhitektura sustava i obrazloženje
# 1. Arhitektura i međuservisna komunikacija
Aplikacija se sastoji od pet servisa raspoređenih u tri sloja: prezentacijski (frontend), aplikacijski (API i worker) i podatkovni (PostgreSQL i Redis).
 # Prezentacijski sloj (Frontend) -> 
    Express servis servira statičke datoteke (HTML/JS) te endpoint /config koji preglednik obavještava o adresi API servisa. 
    Kod pokretanja stranice, klijentski JavaScript prvo poziva getConfig(), koji dohvaća apiBaseUrl s frontend servisa. 
    Ta adresa se zatim prosljeđuje funkciji loadEvents(apiBaseUrl), koja izravno - bez posrednika, dohvaća popis dostupnih evenata s API servisa (fetch(\${apiBaseUrl}/events`)).
    Isti princip vrijedi za kupnju karte: 
    klik na gumb pokreće buy(apiBaseUrl)`, koja šalje POST zahtjev izravno na API.
    Frontend servis dakle ne djeluje kao proxy ili Backend-for-Frontend (BFF) sloj nego isporučuje samo statički sadržaj i konfiguraciju, dok sva stvarna komunikacija ide izravno --> preglednik → API.
 # Aplikacijski sloj (API + Worker) -> 
    API servis (Express, port 8080) implementira asinkroni worker uzorak (eng. asynchronous worker pattern):
    sinkrone HTTP zahtjeve obrađuje API, dok se stvarna obrada (upis u bazu) prebacuje na zaseban proces — worker. 
    Uzorak omogućuje aplikaciji da se nosi s naglim navalama zahtjeva (npr. navala kupnje karata za popularan događaj), jer API ne čeka spor upis u bazu prije nego odgovori korisniku.
 # |||| API Breakdown ||||
    Na vrhu server.js datoteke definiraju se middleware komponente: CORS zaglavlja koja dopuštaju frontendu da s drugog porta nesmetano šalje zahtjeve API-ju na portu 8080, te middleware koji uklanja /api prefiks iz putanje kad aplikacija radi iza reverse proxyja — time Express ispravno prepoznaje internu rutu /events bez obzira dolazi li zahtjev izravno ili kroz proxy. 
    Port se čita iz varijable okruženja API_PORT, a ako nije postavljena, koristi se zadana vrijednost 8080.
    Popis dostupnih događaja trenutno je simuliran u memoriji radi trenutačnog odgovora, dok se veze prema bazama uspostavljaju kroz pgPool (connection pool za PostgreSQL) i redisClient (veza prema Redisu).
    Endpointi /healthz i /readyz služe kao dijagnostički sloj:
    /healthz odmah vraća status 200 ako je proces živ, 
    dok /readyz dodatno izvršava SELECT 1 nad bazom i PING nad Redisom — ako bilo koja provjera ne uspije, ruta vraća 503 (server unavailable), čime se sprječava usmjeravanje korisnika na instancu API-ja koja trenutno ne može obraditi narudžbu (osnova za Kubernetes liveness/readiness probe).
    Glavna poslovna ruta - /tickets/purchase provjerava jesu li poslani email, ID događaja i pozitivna količina karata, potom kreira objekt narudžbe s nasumično generiranim orderId (UUID) i statusom "queued".
    Narudžba se serijalizira u JSON string i gura na Redis listu naredbom LPUSH — API ne čeka upis u bazu, nego odmah vraća potvrdu klijentu, čime je vrijeme odgovora minimalno neovisno o trenutnom opterećenju baze.
# Worker ->
    Worker je potpuno odvojena Node.js aplikacija — nema HTTP ruta, ne koristi Express i ne sluša portove. 
    Njegov jedini zadatak je posredovanje između Redisa i PostgreSQL-a: čim API izvrši LPUSH, Redis odmah "budi" workera koji preuzima narudžbu i upisuje je u bazu kroz funkciju processOrder.
    Ako PostgreSQL privremeno postane nedostupan, worker se neće srušiti — ispisat će grešku, pričekati dvije sekunde i ponovno pokušati, dok narudžbe u međuvremenu sigurno ostaju u Redis redu čekajući da se baza oporavi.
    Oba procesa, API i worker, implementiraju graceful shutdown kroz rukovanje SIGTERM signalom: prije gašenja procesa uredno se zatvaraju sve otvorene veze prema PostgreSQL-u i Redisu, čime se izbjegava nasilno prekidanje aktivnog upisa podataka.
# Database Layer (POSTGREsql) ->
    PostgreSQL trajno pohranjuje obrađene narudžbe u tablici ticket_orders, dok Redis služi kao queue/cache sloj koji omogućuje asinkronu komunikaciju između API-ja i workera.

