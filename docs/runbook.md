# Runbook

Popis problema tijekom izrade i načini njihovog rješavanja.

## Problem 1: Hot-reload ne radi na Windowsu

**Simptom:** Promjena u `server.js`(ili drugoj promijenjenoj datoteci) ne primjenjuje se uživo iako ima postavljen hot-reloadu.

**Dijagnoza:**
```bash
docker compose exec api cat src/server.js
```
Pokazuje da kontejner vidi novi sadržaj datoteke. Problem je u `nodemon`-u.

**Uzrok:** `nodemon` koristi `inotify` signale koji se gube kroz Windows → WSL2 → Docker Desktop slojeve.

**Popravak:**
```yaml
command: npx nodemon --legacy-watch src/server.js
```

**Provjera:** Promjena koda odmah izaziva `[nodemon] restarting due to changes...` i vidljiv rezultat.

---

## Problem 2: Postgres Container Error pri pokretanju


### Korak 1 — CreateContainerConfigError

**Simptom:**
```
Error: container has runAsNonRoot and image will run as root
```

**Dijagnoza:**
```bash
kubectl get pods
kubectl describe pod <postgres-pod>
```
Events sekcija pokazuje točan uzrok.

**Uzrok:** Postavljen `runAsNonRoot: true` bez `runAsUser`. K8s neće pokrenuti kontejner kad ne zna koji non-root UID koristiti.

**Popravak:**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 999
```
UID 999 potvrđen (pretragom i vlastitim testiranjem) kao stvarni `postgres` korisnik unutar slike.

**Rezultat:** Kontejner se pokreće, ali odmah pada s novom greškom (korak 2).

### Korak 2 — Operation not permitted na novom volumenu

**Simptom:**
```bash
kubectl logs <postgres-pod>
```
```
chmod: changing permissions of '/var/lib/postgresql/data': Operation not permitted
initdb: error: could not change permissions of directory "/var/lib/postgresql/data": Operation not permitted
```
Pod u `CrashLoopBackOff`, `Exit Code: 1`.

**Uzrok:** Nov, prazan PVC je po defaultu vlasništvo roota. Postgres sad radi kao korisnik 999 (ne root) i ne može sam promijeniti vlasništvo direktorija koji mu ne pripada.

**Prvi pokušaj popravka — `fsGroup` (nije upalio):**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 999
  fsGroup: 999
```
`fsGroup` je  K8s mehanizam koji treba promijeniti grupno vlasništvo montiranog volumena prije starta kontejnera. Nakon primjene, `kubectl logs` na novom Podu pokazuje **identičnu** grešku — potvrđeno da `fsGroup` sam po sebi nije dovoljan, barem ne kod nas.

### Korak 3 — Init Container kao stvarno rješenje

**Popravak:** Dodan Init Container koji, kao root, ispravi vlasništvo direktorija prije nego se glavni (non-root) postgres kontejner uopće pokrene:
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 999
    fsGroup: 999
  initContainers:
    - name: fix-permissions
      image: busybox
      command: ["sh", "-c", "chown -R 999:999 /var/lib/postgresql/data"]
      volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      securityContext:
        runAsUser: 0
        runAsNonRoot: false
  containers:
    - name: postgres
      ...
```
## Problem 3: Tablica ticket_orders ne postoji

**Simptom:**
```json
{"error": "Unable to read orders", "details": "relation \"ticket_orders\" does not exist"}
```

**Dijagnoza:** Baza radi, ali tablica u koju worker sprema narudžbe nikad nije kreirana.

**Uzrok:** Docker Compose je koristio bind mount da Postgres image automatski izvrši `init.sql`. Kubernetes Deployment nikad nije imao takav mehanizam.

**Popravak:**
```yaml
volumeMounts:
  - name: postgres-init
    mountPath: /docker-entrypoint-initdb.d
volumes:
  - name: postgres-init
    configMap:
      name: postgres-init
```
Pošto Postgres izvršava init skripte samo pri prvom pokretanju, potrebno je i obrisati/rekreirati PVC:
```bash
kubectl delete deployment postgres
kubectl delete pvc postgres-pvc
kubectl apply -f k8s/postgres/pvc.yaml
kubectl apply -f k8s/postgres/deployment.yaml
```

**Provjera:**
```bash
kubectl exec -it deployment/postgres -- psql -U ticketing_user -d ticketing -c "\dt"
```
Tablica `ticket_orders` prisutna; kupnja karte rezultira zapisom sa statusom `"processed"`.

---

## Problem 4: CI pada na pushu slike — "unknown blob"

**Simptom:** CI job pada na koraku "Tag and push images", exit code 1, nakon uspješnog builda i čistog Trivy skena. Zadnja poruka u logu:
```
unknown blob
Error: Process completed with exit code 1.
```

**Dijagnoza:** Pregled cijelog loga tog koraka pokazuje da su `ticketing-api` i `ticketing-worker` uspješno pushani (svi slojevi `Pushed` ili `Layer already exists`), ali `ticketing-frontend` staje usred pusha — jedan sloj nikad ne dobije potvrdu prije nego se pojavi `unknown blob`.

**Uzrok:** Docker koristi cross-repo layer mounting — kad tri slike dijele zajedničke slojeve (istu `node:24-alpine` osnovu), pokušava ih "posuditi" (`Mounted from ...`) između repozitorija umjesto ponovnog pushanja, radi brzine. Kad se tri slike pushaju brzo jedna za drugom u istoj petlji, GHCR povremeno ne uspije ispravno dovršiti to posuđivanje za treću sliku — prolazna (tranzijentna) greška registra, ne problem u konfiguraciji ili kodu.

**Popravak:** Nema izmjene koda. Na GitHub Actions stranici neuspjelog run-a, "Re-run failed jobs".

**Provjera:** Identičan commit, isti kod, ponovno pokrenut job prolazi zeleno — potvrđuje da je uzrok bio privremen, ne strukturan.

---

## Problem 5: Worker pada s ECONNREFUSED

**Simptom:**
```
Worker fatal error: Error: connect ECONNREFUSED 172.18.0.3:5432
```
Worker kontejner ostaje `Exited (1)`, ostali servisi rade normalno.

**Dijagnoza:**
```bash
docker compose ps -a
docker compose logs worker
```

**Uzrok:** Worker se na startu odmah pokušava spojiti na PostgreSQL. Kontejner baze je tehnički pokrenut, ali sam proces baze još nije spreman primati konekcije — race condition. Worker nema retry logiku, pa se odmah ugasi.

**Popravak:**
```yaml
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
    interval: 5s
    timeout: 5s
    retries: 5

worker:
  depends_on:
    postgres:
      condition: service_healthy
```

**Provjera:**
```bash
docker compose up --build
```
Worker se pokreće tek nakon što je postgres zaista spreman, bez pucanja.

---

## Problem 6: Trivy sken obara CI pipeline

**Simptom:** GitHub Actions sagradi sliku, ali padne na koraku "Scan api image" s exit code 1. Tablica pokazuje 14 HIGH ranjivosti.

**Dijagnoza:** Otvoren failing job → tablica ranjivosti pokazuje 2 nalaza u `libcrypto3`/`libssl3` (Alpine OS paketi) i 12 nalaza unutar `npm`-ovih internih ovisnosti (`cross-spawn`, `glob`, `minimatch`, `tar`, `sigstore`).

**Uzrok:** Bazna `node:XX-alpine` slika dolazi s `npm` alatom koji ima svoje vlastite, zastarjele ovisnosti. Nijedan nalaz nije u vlastitim ovisnostima aplikacije.

**Popravak:**
```dockerfile
RUN npm ci --omit=dev
RUN apk upgrade --no-cache && \
    rm -rf /usr/local/lib/node_modules/npm \
           /usr/local/lib/node_modules/corepack \
           /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack
```

**Provjera:** CI ponovno prolazi zeleno; Trivy izvještaj pokazuje 0 nalaza te ozbiljnosti.

---

## Problem 7: PVC zaglavljen u Pending

**Simptom:** Nakon izmjene Deploymenta (dodavanje volumena), dva postgres Poda postoje istovremeno — stari `Running`, novi `Pending`, unedogled.

**Dijagnoza:**
```bash
kubectl get pvc
```
PVC ostaje `Pending`.

**Uzrok:** `ReadWriteOnce` PVC dopušta pristup samo jednom Podu istovremeno. Zadana `RollingUpdate` strategija pokušava prvo pokrenuti novi Pod, tek onda ugasiti stari — ali stari Pod još drži volume, pa novi ne može startati.

**Popravak:**
```yaml
spec:
  strategy:
    type: Recreate
```

**Provjera:**
```bash
kubectl get pods
kubectl get pvc
```
Samo jedan postgres Pod; PVC status `Bound`.

---

## Problem 8: Ingress vraća pogrešan sadržaj

**Simptom:** I `/healthz` i `/api/healthz` vraćaju istu HTML stranicu frontenda; frontend forma javlja grešku `Unexpected token '<'... is not valid JSON`.

**Dijagnoza:** Provjeren `rewrite-target: /$2` u odnosu na regex uzorke oba pravila u `ingress.yaml`.

**Uzrok:** Pravilo za `/api` ima dvije hvatne regex-grupe, pa `$2` ispravno referencira drugu grupu. Pravilo za frontend imalo je samo jednu grupu `$2` je za njega uvijek prazan, pa se svaka putanja prepisivala u prazno (`/`).

**Popravak:**
```yaml
- path: /()(.*)
```
Dodana prazna grupa da broj grupa bude usklađen.

**Provjera:**
```bash
curl.exe http://localhost/healthz          # {"status":"ok","service":"frontend"}
curl.exe http://localhost/api/healthz      # {"status":"ok","service":"api"}
ili direktno kroz browser
```

---

## Problem 9: Loš image tag pri deployu + demonstracija rolling update i rollback

**Simptom:** Nakon `kubectl set image` s nepstojećim tagom, novi pod se ne diže.

**Baseline prije promjene:**
```bash
kubectl get pods -l app=api
```
```
NAME                   READY   STATUS    RESTARTS   AGE
api-7d5ccbcbbf-2nt65   1/1     Running   0          26s
api-7d5ccbcbbf-zhbzf   1/1     Running   0          13s
```

**Pokvareni deploy:**
```bash
kubectl set image deployment/api api=ghcr.io/ailaephos/uvod-u-dev-ops/ticketing-api:krivi
kubectl get pods -l app=api
```
```
NAME                   READY   STATUS         RESTARTS   AGE
api-65ddc7958c-sp4r9   1/1     Running        0          2m18s
api-65ddc7958c-xswbk   1/1     Running        0          2m30s
api-bd6cb9cbc-dcmrx    0/1     ErrImagePull   0          25s
```
Stara dva poda ostaju up RollingUpdate ne gasi stare replike dok nova nije potvrđeno `Ready` (readinessProbe).

**Dijagnoza:**
```bash
kubectl describe pod api-bd6cb9cbc-dcmrx
```
Events sekcija:
```
Warning  Failed  kubelet  Failed to pull image "...ticketing-api:krivi": rpc error: code = NotFound
desc = failed to resolve reference "...ticketing-api:krivi": ...ticketing-api:krivi: not found
```

**Uzrok:** Referenciran tag koji ne postoji na GHCR registryju.

**Popravak, rollback na prijašnju:**
```bash
kubectl rollout undo deployment/api
kubectl rollout status deployment/api
```
```
deployment "api" successfully rolled out
```

**Provjera:**
```bash
kubectl get pods -l app=api
```
```
NAME                   READY   STATUS    RESTARTS   AGE
api-65ddc7958c-sp4r9   1/1     Running   0          3m17s
api-65ddc7958c-xswbk   1/1     Running   0          3m29s
```
Isti podovi,nikad nisu bili gašeni tijekom cijelog incidenta, nula downtimea.
```bash
curl http://localhost/api/healthz
# {"status":"ok","service":"api"}
```

Sinkronizacija manifesta s klasterom nakon rollbacka (izbjegava drift upozorenje kod idućeg `kubectl apply`):
```bash
kubectl apply -f k8s/api/deployment.yaml
```

**Dodatna napomena, pravi (uspješan) rolling update:** Prije nego je simuliran ovaj incident, `kubectl rollout restart deployment/api` je demonstrirao standardni rolling update: novi Pod je čekao readinessProbe prije nego je stari ugašen (`1 od 2 nova replika ažurirana` → `1 stari replika čeka gašenje` → `successfully rolled out`), bez ijednog trenutka nedostupnosti servisa.

---