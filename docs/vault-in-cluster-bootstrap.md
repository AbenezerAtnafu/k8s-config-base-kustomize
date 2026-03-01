# Vault In-Cluster Bootstrap

This runbook bootstraps self-hosted HashiCorp Vault deployed in Kubernetes via `platform/argocd/app-vault.yaml`.

## 1) Deploy Vault and ESO

```bash
kubectl apply -k platform/argocd
kubectl -n vault get pods
kubectl -n external-secrets get pods
```

## 2) Initialize Vault (one-time)

```bash
kubectl -n vault exec -it vault-0 -- vault operator init -key-shares=1 -key-threshold=1
```

Save the output securely:

- `Unseal Key 1`
- `Initial Root Token`

## 3) Unseal Vault pods

```bash
kubectl -n vault exec -it vault-0 -- vault operator unseal <UNSEAL_KEY>
kubectl -n vault exec -it vault-1 -- vault operator unseal <UNSEAL_KEY>
kubectl -n vault exec -it vault-2 -- vault operator unseal <UNSEAL_KEY>
```

## 4) Login and enable secret/auth engines

```bash
kubectl -n vault exec -it vault-0 -- sh
export VAULT_ADDR=http://127.0.0.1:8200
vault login <ROOT_TOKEN>
vault secrets enable -path=kv kv-v2
vault auth enable kubernetes
exit
```

## 5) Configure Kubernetes auth

```bash
SA_SECRET="$(kubectl -n external-secrets get sa external-secrets -o jsonpath='{.secrets[0].name}')"
TOKEN="$(kubectl -n external-secrets get secret "$SA_SECRET" -o jsonpath='{.data.token}' | base64 -d)"
CA_CRT="$(kubectl -n external-secrets get secret "$SA_SECRET" -o jsonpath='{.data.ca\.crt}' | base64 -d)"

kubectl -n vault exec -i vault-0 -- sh -c "export VAULT_ADDR=http://127.0.0.1:8200 && vault login <ROOT_TOKEN> >/dev/null && vault write auth/kubernetes/config kubernetes_host='https://kubernetes.default.svc:443' kubernetes_ca_cert='$CA_CRT' token_reviewer_jwt='$TOKEN'"
```

## 6) Create policies and roles per environment

```bash
kubectl -n vault exec -i vault-0 -- sh -c "cat <<'EOF' > /tmp/bsis-app-dev.hcl
path \"kv/data/bsis/dev/app\" {
  capabilities = [\"read\"]
}
EOF
export VAULT_ADDR=http://127.0.0.1:8200
vault login <ROOT_TOKEN> >/dev/null
vault policy write bsis-app-dev /tmp/bsis-app-dev.hcl
vault write auth/kubernetes/role/bsis-app-dev bound_service_account_names=bsis-app bound_service_account_namespaces=app-dev policies=bsis-app-dev ttl=24h"
```

Repeat for:

- `bsis-app-stage` with path `kv/data/bsis/stage/app` and namespace `app-stage`
- `bsis-app-prod` with path `kv/data/bsis/prod/app` and namespace `app-prod`

## 7) Seed secrets

```bash
kubectl -n vault exec -it vault-0 -- sh -c "export VAULT_ADDR=http://127.0.0.1:8200 && vault login <ROOT_TOKEN> >/dev/null && vault kv put kv/bsis/dev/app DB_URL='postgres://dev...' JWT_SECRET='dev-secret'"
kubectl -n vault exec -it vault-0 -- sh -c "export VAULT_ADDR=http://127.0.0.1:8200 && vault login <ROOT_TOKEN> >/dev/null && vault kv put kv/bsis/stage/app DB_URL='postgres://stage...' JWT_SECRET='stage-secret'"
kubectl -n vault exec -it vault-0 -- sh -c "export VAULT_ADDR=http://127.0.0.1:8200 && vault login <ROOT_TOKEN> >/dev/null && vault kv put kv/bsis/prod/app DB_URL='postgres://prod...' JWT_SECRET='prod-secret'"
```

## 8) Validate ESO sync

```bash
kubectl -n app-dev get externalsecret,secret | grep bsis-app-secrets
kubectl -n app-stage get externalsecret,secret | grep bsis-app-secrets
kubectl -n app-prod get externalsecret,secret | grep bsis-app-secrets
```

## Notes

- Current chart values use `tlsDisable: true` for initial bootstrap speed. Enable TLS before production hardening.
- Configure auto-unseal (KMS) for production to avoid manual unseal on restarts.
