# Postgres backups

The SBOM dashboard's database lives on the droplet's disk, so it is backed up every night to a private DigitalOcean Spaces bucket (`jsundquist-k8s-backups`, `sfo3`, Standard storage).

## How it works

`apps/sbom/backup.yaml` is a CronJob (03:17 UTC) with two steps sharing a scratch `emptyDir`:

1. **dump**: an init container (`postgres:16`) runs `pg_dump --format=custom` into `/work`.
2. **upload**: an `rclone` container copies it to the bucket through a `crypt` remote, so file contents and names are encrypted before they leave the cluster. It then deletes backups older than 14 days.

Credentials come from the Sealed Secret `sbom-backup`. The CronJob refuses to run concurrently, and keeps the last three finished Jobs for debugging.

## Decisions

- **Standard storage, not Cold.** Cold has a 30-day minimum retention, which conflicts with pruning at 14 days, and these dumps are small enough that the price difference is negligible.
- **rclone crypt rather than age.** One official image does encryption, upload and pruning, so there is no custom image to build. The cost is that encryption is symmetric: the cluster holds the key, so it protects against a leaked bucket, not against someone with cluster access. Switch to `age` (public-key) if that threat matters.
- **Dedicated Spaces key** limited to this bucket where possible. Do not reuse a personal key.
- **Custom format dumps** (`pg_dump -Fc`) are compressed and restore selectively with `pg_restore`.
- **Retention is the job's own pruning** (14 days). There is no bucket lifecycle rule: the Spaces console has no UI for them (only the API or `s3cmd`), and they have been reported to be unreliable. The job logs the bucket's total size every night, and a glance at it every month or two catches a pruning failure.

## Setup

1. Create a Spaces access key (API, Spaces Keys) in the DigitalOcean console. Keep the secret out of chat and out of git.
2. Seal the secret. `brew install rclone` is needed locally, for `rclone obscure`.

```sh
export KUBECONFIG=~/.kube/k8s-droplet.yaml
printf '\e[?2004l'              # disable bracketed paste, so pastes arrive clean
read -rs AK; echo ${#AK}         # paste the access key id, then Enter
read -rs SK; echo ${#SK}         # paste the secret key, then Enter
PASS=$(openssl rand -base64 32); SALT=$(openssl rand -base64 32)
echo "$PASS"; echo "$SALT"       # save BOTH in a password manager now: they are required to restore

kubectl create secret generic sbom-backup -n sbom --dry-run=client -o yaml \
  --from-literal=access-key-id="$AK" \
  --from-literal=secret-access-key="$SK" \
  --from-literal=crypt-password="$(rclone obscure "$PASS")" \
  --from-literal=crypt-password2="$(rclone obscure "$SALT")" \
  | kubeseal --format yaml > apps/sbom/sealed-backup.yaml
unset AK SK PASS SALT
```

3. Uncomment `sealed-backup.yaml` and `backup.yaml` in `apps/sbom/kustomization.yaml`, then commit and push.
4. Run it once by hand instead of waiting for 03:17:

```sh
kubectl create job --from=cronjob/postgres-backup backup-test -n sbom
kubectl logs -n sbom job/backup-test -c upload -f
kubectl delete job backup-test -n sbom
```

The log ends with the bucket's total object count and size.

## Restore (and test it)

A backup you have never restored is a hope. Test it once, into a scratch database and never over the live one.

```sh
# 1. Configure rclone for the bucket and decryption, from env vars (obscured passwords).
export RCLONE_CONFIG_BK_TYPE=s3 RCLONE_CONFIG_BK_PROVIDER=DigitalOcean \
  RCLONE_CONFIG_BK_ENDPOINT=sfo3.digitaloceanspaces.com RCLONE_CONFIG_BK_NO_CHECK_BUCKET=true
read -rs RCLONE_CONFIG_BK_ACCESS_KEY_ID; read -rs RCLONE_CONFIG_BK_SECRET_ACCESS_KEY
export RCLONE_CONFIG_BK_ACCESS_KEY_ID RCLONE_CONFIG_BK_SECRET_ACCESS_KEY
export RCLONE_CONFIG_ENC_TYPE=crypt RCLONE_CONFIG_ENC_REMOTE=bk:jsundquist-k8s-backups/postgres
read -rs P; export RCLONE_CONFIG_ENC_PASSWORD=$(rclone obscure "$P")
read -rs S; export RCLONE_CONFIG_ENC_PASSWORD2=$(rclone obscure "$S")

# 2. List, then fetch the newest dump.
rclone ls enc:
rclone copyto enc:<filename>.dump ./sbom.dump

# 3. Restore into a throwaway Postgres.
docker run -d --name restore-test -e POSTGRES_PASSWORD=test -p 55432:5432 postgres:16
sleep 5
pg_restore --no-owner -h localhost -p 55432 -U postgres -d postgres --create ./sbom.dump
docker exec restore-test psql -U postgres -d sbom -c '\dt'
docker rm -f restore-test; rm ./sbom.dump
```

To restore onto the cluster after data loss, scale `web` and `worker` to zero, restore into the `sbom` database, then scale them back up.

## Known limitations

- The backup is a logical dump taken while the database is live, which is consistent but not point-in-time. Anything written after 03:17 UTC is lost in a restore.
- A failed backup is visible only in the Job history (`kubectl get jobs -n sbom`). Alerting on it belongs with the observability work.
- Cluster NetworkPolicies, once added, must allow this Job to reach `postgres:5432` and the Spaces endpoint over 443.
