# Secure Event Ticketing Platform (Sample DevSecOps Project)

Ovaj repozitorij je referentni uzorak aplikacije za kolegij **Uvod u DevOps - DevSecOps**.
Prikazuje cijeli tok: lokalni razvoj kroz Docker Compose i produkcijski deployment kroz Kubernetes manifeste.

## Arhitektura

- `frontend` - web UI za pregled evenata i kupnju karata
- `api` - REST API za evente, narudzbe i health provjere
- `worker` - pozadinska obrada queue poruka
- `postgres` - trajna pohrana narudzbi
- `redis` - queue/cache sloj

## 1.dio Lokalni Razvoj - Docker Compose

- Preduvjet: Docker Desktop, Git

Koraci: 
1. Kloniraj repo i uđi u folder projekta
2. Kopiraj .env.example u .env

```bash
cp .env.example .env
```

Vrijednosti mogu ostati iste za lokalni razvoj.

### Pokretanje: Dev mod + hot-reload

```bash
docker compose up --build
```

Učitava docker-compose.override.yml
```bash
Windows: hot-reload koristi --legacy-watch ( više u docs/runbook.md)
```
### Gašenje

```bash
docker compose down
```

### Gašenje s pražnjenjem baze 

```bash
docker compose down -v
```

### Brza validacija funkcionalnosti

1. Health API:
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


## 2. dio - Deploy na produkciju

- Preduvjeti: Docker Desktop s uključenim Kubernetesom, kubectl koji dolazi uz Docker Desktop, Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.3/deploy/static/provider/cloud/deploy.yaml
```
Pričekaj da ingress-nginx-controller bude Running:

```bash
kubectl get pods -n ingress-nginx
```

Koraci:

1. Example u stvarni Secret file

```bash
cp k8s/secret.example.yaml k8s/secret.yaml
```

 (stvarni secret u .gitignore)

2. Primjeni manifeste

```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/postgres/init-configmap.yaml
kubectl apply -f k8s/postgres/pvc.yaml
kubectl apply -f k8s/postgres/deployment.yaml
kubectl apply -f k8s/postgres/service.yaml
kubectl apply -f k8s/redis/deployment.yaml
kubectl apply -f k8s/redis/service.yaml
kubectl apply -f k8s/rbac/serviceaccount.yaml
kubectl apply -f k8s/rbac/role.yaml
kubectl apply -f k8s/rbac/rolebinding.yaml
kubectl apply -f k8s/api/deployment.yaml
kubectl apply -f k8s/api/service.yaml
kubectl apply -f k8s/worker/deployment.yaml
kubectl apply -f k8s/frontend/deployment.yaml
kubectl apply -f k8s/frontend/service.yaml
kubectl apply -f k8s/networkpolicy.yaml
kubectl apply -f k8s/ingress.yaml
```

3. Check

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```

### Validacija:

```bash
curl http://localhost/api/healthz
curl http://localhost/api/readyz
curl http://localhost/api/events
curl -X POST http://localhost/api/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
curl http://localhost/api/tickets/orders
 UI:
   - Otvori `http://localhost`
```

### Gašenje:

```bash
kubectl delete -f k8s/ --recursive,
```
(Ukoliko nakon gašenja i ponovnog dizanja postgres ne runna check docs/runbook.md)

### Potpuni reset: 
Docker Desktop UI - Reset Kubernetes Cluster

## Sigurnosni elementi

- Multi-stage Docker build i non-root runtime korisnik
(USER node u Dockerfileu; securityContext s runAsNonRoot/runAsUser na K8s razini za sve servise)
- Secret + ConfigMap odvojena konfiguracija
(bez hardkodiranih lozinki; k8s/secret.yaml isključen iz Gita)
- Liveness/Readiness probe
(exec/pgrep tip za worker jer nema HTTP server, pa gleda postoji li proces umjesto pozivanja URL-a)
- Resource requests/limits
- ServiceAccount + RBAC
(least-privilege, testirano kubectl auth can-i)
- NetworkPolicy segmentacija
(deny-by-default + eksplicitne iznimke, testirano wget)
- Trivy skeniranje slika u CI pipelineu
(s quality gateom (exit-code: 1))
- Trivy config sken K8s manifesta (informativan; dokumentirane iznimke u .trivyignore.yaml)

- Detalji skeniranja: `docs/security/` (Trivy izvještaji za sve tri slike i K8s manifeste)

- Dodatna dokumentacija: `docs/runbook.md` — dijagnostika i rješenja stvarnih problema na koje se naišlo tijekom razvoja