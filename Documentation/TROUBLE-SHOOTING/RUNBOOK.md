
## Runbook — troubleshooting postupci 

## Sistematičan troubleshooting postupak

    Za svaki incident primjenjuje se isti redoslijed koraka:

    1 ->  Opažanje simptoma
    Pomoću naredbe: kubectl get pods -n ticketing
    Provjerava se status kolone i restart count.
    2 -> Dijagnostika
    Izvodi se kroz naredbe:
    * kubectl describe pod
    * kubectl logs
    * kubectl get events -n ticketing

    3 -> Provjera izvora istine za konfiguraciju
    Vrijednosti se čitaju iz stvarnog ConfigMap ili Secret objekta naredbom:
    kubectl get configmap [naziv_configmapa] -o yaml
   

    4 -> Korektivna mjera
    Minimalna, ciljana izmjena koja rješava utvrđeni uzrok.
    5 -> Validacija
    Funkcionalna provjera end-to-end.

---

