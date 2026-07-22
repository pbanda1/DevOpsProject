
# Upute za deployment u produkciju (Kubernetes)

## Preduvjeti
    1. Kubernetes klaster (lokalno npr. Docker Desktop s uključenim Kubernetes-om, ili kind/minikube; u produkciji stvarni klaster/OpenShift).
    2. Instaliran NGINX Ingress kontroler u klasteru.
    3. Unos u hosts datoteci koji domenu ticketing.local mapira na IP klastera (na Windowsu: C:\Windows\System32\drivers\etc\hosts).
    4. kubectl konfiguriran i spojen na ciljani klaster (kubectl config current-context).
    5. Slike servisa (frontend, api, worker) već objavljene u GHCR-u kroz CI pipeline (vidi Documentation/CI-CD/Pipeline-Explained.md).


## Priprema tajni
    Stvarni Secret (k8s/02-secret.yaml) namjerno nije u repozitoriju (vidi .gitignore) - u repozitoriju postoji samo predložak k8s/02-secret.example.yaml.
    Prije prvog deploya na novom klasteru:
        cp k8s/02-secret.example.yaml k8s/02-secret.yaml

    Zatim u k8s/02-secret.yaml zamijeni placeholder vrijednost POSTGRES_PASSWORD stvarnom lozinkom.

    Ako se slike povlače iz privatnog GHCR paketa, potreban je i imagePullSecret (ghcr-pull-secret) koji se referencira u api/frontend/worker manifestima:
    kubectl create secret docker-registry ghcr-pull-secret --docker-server=ghcr.io --docker-username=<github-username> --docker-password=<github-token> --namespace=ticketing


## Redoslijed primjene manifesta
    Datoteke u k8s/ su namjerno numerirane tako da redoslijed primjene prati stvarne ovisnosti (namespace prije svega ostalog, baze prije aplikacijskog sloja, RBAC/NetworkPolicy na kraju kad su svi objekti već poznati):

    00-namespace.yaml       - ticketing namespace
    01-configmap.yaml       - neosjetljiva konfiguracija (POSTGRES_DB, POSTGRES_USER, itd.)
    02-secret.yaml          - lozinke (kreirano iz .example predloška, vidi gore)
    03-postgres.yaml        - PVC + Deployment + Service za Postgres
    04-redis.yaml           - Deployment + Service za Redis
    05-api.yaml             - Deployment + Service za api
    06-worker.yaml          - Deployment za worker (nema Service, nema HTTP port)
    07-frontend.yaml        - Deployment + Service za frontend
    08-ingress.yaml         - Ingress (ticketing.local -> frontend / api)
    09-rbac.yaml            - ServiceAccounts + RBAC (least privilege po servisu)
    10-networkpolicy.yaml   - segmentacija mrežnog prometa

    Cijeli stack se diže jednom naredbom - kubectl primjenjuje datoteke iz direktorija abecednim redom, što ovdje odgovara ispravnom redoslijedu:

        kubectl apply -f k8s/


## Provjera stanja nakon deploya
    kubectl get pods -n ticketing
    kubectl get svc -n ticketing
    kubectl get ingress -n ticketing

    Svi podovi trebaju biti u stanju Running s READY 1/1. Ako neki pod nije spreman dulje vrijeme, vidi Documentation/TROUBLE-SHOOTING/RUNBOOK.md za dijagnostički postupak.


## Validacija funkcionalnosti
    curl http://ticketing.local/api/healthz
    curl http://ticketing.local/api/readyz
    curl http://ticketing.local/api/events

    curl -X POST http://ticketing.local/api/tickets/purchase ^
      -H "Content-Type: application/json" ^
      -d "{\"eventId\":\"evt-1001\",\"customerEmail\":\"student@example.com\",\"quantity\":1}"

    curl http://ticketing.local/api/tickets/orders

    UI: otvori http://ticketing.local u pregledniku i provedi kupovni tok od početka do kraja.


## Objava nove verzije (rolling update)
    Nakon što CI pipeline objavi novu sliku u GHCR (novi immutable tag, vidi Documentation/IMAGE-POLICY.md), ažuriraj odgovarajući Deployment:

    kubectl set image deployment/api api=ghcr.io/<owner>/devopsproject-api:<novi-tag> -n ticketing
    kubectl rollout status deployment/api -n ticketing

    Kubernetes sam provodi postupan (rolling) update - stari pod ostaje aktivan dok novi ne prođe readiness probe, pa nema prekida u radu aplikacije.


## Rollback na prethodnu verziju
    Ako novi deploy pokaže problem (npr. CrashLoopBackOff, ImagePullBackOff):

    kubectl rollout history deployment/api -n ticketing
    kubectl rollout undo deployment/api -n ticketing --to-revision=<broj-ispravne-revizije>
    kubectl rollout status deployment/api -n ticketing

    Napomena: eksplicitno navođenje --to-revision je pouzdanije od oslanjanja na zadano "prethodna revizija" ponašanje - vidi stvaran primjer i objašnjenje u Documentation/TROUBLE-SHOOTING/RUNBOOK.md.


## Troubleshooting
    Za dijagnostiku bilo kojeg problema u produkciji (pad poda, nedostupna baza, kriva konfiguracija) koristi sistematičan postupak i stvarne primjere iz Documentation/TROUBLE-SHOOTING/RUNBOOK.md.


## Gašenje / čišćenje okoline
    Gašenje svih resursa iz ovog projekta (namespace i sve unutar njega):

    kubectl delete namespace ticketing

    Napomena: ovo briše i PVC s podacima Postgresa. Za produkciju gdje se podaci moraju sačuvati, umjesto brisanja namespacea gasiti/skalirati pojedine Deploymente (kubectl scale deployment/<ime> --replicas=0).
```