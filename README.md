# Pipelines as Code με GitLab - Setup Guide

## Περιγραφή
Οδηγός για Pipelines as Code με GitLab, δείχνοντας Task Reusability και Selective/Full Pipeline Execution.

**Repository:** `http://cicd-scm.solutioncenter.uni/unisystemsstm-fiware/fiwaremqtt-client`  
**Namespace:** `pac-poc`

---

## Προαπαιτούμενα

- OpenShift Cluster με Red Hat OpenShift Pipelines
- GitLab Personal Access Token με: `api`, `read_repository`, `write_repository`
- Πρόσβαση στο namespace `pac-poc`

---

## Βήμα 1: GitLab Personal Access Token

1. GitLab → **Settings → Access Tokens**
2. Scopes: `api`, `read_repository`, `write_repository`
3. Αποθηκεύστε το token

---

## Βήμα 2: Repository Setup

> **💡 Git Provider Detection:** Ο PaC Controller αναγνωρίζει αυτόματα τον Git provider (GitLab, GitHub, Gitea, etc.) από το **git_provider.url**. Για GitLab χρησιμοποιεί το GitLab API endpoint.

> **🤖 Auto-Created Resources:** Όταν δημιουργείς Repository CR, ο PaC controller δημιουργεί αυτόματα:
> - **ServiceAccount** (`<repo-name>-sa`)
> - **RoleBinding** (permissions για pipeline execution)
> - **Git Auth Secret** (`pac-gitauth-<hash>`) για private repos
> - **PipelineRun** (dynamically σε κάθε Git event)
>
> ❌ **ΔΕΝ χρειάζονται:** TriggerTemplate, TriggerBinding, EventListener

### Με CLI (Αυτόματο)
```bash
tkn pac create repository -n pac-poc
```

Θα ζητήσει:
- Git URL: `http://cicd-scm.solutioncenter.uni/unisystemsstm-fiware/fiwaremqtt-client`
- **GitLab API URL:** `http://cicd-scm.solutioncenter.uni` ← Έτσι καταλαβαίνει GitLab
- Token: Το token σας

### Με YAML (Χειροκίνητο)

**Secret:**
```bash
oc create secret generic gitlab-webhook-config \
  --from-literal provider.token="<YOUR_TOKEN>" \
  --from-literal webhook.secret="$(openssl rand -hex 20)" \
  -n pac-poc
```

**Repository CR:**
```yaml
apiVersion: "pipelinesascode.tekton.dev/v1alpha1"
kind: Repository
metadata:
  name: fiwaremqtt-client
  namespace: pac-poc
spec:
  url: "http://cicd-scm.solutioncenter.uni/unisystemsstm-fiware/fiwaremqtt-client"
  git_provider:
    url: "http://cicd-scm.solutioncenter.uni"
    secret:
      name: "gitlab-webhook-config"
      key: "provider.token"
    webhook_secret:
      name: "gitlab-webhook-config"
      key: "webhook.secret"
```

```bash
oc apply -f repository.yaml -n pac-poc
```

---

## Βήμα 3: Webhook Setup (στο GitLab)

### Με CLI (Αυτόματο)
```bash
tkn pac webhook add -n pac-poc
```
*Το command συνδέεται στο GitLab και δημιουργεί αυτόματα το webhook.*

### Με YAML (Χειροκίνητο)

1. Πάρτε το PAC URL:
```bash
echo https://$(oc get route -n openshift-pipelines pipelines-as-code-controller -o jsonpath='{.spec.host}')
```

2. **Στο GitLab:** Repository → **Settings → Webhooks**
   - **URL:** PAC controller URL (από πάνω)
   - **Secret token:** Από το secret (βλέπε παρακάτω)
   - **Trigger events:**
     - ✅ Push events
     - ✅ Comments
     - ✅ Merge request events
   - **Enable SSL verification:** Αν έχεις valid certificate

3. Πάρε το webhook secret:
```bash
oc get secret gitlab-webhook-config -n pac-poc -o jsonpath='{.data.webhook\.secret}' | base64 -d
```

---

## Βήμα 4: Δημιουργία Tasks & Pipelines

### Εγκατάσταση στο Cluster

**Tasks (Reusable):**
```bash
oc apply -f tasks/test-task.yaml -n pac-poc
oc apply -f tasks/build-task.yaml -n pac-poc
oc apply -f tasks/deploy-task.yaml -n pac-poc
```

**Pipelines:**
```bash
oc apply -f pipelines/test-pipeline.yaml -n pac-poc
oc apply -f pipelines/build-pipeline.yaml -n pac-poc
oc apply -f pipelines/deploy-pipeline.yaml -n pac-poc
oc apply -f pipelines/full-cicd-pipeline.yaml -n pac-poc
```

---

## Βήμα 5: Προσθήκη .tekton/ στο Git

Δημιουργήστε στο repository:

**`.tekton/test-on-pr.yaml`** (Auto-trigger):
```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: test-on-pr
  annotations:
    pipelinesascode.tekton.dev/on-event: "[pull_request]"
    pipelinesascode.tekton.dev/on-target-branch: "[main]"
spec:
  pipelineRef:
    name: test-pipeline
```

**`.tekton/test-on-comment.yaml`** (Manual `/test`):
```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: test-on-comment
  annotations:
    pipelinesascode.tekton.dev/on-event: "[pull_request]"
    pipelinesascode.tekton.dev/on-comment: "^/test$"
spec:
  pipelineRef:
    name: test-pipeline
```

**`.tekton/build-on-comment.yaml`** (Manual `/build`):
```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: build-on-comment
  annotations:
    pipelinesascode.tekton.dev/on-event: "[pull_request]"
    pipelinesascode.tekton.dev/on-comment: "^/build$"
spec:
  pipelineRef:
    name: build-pipeline
```

**`.tekton/deploy-on-comment.yaml`** (Manual `/deploy`):
```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: deploy-on-comment
  annotations:
    pipelinesascode.tekton.dev/on-event: "[pull_request]"
    pipelinesascode.tekton.dev/on-comment: "^/deploy$"
spec:
  pipelineRef:
    name: deploy-pipeline
```

**`.tekton/full-cicd-on-comment.yaml`** (Manual `/run-all`):
```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: full-cicd-on-comment
  annotations:
    pipelinesascode.tekton.dev/on-event: "[pull_request]"
    pipelinesascode.tekton.dev/on-comment: "^/run-all$"
spec:
  pipelineRef:
    name: full-cicd-pipeline
```

---

## Βήμα 6: Commit & Push

```bash
git add .tekton/
git commit -m "Add PaC triggers"
git push
```

---

## Χρήση

### Αυτόματο
- Άνοιξε PR → test-pipeline τρέχει αυτόματα

### Manual (PR Comments)
| Comment | Pipeline | Τρέχει |
|---------|----------|--------|
| `/test` | test-pipeline | test-task |
| `/build` | build-pipeline | build-task |
| `/deploy` | deploy-pipeline | deploy-task |
| `/run-all` | full-cicd-pipeline | test → build → deploy |

---

## Monitoring

### CLI
```bash
# List repositories
tkn pac list -n pac-poc

# Show logs
tkn pac logs -n pac-poc -L

# Real-time logs
tkn pac logs -n pac-poc -f

# PipelineRuns
oc get pipelineruns -n pac-poc
```

### Console
OpenShift Console → **Pipelines → PipelineRuns** → namespace `pac-poc`

---

## Troubleshooting

### CLI
```bash
# PAC controller logs
oc logs -f deployment/pipelines-as-code-controller -n openshift-pipelines

# Events
oc get events -n pac-poc --sort-by='.lastTimestamp'

# Verify repository
oc describe repository fiwaremqtt-client -n pac-poc
```

### YAML
```bash
# Check webhook secret
oc get secret gitlab-webhook-config -n pac-poc -o yaml

# Test pipeline without commit
tkn pac resolve -f .tekton/test-on-comment.yaml | oc apply -f -
```

---

## Βασικές Έννοιες

**Task** = Reusable component (ζει στο cluster)  
**Pipeline** = Orchestration (ζει στο cluster, χρησιμοποιεί tasks)  
**PipelineRun** = Event trigger (ζει στο Git `.tekton/`, καλεί pipeline)

**Selective Execution:** `/test`, `/build`, `/deploy` - ένα stage κάθε φορά  
**Full Pipeline:** `/run-all` - όλα μαζί με σειρά

---

## GitOps Comments

- `/test`, `/build`, `/deploy`, `/run-all` - Εκτέλεση pipelines
- `/retest` - Επανεκτέλεση
- `/cancel` - Ακύρωση

---

## Resources

- [Demo README](README.md:1) - Πλήρης τεκμηρίωση
- [CLI Comparison](CLI-DIFFERENCES.md:1) - `opc` vs `tkn pac` vs `tkn-pac`
- [Red Hat Docs](https://docs.redhat.com/en/documentation/red_hat_openshift_pipelines/1.12/html-single/pipelines_as_code/index)
# pac-poc
# pac-poc
