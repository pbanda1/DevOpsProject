
# CI/CD pipeline 

## Opis toka
    Pipeline (.github/workflows/ci.yml) pokreće se na svaki push i pull_request prema main grani, paralelno za sva tri servisa (frontend, api, worker) preko GitHub Actions matrixa.

    1. Checkout — preuzimanje koda iz repozitorija.

    2. Build validacija (npm ci) — instalacija ovisnosti točno prema package-lock.json, provjerava da su lockfile i ovisnosti ispravni prije nego se uopće krene na build slike.

    3. Docker build (--target runtime) — gradi se ista produkcijska (hardenirana) slika koja se stvarno objavljuje i deploya, lokalno unutar runnera (load: true), spremna za skeniranje prije nego ide bilo kamo dalje.

    4. Trivy sken — quality gate — slika se skenira na CRITICAL/HIGH ranjivosti. Ako se nešto takvo pronađe, korak vraća exit code 1 i cijeli pipeline staje — slika se ne pusha dalje.

    5. Odluka (grana + event) — push u registry se izvršava samo ako je sken prošao i radi se o push eventu na main grani (pull requestovi nikad ne pushaju sliku, samo se validiraju).

    6. Push u GHCR — slika se objavljuje u GitHub Container Registry (ghcr.io/pbanda1/devopsproject-servis), u skladu s politikom tagiranja iz image-policy.md.

## Sigurnost pipelinea

    Autentikacija prema GHCR ide preko ugrađenog, automatski generiranog GITHUB_TOKEN-a — nema ručno unesenih tajni ni u workflow datoteci ni bilo gdje drugdje u repozitoriju.

    permissions: packages: write je eksplicitno ograničen samo na ono što je potrebno (least privilege na razini workflow tokena).

    Sken je postavljen kao tvrdi gate, ne samo informativni izvještaj — build koji uvede novu CRITICAL/HIGH ranjivost fizički ne može stići do registryja.

## Verifikacija

    Pipeline je testiran uživo (push na main) — sva tri joba (frontend, api, worker) uspješno su prošla build, test-validaciju i Trivy sken, te su tri slike potvrđeno objavljene u GHCR-u (devopsproject-frontend, devopsproject-api, devopsproject-worker, svaka kao privatan paket vezan uz repozitorij).