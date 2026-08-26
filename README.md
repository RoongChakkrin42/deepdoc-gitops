# deepdoc-gitops

The desired state of the DeepDoc cluster. ArgoCD reconciles the cluster against
this repository; nothing is deployed by hand after the one-time bootstrap below.

- [`deepdoc`](https://github.com/RoongChakkrin42/deepdoc) — the NestJS API
- [`deepdoc_client`](https://github.com/RoongChakkrin42/deepdoc_client) — the Next.js frontend

> 🇹🇭 [สรุปภาษาไทยอยู่ท้ายไฟล์](#สรุปภาษาไทย)

```
 merge to master/main
        │
   GitHub Actions ─ test ─ build (arm64) ─▶ ghcr.io/…:sha-abc1234
        │
        └─ kustomize edit set image ──▶ THIS REPO (a commit)
                                              │
                                        ArgoCD polls, every 3 min
                                              ▼
              ╔═══════════ Oracle ARM VM · k3s ═══════════╗
              ║  Traefik   https://deepdoc.<ip>.nip.io    ║
              ║    /      → web   (Next, replicas 2)      ║
              ║    /api   → api   (Nest, replicas 1)      ║
              ║  https://files.<ip>.nip.io → minio        ║
              ║  mongodb · minio   (StatefulSet + PVC)    ║
              ╚═══════════════════════════════════════════╝
```

## Layout

```
bootstrap/root-app.yaml   the only thing ever applied by hand
apps/                     four Argo Applications, ordered by sync-wave
manifests/data/           wave 0 — MongoDB, MinIO, bucket job
manifests/api/            wave 1 — kustomization.yaml is bumped by deepdoc CI
manifests/web/            wave 1 — kustomization.yaml is bumped by deepdoc_client CI
manifests/ingress/        wave 2 — Traefik routing and TLS
```

There is no `base/` + `overlays/`. With one cluster and one environment an
overlay containing only an `images:` block doubles the file count and adds an
indirection to every debugging session. Directory-per-component buys the thing
that actually matters here: **each CI pipeline owns a different
`kustomization.yaml`, so the two repos can never race on the same file.**

---

## Bootstrap

Once, in this order. Steps 1–3 are the ones that most often go wrong.

### 1. The VM

Oracle Cloud **Always Free**: `VM.Standard.A1.Flex`, 4 OCPU / 24 GB, Ubuntu
24.04 (aarch64).

- `Out of host capacity` on A1 is the normal experience, not a bug. Retry with
  the `oci` CLI in a loop rather than clicking. A less popular home region
  helps — and **the home region is permanent**, so it is the one irreversible
  choice here.
- The 4 OCPU / 24 GB quota is shared across *all* A1 instances, so one VM of
  this size uses the whole allowance.
- **Set the boot volume to ~100 GB at creation.** `local-path` PVCs live on it
  and the 47 GB default gets tight. Resizing afterwards is possible but fiddly.

### 2. Open 80 and 443 — in *both* firewalls

Fixing only one half produces exactly the same symptom as fixing neither (a
connection timeout), which is why this eats afternoons.

**a. OCI Security List / NSG** — VCN → Subnet → Security List → add ingress
rules: source `0.0.0.0/0`, TCP, destination ports 80 and 443.

**b. The instance's own iptables.** Oracle's images ship a populated `INPUT`
chain that ends in a REJECT, so appending with `-A` puts your rule *after* the
REJECT where it does nothing. Insert instead:

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80  -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo iptables -L INPUT --line-numbers        # confirm both precede the REJECT
sudo netfilter-persistent save
```

### 3. k3s

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="\
  --write-kubeconfig-mode 644 --tls-san <PUBLIC_IP>" sh -
kubectl get nodes
```

Keep every default: Traefik is the ingress controller, `local-path` is the
storage class, and klipper-lb is what binds Traefik to the host's :80/:443.

To drive the cluster from a laptop, copy `/etc/rancher/k3s/k3s.yaml` and replace
`127.0.0.1` with the public IP — `--tls-san` is what makes that certificate
valid.

### 4. Traefik — two overrides that both matter

k3s manages Traefik through a HelmChart, so override it by dropping a file into
the auto-deploy directory. k3s picks it up and redeploys on its own.

`/var/lib/rancher/k3s/server/manifests/traefik-config.yaml`:

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    service:
      spec:
        externalTrafficPolicy: Local
    ports:
      web:
        transport:
          respondingTimeouts:
            readTimeout: 600s
      websecure:
        transport:
          respondingTimeouts:
            readTimeout: 600s
```

- `externalTrafficPolicy: Local` stops kube-proxy rewriting the source address.
  Together with `trust proxy` in the API, it is what makes the throttler's
  per-IP limits actually per-IP. Without it, `req.ip` is one constant value for
  the entire internet and the 5-uploads-per-hour cap applies globally — on an
  endpoint that is unauthenticated by design with rate limiting as its only
  protection.
- Traefik v3 defaults `readTimeout` to 60 s and that covers reading the whole
  request body, while the client deliberately allows ten minutes for an upload.
  Without this, a large PDF over a slow connection 408s.

### 5. cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
```

Create **two** ClusterIssuers, `letsencrypt-staging` and `letsencrypt-prod`,
both using `http01` with `ingress: {class: traefik}`.

**Start on staging.** Let's Encrypt allows 5 failed validations per hour in
production; a misconfigured Ingress burns that in ten minutes and then locks you
out while you are still iterating. Switch the annotation in
`manifests/ingress/` to `letsencrypt-prod` only after a staging certificate
issues, then delete the staging TLS Secrets to force reissuance.

### 6. ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Change the admin password immediately. **Do not expose the Argo UI publicly** —
an internet-facing ArgoCD on a default password is a full cluster takeover and a
well-scanned target. `port-forward` costs nothing.

### 7. Set the hostname

Every manifest carries the placeholder `IP-IP-IP-IP`. Replace it with the VM's
public IP, dashes instead of dots (`nip.io` resolves those to the address):

```bash
IP_DASHED=$(echo "<PUBLIC_IP>" | tr . -)
grep -rl 'IP-IP-IP-IP' manifests | xargs sed -i '' "s/IP-IP-IP-IP/${IP_DASHED}/g"   # macOS
grep -rl 'IP-IP-IP-IP' manifests | xargs sed -i    "s/IP-IP-IP-IP/${IP_DASHED}/g"   # Linux
git commit -am "Point at the cluster's public address"
```

Three places depend on it and must agree: the two Ingress hosts, `CORS_ORIGINS`,
and `S3_PUBLIC_ENDPOINT`.

### 8. Secrets

Nothing secret is committed here. Create the three Secrets on the cluster
directly — see `manifests/api/secret.example.yaml` for the key names.

```bash
kubectl create namespace deepdoc

kubectl -n deepdoc create secret generic minio-credentials \
  --from-literal=MINIO_ROOT_USER='deepdoc' \
  --from-literal=MINIO_ROOT_PASSWORD='<generate one>'

kubectl -n deepdoc create secret generic mongodb-credentials \
  --from-literal=MONGO_INITDB_ROOT_USERNAME='deepdoc' \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD='<generate one>'

kubectl -n deepdoc create secret generic deepdoc-api-secrets \
  --from-literal=MONGODB_URI='mongodb://deepdoc:<mongo password>@mongodb-0.mongodb.deepdoc.svc.cluster.local:27017/deepdoc?authSource=admin' \
  --from-literal=JWT_SECRET="$(openssl rand -base64 32)" \
  --from-literal=GEMINI_API_KEY='<your key>' \
  --from-literal=AWS_ACCESS_KEY_ID='deepdoc' \
  --from-literal=AWS_SECRET_ACCESS_KEY='<the same minio password>'
```

Two couplings that break things silently if they drift:

- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` must equal `MINIO_ROOT_USER` /
  `MINIO_ROOT_PASSWORD`.
- `MONGO_INITDB_ROOT_*` only takes effect on an **empty** data directory. Get
  them right before the first apply, or adding auth later means an interactive
  `db.createUser` or deleting the PVC.

> Migrating to Sealed Secrets later is drop-in: the Deployment refers to these
> by name either way, so nothing in `manifests/` changes.

### 9. Images

Merge to `master` / `main` in both app repos so CI publishes the first images.
**Then set both GHCR packages to Public** — a new package is private by default
even when the repository is public, and the symptom is `ImagePullBackOff`.

CI will also have pinned the real tags in the two `kustomization.yaml` files.

### 10. Launch

```bash
kubectl apply -f bootstrap/root-app.yaml
kubectl -n deepdoc get pods -w
```

### 11. Seed the reviewer account

`/results` is behind a JWT and nothing creates an account automatically. Edit
the password in `manifests/api/seed-reviewer-job.yaml`, then:

```bash
kubectl apply -f manifests/api/seed-reviewer-job.yaml
kubectl -n deepdoc logs job/seed-reviewer
kubectl -n deepdoc delete job seed-reviewer
```

That file is deliberately *not* in `kustomization.yaml`: Argo neither applies
nor prunes it, and it is not a sync hook because registering an existing
username throws a conflict, which would fail every sync after the first.

---

## Verifying

```bash
kubectl -n deepdoc get pods
kubectl -n deepdoc exec minio-0 -- mc ls local/deepdoc          # bucket exists
kubectl -n deepdoc port-forward svc/api 8000:8000 &
curl -sf localhost:8000/readyz                                  # API sees Mongo
```

**The one command that proves the ingress and the strip-prefix middleware:**

```bash
curl -s https://deepdoc.<ip>.nip.io/api/submissions/form-schema | head -c 200
```

A JSON rubric means the middleware annotation is right. A 404 whose body echoes
`"path":"/api/submissions/form-schema"` means the annotation format is wrong —
it is `deepdoc-strip-api@kubernetescrd`, with a hyphen, not a slash.

Then, end to end: open the site, upload a PDF, watch
`kubectl -n deepdoc logs -f deploy/api` for `Submission <id> scored N/100`, log
in as the seeded reviewer, and **click the report link** — that last step is the
specific test for `S3_PUBLIC_ENDPOINT`, because it is the only one that exercises
a presigned URL from a browser.

To see Argo actually owning the cluster:

```bash
kubectl -n deepdoc scale deploy api --replicas=3
kubectl -n deepdoc get deploy api -w      # reverted to 1 within ~3 minutes
```

---

## Why the API runs a single replica

`replicas: 1` and `strategy: Recreate` on the API are deliberate, not an
oversight.

On boot the API selects every submission still in `pending` or `processing` and
starts grading each one, and `markProcessing` updates the document with no
status precondition. A second pod therefore re-runs analyses the first is
already running: duplicated Gemini calls, and two writers racing on one
document. A rolling update creates exactly that overlap, so `Recreate` is used
instead and the cost is a few seconds of 502 per deploy.

Four changes in the API would be needed before this can scale: a conditional
claim in `markProcessing`, a lease reaper instead of the boot-time fan-out,
out-of-process throttler storage, and the same conditional claim in `retry()`.
Until then, do not add an HPA.

---

## สรุปภาษาไทย

repo นี้เก็บ **สถานะที่ต้องการของ cluster** ArgoCD จะคอยดึงจาก repo นี้ไปทำให้
cluster ตรงตามที่เขียนไว้ หลัง bootstrap ครั้งแรกแล้วจะไม่มีการ deploy ด้วยมืออีก
ทุกการเปลี่ยนแปลงคือการ `git push`

**เส้นทาง:** merge เข้า master/main → GitHub Actions build image ขึ้น GHCR →
CI แก้ tag ใน repo นี้ → ArgoCD เห็น commit แล้ว sync ลง k3s

**โครงสร้าง**

| โฟลเดอร์ | คืออะไร |
| --- | --- |
| `bootstrap/` | ไฟล์เดียวที่ apply ด้วยมือ และ apply แค่ครั้งเดียวตลอด |
| `apps/` | Argo Application 4 ตัว เรียงลำดับด้วย sync-wave |
| `manifests/data/` | MongoDB + MinIO (wave 0) |
| `manifests/api/` | API — `kustomization.yaml` ถูกแก้โดย CI ของ repo deepdoc |
| `manifests/web/` | frontend — `kustomization.yaml` ถูกแก้โดย CI ของ repo deepdoc_client |
| `manifests/ingress/` | Traefik routing + TLS (wave 2) |

ไม่ได้ใช้ `base/` + `overlays/` เพราะมี cluster เดียว environment เดียว การแยก
overlay ที่มีแค่ `images:` block ทำให้ไฟล์เพิ่มเป็นสองเท่าโดยไม่ได้อะไร ส่วนการ
แยกเป็นโฟลเดอร์ต่อ component ได้ประโยชน์จริงคือ **CI ของสอง repo แก้คนละไฟล์
จึงชนกันไม่ได้**

**ขั้นตอน bootstrap (ทำครั้งเดียว)** — รายละเอียดอยู่ในหัวข้อภาษาอังกฤษด้านบน

1. สร้าง VM (A1.Flex 4 OCPU / 24 GB, Ubuntu 24.04 ARM) — **boot volume ตั้ง 100 GB
   ตั้งแต่ตอนสร้าง** และ region ที่เลือกตอนสมัคร **เปลี่ยนทีหลังไม่ได้**
2. **เปิด port 80/443 ทั้งสองชั้น** — ทั้ง Security List ของ OCI และ iptables ในเครื่อง
   (ต้องใช้ `-I` ไม่ใช่ `-A` เพราะ chain จบด้วย REJECT) แก้ชั้นเดียวอาการจะเหมือน
   ไม่ได้แก้เลย
3. ลง k3s (ใช้ค่า default ทั้งหมด — Traefik, local-path, klipper-lb)
4. ตั้งค่า Traefik สองอย่าง: `externalTrafficPolicy: Local` (ให้ rate limit ทำงาน
   ต่อ IP จริง) และ `readTimeout: 600s` (ไม่งั้นอัปโหลดไฟล์ใหญ่จะ 408 ที่ 60 วิ)
5. ลง cert-manager แล้ว **เริ่มจาก staging issuer ก่อนเสมอ** — production ยอมให้
   validate พลาดได้ชั่วโมงละ 5 ครั้ง เกินแล้วโดนล็อก
6. ลง ArgoCD เปลี่ยนรหัส admin ทันที และ **ห้ามเปิด UI ออกอินเทอร์เน็ต**
7. แทนที่ `IP-IP-IP-IP` ในทุก manifest ด้วย IP ของ VM (เปลี่ยนจุดเป็นขีด)
8. สร้าง Secret 3 ตัวด้วยมือ — ระวังสองจุดที่ต้องตรงกัน: คีย์ AWS ต้องเท่ากับคีย์
   MinIO และรหัส Mongo ตั้งได้ครั้งเดียวตอน data directory ยังว่าง
9. merge เข้า master/main ให้ CI build image แล้ว **ตั้ง GHCR package เป็น Public**
10. `kubectl apply -f bootstrap/root-app.yaml`
11. รัน Job สร้างบัญชีผู้ตรวจ (แก้รหัสในไฟล์ก่อน)

**คำสั่งเดียวที่พิสูจน์ว่า ingress ถูกต้อง**

```bash
curl -s https://deepdoc.<ip>.nip.io/api/submissions/form-schema | head -c 200
```

ได้ JSON = ถูก ได้ 404 ที่บอก `"path":"/api/submissions/form-schema"` = annotation
ของ middleware ผิดรูปแบบ (ต้องเป็น `deepdoc-strip-api@kubernetescrd` ใช้ขีด ไม่ใช่ทับ)

**ทำไม API รัน replica เดียว** — ตอนบูต API จะไล่หา submission ที่ค้างแล้วสั่งตรวจใหม่
ทุกอัน และ `markProcessing` ไม่ได้เช็คสถานะก่อนเขียน ถ้ามี pod ที่สองขึ้นมาพร้อมกัน
มันจะตรวจซ้ำกับที่ pod แรกกำลังตรวจอยู่ = จ่ายค่า Gemini สองเท่า และเขียนทับกันเอง
จึงใช้ `Recreate` แทน rolling update ยอมให้ 502 สองสามวินาทีตอน deploy
