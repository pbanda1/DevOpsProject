
## Obrazloženje odabira GHCR-a

    GitHub Container Registry odabran je umjesto alternativa (npr. Docker Hub) iz nekoliko konkretnih razloga specifičnih za ovaj projekt:

    Autentikacija bez dodatnih tajni - GHCR se autenticira kroz ugrađeni, automatski generirani GITHUB_TOKEN unutar GitHub Actions workflowa. Docker Hub bi zahtijevao ručno kreiranje i čuvanje zasebnog access tokena kao GitHub Secreta, što je dodatna površina za curenje tajni i dodatno održavanje.

    Rate limit - Docker Hub nameće ograničenje broja anonimnih/besplatnih pull zahtjeva po IP adresi, što može uzrokovati nepredvidive greške tijekom čestih CI buildova ili k8s rolling updateova. GHCR unutar GitHub ekosustava nema tu vrstu ograničenja za ovaj opseg korištenja.

    Vezanost paketa uz repozitorij - slike u GHCR-u su izravno povezane s repozitorijem projekta (vidljive u istom GitHub sučelju kao Packages), što olakšava sljedivost - jasno je koji commit/repo je proizveo koju sliku, bez potrebe za zasebnim organizacijskim računom na drugom servisu.

    Permissions na razini tokena - permissions: packages: write u workflowu ograničava token isključivo na ono što je potrebno (least privilege), što je teže postići jednako granularno s vanjskim registryjem.

    Trošak - privatni paketi u GHCR-u vezani uz repozitorij besplatni su za ovaj opseg korištenja, bez potrebe za plaćenim planom kao kod nekih alternativa za privatne repozitorije na Docker Hubu.
