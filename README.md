1. Backup infrastructure

 infra/modules/dr/main.tf
resource "aws_backup_vault" "main" {
  name        = "${var.app_name}-${var.environment}-vault"
  kms_key_arn = var.kms_key_arn
  tags        = { Environment = var.environment }
}

resource "aws_backup_plan" "main" {
  name = "${var.app_name}-${var.environment}-backup"

  rule {
    rule_name         = "hourly"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 * * * ? *)"
    lifecycle         { delete_after = 7 }
  }

  rule {
    rule_name         = "daily"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 2 * * ? *)"
    lifecycle         { delete_after = 30 }
    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn
      lifecycle             { delete_after = 30 }
    }
  }

  rule {
    rule_name         = "weekly"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 3 ? * 1 *)"
    lifecycle         { delete_after = 90 }
    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn
      lifecycle             { delete_after = 90 }
    }
  }
}

resource "aws_backup_vault" "dr" {
  provider    = aws.dr_region
  name        = "${var.app_name}-${var.environment}-dr-vault"
  kms_key_arn = var.dr_kms_key_arn
  tags        = { Environment = var.environment }
}

resource "aws_backup_selection" "main" {
  name         = "${var.app_name}-${var.environment}"
  iam_role_arn = aws_iam_role.backup.arn
  plan_id      = aws_backup_plan.main.id
  resources    = [var.db_arn, var.s3_document_arn, var.s3_archive_arn]
}

resource "aws_iam_role" "backup" {
  name = "${var.app_name}-${var.environment}-backup"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "backup.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "backup" {
  role       = aws_iam_role.backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup"

  terraform apply -var-file="terraform.tfvars"

2. DB backup + restore scripts

!/bin/bash
 scripts/backup-db.sh
set -euo pipefail

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
FILE="placemux-${ENVIRONMENT}-${TIMESTAMP}.sql.gz"
BUCKET="placemux-${ENVIRONMENT}-backups"

pg_dump $DATABASE_URL | gzip > /tmp/$FILE
aws s3 cp /tmp/$FILE s3://$BUCKET/db/$FILE --sse aws:kms
echo "{\"backup\":\"$FILE\",\"ts\":\"$TIMESTAMP\"}"
rm /tmp/$FILE
}

!/bin/bash
scripts/restore-db.sh
set -euo pipefail

FILE=$1
BUCKET="placemux-${ENVIRONMENT}-backups"

echo "restoring from $FILE"
aws s3 cp s3://$BUCKET/db/$FILE /tmp/restore.sql.gz
dropdb   --if-exists placemux_restore
createdb placemux_restore
gunzip -c /tmp/restore.sql.gz | psql $DATABASE_URL
echo "restore complete — verify before promoting"
rm /tmp/restore.sql.gz

kubectl apply -f k8s/production/backup-cronjob.yaml

3. RTO/RPO targets

 docs/dr-targets.md
| Scenario       | RPO   | RTO    |
|----------------|-------|--------|
| Pod failure    | 0     | <30s   |
| Node failure   | 0     | <60s   |
| DB failure     | 1hr   | <30min |
| Region failure | 24hr  | <4hr   |
| Full data loss | 24hr  | <8hr   |

4. DR runbooks

!/bin/bash
 scripts/dr-pod-failure.sh
set -euo pipefail

echo "=== pod failure recovery ==="
FAILED=$(kubectl get pods -n production \
  --field-selector=status.phase=Failed \
  -o jsonpath='{.items[*].metadata.name}')

for pod in $FAILED; do
  kubectl delete pod $pod -n production
done

kubectl get pods -n production -w &
sleep 60; kill %1

curl -sf https://prod.placemux.com/health | jq .

!/bin/bash
 scripts/dr-node-failure.sh
set -euo pipefail

NODE=$1
echo "=== node failure: $NODE ==="

kubectl cordon $NODE
kubectl drain $NODE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=30

kubectl get pods -n production -o wide -w &
sleep 90; kill %1

curl -sf https://prod.placemux.com/health | jq .
kubectl uncordon $NODE

!/bin/bash
 scripts/dr-region-failure.sh
set -euo pipefail

DR_REGION="ap-southeast-1"
echo "=== region failover to $DR_REGION ==="

LATEST=$(aws s3 ls s3://placemux-prod-backups/db/ \
  --region $DR_REGION | sort | tail -1 | awk '{print $4}')

aws rds restore-db-instance-from-s3 \
  --db-instance-identifier placemux-dr \
  --db-name placemux \
  --s3-bucket-name placemux-prod-backups \
  --s3-prefix db/$LATEST \
  --region $DR_REGION

aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"prod.placemux.com\",
        \"Type\": \"CNAME\",
        \"TTL\": 60,
        \"ResourceRecords\": [{\"Value\": \"$DR_ALB_DNS\"}]
      }
    }]
  }"

echo "failover complete"

5. DR alerts

 k8s/production/dr-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dr-alerts
  namespace: production
spec:
  groups:
    - name: dr
      rules:
        - alert: BackupJobFailed
          expr: kube_job_status_failed{namespace="production",job_name=~"db-backup.*"} > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "DB backup job failed"
            runbook: "docs/dr-runbook.md#backup-failed"
        - alert: BackupMissed
          expr: |
            time() - kube_job_status_completion_time{job_name=~"db-backup.*"} > 7200
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "No successful backup in 2hrs"
            runbook: "docs/dr-runbook.md#backup-missed"
        - alert: ReplicaLagHigh
          expr: aws_rds_replica_lag_average > 300
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "DB replica lag >5min"
            runbook: "docs/dr-runbook.md#replica-lag"

  kubectl apply -f k8s/production/dr-alerts.yaml

  6. Demo

!/bin/bash
scripts/dr-demo.sh
set -euo pipefail

echo "=== 1. verify backups ==="
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name placemux-prod-vault \
  --query 'RecoveryPoints[0:3].{status:Status,created:CreationDate}' | jq .
aws s3 ls s3://placemux-prod-backups/db/ | tail -5

echo "=== 2. pod failure ==="
POD=$(kubectl get pods -n production -l app=my-app \
  -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD -n production
sleep 5
kubectl get pods -n production -w &
sleep 40; kill %1
curl -sf https://prod.placemux.com/health | jq .

echo "=== 3. node failure ==="
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
bash scripts/dr-node-failure.sh $NODE
curl -sf https://prod.placemux.com/health | jq .

echo "=== 4. restore test ==="
LATEST=$(aws s3 ls s3://placemux-prod-backups/db/ \
  | sort | tail -1 | awk '{print $4}')
bash scripts/restore-db.sh $LATEST

kubectl exec -it <db-pod> -n production -- psql -U placemux placemux_restore -c \
  "SELECT COUNT(*) FROM orders;"
kubectl exec -it <db-pod> -n production -- psql -U placemux placemux_restore -c \
  "SELECT COUNT(*) FROM tenants;"

echo "=== 5. replica lag ==="
aws rds describe-db-instances \
  --db-instance-identifier placemux-prod-replica \
  --query 'DBInstances[0].StatusInfos' | jq .

echo "=== 6. trigger backup now ==="
kubectl create job backup-now --from=cronjob/db-backup -n production
kubectl logs -f job/backup-now -n production

echo "=== 7. alerts ==="
curl -s http://prometheus:9090/api/v1/alerts \
  | jq '.data.alerts[] | select(.labels.alertname | startswith("Backup")) | {name:.labels.alertname,state}'

  bash scripts/dr-demo.sh

  Expected output:

backups         3 recovery points COMPLETED, hourly .sql.gz in S3
pod failure     new pod running in <30s, 200 OK
node failure    pods rescheduled in <60s, 200 OK
restore         orders + tenants counts match prod
replica lag     <5s
backup job      {"backup":"placemux-prod-xxx.sql","ts":"..."}
alerts          BackupMissed fires if job skipped

<img width="1536" height="1024" alt="ss3014" src="https://github.com/user-attachments/assets/94d6d0ae-a6d6-4601-8dc6-8dcba9cfc29e" />
