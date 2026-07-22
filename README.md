# Secure Event Ticketing Platform (Sample DevSecOps Project)

Ovaj repozitorij je referentni uzorak aplikacije za kolegij **Uvod u DevOps - DevSecOps**.
Prikazuje cijeli tok: lokalni razvoj kroz Compose i produkcijski deployment kroz Kubernetes manifeste.

## Arhitektura

- `frontend` - web UI za pregled evenata i kupnju karata
- `api` - REST API za evente, narudzbe i health provjere
- `worker` - pozadinska obrada queue poruka
- `postgres` - trajna pohrana narudzbi
- `redis` - queue/cache sloj

### Brza validacija funkcionalnosti

1. Health API:y
   ```bash
   curl http://localhost:8080/healthz
   curl http://localhost:8080/readyz
   ```
2. Dohvati evente:
   ```bash
   curl http://localhost:8080/events
   ```
3. Posalji narudzbu:
   ```bash
   curl -X POST http://localhost:8080/tickets/purchase \
     -H "Content-Type: application/json" \
     -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
   ```
4. Provjeri obradene narudzbe:
   ```bash
   curl http://localhost:8080/tickets/orders
   ```
5. UI:
   - Otvori `http://localhost:3000`
   ## Produkcijski deployment (Kubernetes)

Preduvjeti:
- Kubernetes klaster s uključenim NGINX Ingress kontrolerom
- Unos u hosts datoteci koji `ticketing.local` mapira na IP klastera

Priprema tajni (Secret nije u repozitoriju, samo predložak):
```bash
cp k8s/02-secret.example.yaml k8s/02-secret.yaml
# uredi k8s/02-secret.yaml i postavi stvarnu POSTGRES_PASSWORD vrijednost
```

Deploy cijelog stacka (redoslijed po broju datoteke - namespace, config, baze, aplikacija, ingress, RBAC, NetworkPolicy):
```bash
kubectl apply -f k8s/
```

Provjera statusa:
```bash
kubectl get pods -n ticketing
kubectl get ingress -n ticketing
```

Validacija funkcionalnosti kroz Ingress:
```bash
curl http://ticketing.local/api/readyz
curl http://ticketing.local/api/events
```
- UI: otvori `http://ticketing.local`

Za rolling update, rollback i rješavanje problema u produkciji vidi `Documentation/TROUBLE-SHOOTING/RUNBOOK.md`.

## Sigurnosni elementi

- Multi-stage Docker build i non-root runtime korisnik
- Secret + ConfigMap odvojena konfiguracija
- Liveness/Readiness probe
- Resource requests/limits
- ServiceAccount + RBAC
- NetworkPolicy segmentacija
- Trivy skeniranje slika u CI pipelineu

Detalji skeniranja: Documentation/SECURITY/IMAGE-SCAN-REPORT.md