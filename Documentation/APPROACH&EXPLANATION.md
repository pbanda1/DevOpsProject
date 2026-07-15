# Usporedba kontejnera i virtualnih strojeva
Za ovaj projekt odabrana je kontejnerizacija (Podman/Docker), a ne klasični virtualni strojevi, iz sljedećih razloga specifičnih za ovu aplikaciju:
   # Granularnost(razbijanje na manje dijelove) i broj servisa:
        Aplikacija se sastoji od pet neovisnih procesa (frontend, api, worker, postgres, redis) - Fine-grained granularity
        Sitna granularnost znači da je sustav razbijen na mnoštvo malih, specijaliziranih servisa od kojih svaki radi samo jednu specifičnu stvar.
        Ovaj projekt koristi upravo taj pristup, frontend, api, worker.
        Pokretanje svakog od servisa u zasebnom VM-u zahtijevalo bi vlastiti gostujući OS po servisu, što drastično povećava potrošnju resursa (RAM, disk) i vrijeme pokretanja, dok kontejneri dijele kernel host sustava i pokreću se u sekundama.
   # Brzina razvojnog ciklusa:
        Zahtjev projekta uključuje hot-reload za brzi razvoj. 
        Kontejneri se grade i restartaju u sekundama do desetaka sekundi,dok bi restart VM-a za svaku promjenu u kodu bio neprihvatljivo spor za iterativan razvoj.  
   # Konzistentnost orkuženja:
        Kontejnerska slika sadrži točno definirane verzije runtime-a i ovisnosti (Node.js verzija, npm paketi), 
        isti Containerfile koristi se lokalno (Compose) i u produkciji (Kubernetes/OpenShift).
   # Orkestracija i skaliranje:
        Drugi dio projekta zahtijeva Kubernetes/OpenShift orkestraciju — rolling update, readiness/liveness probe, horizontalno skaliranje pojedinih servisa (npr. dodatne instance API-ja ili workera pod opterećenjem).
        Ovakva granularna, brza orkestracija na razini pojedinačnog procesa prirodno je podržana kontejnerskim platformama, dok bi ekvivalent na VM razini (npr. auto-scaling grupe VM-ova) bio sporiji i skuplji..  
# OPRAVDANJE ZA VM IMPLEMENTACIJU -> 
     Virtualni strojevi ostaju bolji izbor kada je potrebna jača izolacija na razini kernela (multi-tenant okruženja s nepouzdanim korisnicima), različiti operacijski sustavi na istom hostu, ili kada aplikacija zahtijeva izravan pristup hardveru koji kontejnerski runtime ne izlaže. 
     Za ovaj projekt, gdje su svi servisi međusobno povjerljivi dijelovi iste aplikacije, ta razina izolacije nije nužna, pa prednosti kontejnera (brzina, gustoća, prenosivost) nadilaze prednosti VM-ova.