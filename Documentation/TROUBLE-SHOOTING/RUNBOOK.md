
## Runbook — troubleshooting postupci kod incidenata
    Za svaki incident primjenjuje se isti redoslijed koraka:
## 1 ->  Opažanje simptoma
    Pomoću naredbe: kubectl get pods -n ticketing
    Provjerava se status kolone i restart count.
## 2 -> Dijagnostika
    Izvodi se kroz naredbe:
        * kubectl describe pod
        * kubectl logs
        * kubectl get events -n ticketing
## 3 -> Provjera izvora istine za konfiguraciju
    Vrijednosti se čitaju iz stvarnog ConfigMap ili Secret objekta naredbom:
    kubectl get configmap [naziv_configmapa] -o yaml
## 4 -> Korektivna mjera
    Minimalna, ciljana izmjena koja rješava utvrđeni uzrok.
## 5 -> Validacija
    Funkcionalna provjera end-to-end.

## Incident 1: Namjerno pokvaren deploy i rollback 
    Kontekst:
        Deployment api namjerno je promijenjen na nepostojeći image tag da bi se testirao stvaran troubleshooting i rollback postupak (rolling update / rollback).
    Simptom:
        Naredba: kubectl set image deployment/api api=ghcr.io/pbanda1/devopsproject-api:ne-postoji -n ticketing
        
    kubectl get pods -n ticketing pokazao je novi api pod u statusu ImagePullBackOff, dok je stari api pod ostao 1/1 Running - rolling update strategija (maxUnavailable=0, maxSurge=1 za replicas=1) nije ugasila stari pod dok novi nije spreman, pa aplikacija nikad nije pala.

## Dijagnoza
    kubectl get events -n ticketing --field-selector involvedObject.name=<ime poda> - api-5fd946c478-tkm7b

    Lanac događaja: Pulling -> Failed to pull image ... not found -> ErrImagePull -> BackOff -> ImagePullBackOff.

    Prva komplikacija pri rollbacku:
    kubectl rollout undo deployment/api -n ticketing (bez --to-revision) prijavio je "rolled back", 
    ali kubectl get deployment api -n ticketing -o jsonpath="{.spec.template.spec.containers[0].image}" i dalje je pokazivao pokvarenu sliku (ne-postoji). 

    kubectl rollout history je nakon toga pokazivao promijenjenu numeraciju revizija (1, 4, 5 umjesto očekivanih 1, 2, 3) - Kubernetes pri undo ponovno koristi i renumerira postojeće ReplicaSetove umjesto da čuva stare brojeve revizija, što je zbunilo početnu pretpostavku da će zadano ponašanje vratiti ispravnu sliku.

## Analiza uzroka
    kubectl rollout history deployment/api -n ticketing --revision=4 i --revision=5 otkrili su stvarni sadržaj svake revizije: revizija 4 sadrži ispravnu sliku (fae7c06d110d9921151fc7b8af22a705e5bc6bdb), revizija 5 pokvarenu (ne-postoji) i upravo je revizija 5 ostala aktivna nakon prvog undo pokušaja.

## Korektivna mjera
    Umjesto oslanjanja na zadano ponašanje, revizija je ciljana eksplicitno:
    kubectl rollout undo deployment/api -n ticketing --to-revision=4

## Validacija
    kubectl rollout status deployment/api -n ticketing potvrdio je uspješan rollout.
    kubectl get deployment api -n ticketing -o jsonpath="{.spec.template.spec.containers[0].image}" potvrdio je ispravnu sliku.
    kubectl get pods -n ticketing pokazao je sve podove u stanju 1/1 Running.

## Zaključak
    Rolling update strategija osigurala je nulti prekid rada aplikacije tijekom cijelog incidenta, unatoč pokvarenom image tagu. 
    kubectl rollout undo bez eksplicitnog --to-revision može biti nepouzdan kad povijest revizija nije linearna (npr. nakon više uzastopnih promjena) - eksplicitno ciljanje revizije brojem je pouzdanija praksa za produkcijski rollback.

## Incident 2: Pad baze - Postgres nedostupan 
    Kontekst:
    Postgres Deployment namjerno skaliran na 0 replika (namjerno naredjujemo Kubernetesu da ugasi sve instance (pods)) da se simulira nedostupnost baze u produkciji.

    Simptom:
    Naredba: kubectl scale deployment/postgres -n ticketing --replicas=0
    kubectl get pods -n ticketing pokazao je da api pod prelazi u 0/1 Running, s ponovljenim restartima (3 restarta u svega par sekundi).

    Napomena o testiranju:
    Prvi pokušaj provjere preko curl http://localhost:8080/readyz vratio je lažno pozitivan rezultat (200, status ready) jer je lokalni Docker Compose stack i dalje bio pokrenut na istom portu na razvojnom računalu - test je pogodio Compose, ne k8s. 
    Ispravan test za k8s prolazi kroz Ingress: curl http://ticketing.local/api/readyz -UseBasicParsing

## Dijagnoza
    kubectl logs deployment/api -n ticketing --previous otkrio je stvarni uzrok pada: "Unhandled 'error' event" i potpun crash Node.js procesa, odmah nakon poruke Postgresa "FATAL: terminating connection due to administrator command".

## Analiza uzroka
    pg Pool objekt u api/src/server.js nema registriran error handler (pool.on('error', ...)). 
    Kad Postgres prekine postojeću vezu (gašenje, restart, skaliranje na 0), pg biblioteka emitira 'error' event na Pool instanci; bez handlera, Node.js default ponašanje je da baci iznimku i sruši cijeli proces - umjesto da API samo javi 503 na /readyz i nastavi raditi dok se baza ne vrati.

## Korektivna mjera (operativna)
    kubectl scale deployment/postgres -n ticketing --replicas=1
    Kubernetes RestartPolicy: Always automatski je ponovno pokretao api pod dok se veza prema bazi nije uspostavila - nije bila potrebna ručna intervencija na api Deploymentu.

## **Preporuka** 
    Dodati pool.on('error', (err) => console.error('Neočekivana greška na Postgres pool-u:', err)) u api/src/server.js, kako gubitak veze prema bazi ne bi rušio cijeli proces nego samo rezultirao neuspješnim /readyz odgovorom dok se veza ne obnovi. Ovo bi svelo broj nepotrebnih restart ciklusa i ubrzalo stvarni oporavak.

## Validacija
    Nakon vraćanja Postgresa: kubectl get pods -n ticketing pokazao je api pod natrag u stanju 1/1 Running, bez novih restartova.
    curl http://ticketing.local/api/readyz -UseBasicParsing vratio je 200 {"status":"ready"} kroz stvarni k8s put (Ingress -> api Service), čime je potvrđen potpun oporavak.

## Zaključak
    Iako je aplikacija dizajnirana s /readyz provjerom baš za ovakve scenarije, sam api proces nije otporan na prekid veze prema bazi tijekom rada. Ovo je konkretan primjer gdje je infrastrukturni sloj (Kubernetes RestartPolicy) nadoknadio nedostatak u aplikacijskom sloju - sustav se oporavio, ali kroz restart umjesto kroz graceful degradaciju.
