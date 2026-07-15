# Odluka o base image i politici tagiranja 

## Base image

    Odabran je node:22-alpine kao osnovna slika za sva tri Node.js servisa (frontend, api, worker).

    Razlozi:

    - Alpine varijanta koristi musl libc i BusyBox umjesto pune Debian/Ubuntu distribucije, što rezultira slikom od svega ~40-50 MB u odnosu na ~350+ MB standardne `node:22` slike. 
    Manja slika znači manju površinu za napad (manje instaliranih paketa = manje potencijalnih CVE-a) i brži pull/deploy.
    - Node 22 je aktivna LTS verzija u trenutku pisanja projekta, s dugoročnom podrškom i sigurnosnim zakrpama.
    -Koristimo isti base image kroz apsolutno sve faze builda (base, dev, deps, runtime). Na taj način smo sigurni da će se aplikacija ponašati identično i kod nas na lokalnom računalu tijekom razvoja i kasnije u produkciji

## Non-root korisnik

    Sve tri runtime slike pokreću proces kao ugrađeni `node` korisnik (UID/GID 1000) iz službene `node` slike, umjesto zadanog `root`. Datoteke aplikacije se eksplicitno vlasnički dodjeljuju (`--chown=node:node`) tom korisniku prilikom kopiranja u finalni stage.

## Multi-stage struktura

    Svaki servis ima 4 stage-a u jednom Dockerfile-u:

    base - zajednički layer s package.json/package-lock.json (cache-friendly)
    dev – instalira apsolutno sve ovisnosti, uključujući i devDependencies. Ovaj stage koristim u compose.yaml jer omogućuje hot-reload pomoću nodemon-a i lokalnog bind mounta.
    deps – ovdje instaliram samo produkcijske ovisnosti koristeći naredbu npm ci --omit=dev
    runtime – zadnji i zadani stage (pokreće se po defaultu ako ne specificiram --target). Uzima čistu node:22-alpine sliku, postavlja node korisnika i iz deps stagea kopira samo node_modules i izvorni kod. Unutra nema nikakvih build alata, npm cachea niti dev paketa.

## Politika tagiranja i objave slika

    - Zabranjeno korištenje :latest taga u produkciji.  Svaka slika koja ide u registry mora imati nepromjenjivi (immutable) tag.
    - Predložena shema tagiranja: <servis>:<semver>-<git-short-sha>, npr. api:1.0.0-a1b2c3d. Semver prati verziju iz package.json, git SHA osigurava        jedinstvenost i sljedivost do točnog commita.
    - Slike se grade i objavljuju kroz CI u GitHub Container Registry (GHCR), povezan s repozitorijem projekta.
    - Rollback  se oslanja upravo na ovu shemu - vraćanje na prethodni immutable tag umjesto oslanjanja na "trenutno stanje" pomičnog taga.
    - Prije objave slike u registry, slika prolazi Trivy sken kao quality gate - slika s critical/high ranjivostima se ne objavljuje.
