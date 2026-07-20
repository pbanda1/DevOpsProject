
# Sigurnosno izvješće skeniranja slika (CP3 / Ishod I2)

## Metodologija

    Skeniranje je provedeno alatom Trivy (Aqua Security), pokrenutim kao Docker kontejner. 
    

    Naredba -> docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache/ aquasec/trivy image <ime-slike>

    Skenirane su tri aplikacijske slike (frontend, api, worker), i to na dva različita builda:

    1. dev target (devops-project-app-api, devops-project-app-worker, devops-project-app-frontend) — slike koje Docker Compose gradi i koristi za lokalni razvoj s hot-reloadom.

    2. runtime target (devops-project-app-api:runtime, devops-project-app-worker:runtime, devops-project-app-frontend:runtime) — produkcijske slike, izgrađene eksplicitno preko docker build --target runtime, koje se stvarno objavljuju u registry i deployaju.

## Početni nalaz

    Sve tri slike (na oba builda, prije ispravka) prijavile su identičan skup od 7 ranjivosti:

    * Biblioteka: picomatch
    * CVE: CVE-2026-33671
    * Ozbiljnost: HIGH

    * Biblioteka: sigstore
    * CVE: CVE-2026-48815
    * Ozbiljnost: HIGH

    * Biblioteka: @sigstore/core
    * CVE: CVE-2026-48758
    * Ozbiljnost: MEDIUM

    * Biblioteka: brace-expansion
    * CVE: CVE-2026-33750
    * Ozbiljnost: MEDIUM

    * Biblioteka: ip-address
    * CVE: CVE-2026-42338
    * Ozbiljnost: MEDIUM

    * Biblioteka: picomatch
    * CVE: CVE-2026-33672
    * Ozbiljnost: MEDIUM

    * Biblioteka: tar
    * CVE: CVE-2026-53655
    * Ozbiljnost: MEDIUM


## Analiza uzroka

    Runtime slika nikad ne poziva npm — aplikacija se pokreće izravno preko node src/server.js (odnosno node src/worker.js). npm je stoga u produkcijskoj slici čist "dead load": povećava površinu za napad i veličinu slike bez ikakve funkcionalne koristi.

## Korektivna mjera

    U runtime stage svih triju Dockerfile-ova dodan je korak koji uklanja npm i njegove komponente prije prebacivanja na non-root korisnika:

    RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/corepack /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack

    Mora se izvršiti prije USER node direktive, jer te putanje pripadaju root korisniku.

## Rezultat nakon ispravka

    Ponovljeno skeniranje :runtime slika (nakon docker build --target runtime bez cachea):
    Rezultat nakon skeniranja za sve tri slike proizvodi 0 ranjivosti. 

## Napomena o dev slikama

    dev target slike (koje koristi docker compose za lokalni razvoj) i dalje sadrže npm — to je namjerno i ispravno, jer nodemon (hot-reload alat) i npm install zahtijevaju npm da bi radili.
    Ove slike se ne objavljuju u registry i ne deployaju — koriste se isključivo lokalno, na developerovom računalu, pa je prihvaćena razina rizika za njih viša nego za produkcijsku sliku.
    Ovo je zabilježeno i u image-policy.md (politika tagiranja)

