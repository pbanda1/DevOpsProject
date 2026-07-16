
# UPUTE ZA DEVSE ZA LOKALNI RAZVOJ 

## Preduvjeti

    Docker Desktop s uključenim WSL2 backendom na Windowsima
    Kopiraj .env.example datoteku u .env pomoću naredbe:
    cp .env.example .env
    Ako si u Windows PowerShellu, koristi naredbu:
    Copy-Item .env.example .env

    Vrijednosti u .env.example datoteci (gdje je POSTGRES_HOST postavljen na postgres, a REDIS_HOST na redis) namjerno su konfigurirane za rad unutar Docker Compose mreže. 
    Tamo se servisi međusobno pronalaze i komuniciraju preko svojih imena, a ne preko localhosta.



## Pokretanje cijelog stacka

    Iz root foldera projekta pokreni naredbu:

      docker compose -f compose.yaml up -d --build

    Ova jedna naredba odrađuje sve: gradi slike za frontend, api i worker (pri čemu koristi dev stage iz Dockerfilea s nodemonom), pokreće bazu postgres i priručnu memoriju redis, čeka da oba servisa postanu potpuno spremna i zdrava, te na kraju pokreće naše aplikacijske servise. Prvo pokretanje i build mogu potrajati par minuta, ali svaki sljedeći put sve ide puno brže jer Docker koristi cache.

    Ako želiš provjeriti status svih pokrenutih servisa, upiši:

      docker compose -f compose.yaml ps

    Za praćenje logova svih servisa odjednom ili pojedinačno (npr. samo za api), koristi ove naredbe:

      docker compose -f compose.yaml logs -f

      docker compose -f compose.yaml logs -f api


## Hot-reload i automatsko osvježavanje

    Izvorni kod svakog našeg servisa (mape api/src, worker/src i frontend/src) spojen je direktno u kontejner preko takozvanog bind mounta. To znači da će bilo kakva izmjena koda koju napraviš u svom editoru na računalu automatski pokrenuti restart aplikacije unutar kontejnera preko nodemona, bez ikakve potrebe da ponovno gradiš cijelu Docker sliku.

    Napomena za ekipu na Windowsima i WSL2: promjene datoteka na Windows disku ponekad se ne prenesu ispravno u kontejner kao standardni file-system događaji. Zbog toga smo u compose.yaml datoteci za api, worker i frontend postavili varijablu CHOKIDAR_USEPOLLING na true. To tjera nodemon da svake sekunde sam provjerava ima li promjena u kodu i rješava taj problem.



## Gašenje servisa

    Kada želiš zaustaviti sve kontejnere, a da ti podaci u postgres_data volumenu i dalje ostanu sigurni i sačuvani, pokreni:

    docker compose -f compose.yaml down

    Ako pak želiš napraviti potpuno čišćenje i pobrisati apsolutno sve, uključujući i same podatke iz baze kako bi vratio sustav na tvorničke postavke, dodaj -v zastavicu:

    docker compose -f compose.yaml down -v



## Validacija nakon pokretanja

    Nakon uspješnog dizanja sustava, sve testiramo na isti način kao i kod ručnog pokretanja. Svi naši endpointi (/healthz, /readyz, /events, /tickets/purchase, /tickets/orders) i web sučelje na adresi http://localhost:3000 moraju raditi potpuno identično, samo što se ovaj put sve vrti kroz orkestrirane Docker kontejnere.