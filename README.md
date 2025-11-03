# Postgres-KCMM
# 🐘 PostgreSQL Multi-Master Replication on Kubernetes

This project deploys **two PostgreSQL 16 instances** (`pg-a` and `pg-b`) inside a Kubernetes cluster with **bi-directional logical replication** (multi-master setup).  
It provides automatic configuration of:
- Databases & roles
- Publications & subscriptions
- Sequences for odd/even ID handling
- Sync validation between both nodes

## 🚀 Features

✅ Fully automated setup using a Kubernetes `Job`  
✅ Two PostgreSQL nodes (`pg-a`, `pg-b`) communicating via ClusterIP Services  
✅ Logical replication (insert/update/delete) both ways  
✅ Customizable tables for replication (via `ConfigMap`)  
✅ Ready-to-use test table: `public.test_mm`

## 🧱 Architecture

pg-a  <——logical replication——>  pg-b  
  │                                  │  
  └——— publication: pub_app          └——— publication: pub_app  
       subscription: sub_from_b           subscription: sub_from_a

Both nodes replicate the same `appdb.public.test_mm` table using PostgreSQL’s built-in logical replication system.

## 📦 Components

| Resource | Description |
|-----------|-------------|
| **Namespace:** `pg-mm` | Dedicated namespace for the deployment |
| **StatefulSets:** `pg-a`, `pg-b` | PostgreSQL 16 Pods (persistent) |
| **Services:** `pg-a`, `pg-b` | Headless services for inter-node access |
| **Secret:** `pg-secrets` | Contains credentials (`postgres/postgres`, `replicator/replicator`) |
| **ConfigMap:** `pg-conf` | PostgreSQL config (`wal_level=logical`) |
| **ConfigMap:** `setup-sql` | Initialization + replication setup script |
| **Job:** `setup-mm` | Automatically configures replication between both nodes |

## ⚙️ Deployment

### 1️⃣ Apply all manifests
kubectl apply -f pg-mm-all.yaml

Wait for pods to be ready:
kubectl -n pg-mm get pods -w

Then monitor setup logs:
kubectl -n pg-mm logs job/setup-mm -f

## 🧪 Testing Replication

### Test insert on `pg-a`
kubectl -n pg-mm exec -it statefulset/pg-a -- bash -lc "export PGPASSWORD=postgres; psql -X -h 127.0.0.1 -U postgres -d appdb -c \"INSERT INTO public.test_mm(payload) VALUES ('A -> B test') RETURNING *;\""

Then check on `pg-b`:
kubectl -n pg-mm exec -it statefulset/pg-b -- bash -lc "export PGPASSWORD=postgres; psql -X -h 127.0.0.1 -U postgres -d appdb -c \"TABLE public.test_mm;\""

✅ You should see the same record replicated!

### Test reverse direction (`pg-b` → `pg-a`)
kubectl -n pg-mm exec -it statefulset/pg-b -- bash -lc "export PGPASSWORD=postgres; psql -X -h 127.0.0.1 -U postgres -d appdb -c \"INSERT INTO public.test_mm(payload) VALUES ('B -> A test') RETURNING *;\""

Then check on `pg-a` again:
kubectl -n pg-mm exec -it statefulset/pg-a -- bash -lc "export PGPASSWORD=postgres; psql -X -h 127.0.0.1 -U postgres -d appdb -c \"TABLE public.test_mm;\""

## 💻 Local Access (via Port Forward)

To connect using a local SQL client (like DBeaver, DataGrip, TablePlus):

kubectl -n pg-mm port-forward statefulset/pg-a 5433:5432

Connection Info:
Host: 127.0.0.1  
Port: 5433  
Database: appdb  
Username: postgres  
Password: postgres

Repeat for pg-b if needed:
kubectl -n pg-mm port-forward statefulset/pg-b 5434:5432

## 🧹 Cleanup

To remove everything:
kubectl delete ns pg-mm

## 🧠 Notes

- PostgreSQL 16 supports fully symmetrical logical replication.
- Sequence offsets (+2) prevent ID collisions.
- To add new replicated tables:
  ALTER PUBLICATION pub_app ADD TABLE public.your_table;
  Then refresh subscriptions:
  ALTER SUBSCRIPTION sub_from_a REFRESH PUBLICATION WITH (copy_data = false);
  ALTER SUBSCRIPTION sub_from_b REFRESH PUBLICATION WITH (copy_data = false);

## 🧩 Author
**@yourgithubusername**  
Automated PostgreSQL bi-directional replication on Kubernetes 🚀
