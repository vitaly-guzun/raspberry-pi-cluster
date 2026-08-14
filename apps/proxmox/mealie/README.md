# Mealie trial runbook

Mealie is the first stage of the grocery workflow. This trial intentionally
starts with recipes and meal planning only. Grocy and Albert Heijn integration
should be added after the workflow has been used successfully for one week.

## Deployment

Flux deploys Mealie from `apps/proxmox/mealie`. The application uses SQLite on
a 5 Gi `local-path` PVC. This is appropriate for the initial single-user trial
because the volume is node-local, not NFS-mounted.

After Flux reconciles the change, verify the deployment:

```shell
kubectl -n mealie rollout status deployment/mealie --timeout=5m
kubectl -n mealie get pod,pvc,service
kubectl -n mealie logs deployment/mealie --tail=100
```

For the initial test, access Mealie without exposing it publicly:

```shell
kubectl -n mealie port-forward service/mealie 9925:9000
```

Open `http://localhost:9925` and sign in with the installation credentials:

```text
Username: changeme@example.com
Password: MyPassword
```

Immediately replace the default email and password in the user profile. Public
sign-up is disabled.

## First-week user test

Do not try to model the entire kitchen yet. The goal is to validate that Mealie
is convenient enough for the weekly routine.

1. Import or create three meals that are cooked regularly.
2. Check ingredient names, quantities, and units in each recipe.
3. Add the three meals to the next seven days in the meal planner.
4. Generate a shopping list from the planned recipes.
5. Remove ingredients that are already at home.
6. Use the shopping list for one real grocery order or store visit.
7. Note any ingredient that was ambiguous, duplicated, or incorrectly scaled.

The trial is successful when:

- adding a recipe takes less than a few minutes;
- changing the number of servings produces sensible quantities;
- the generated shopping list is understandable on a phone;
- the same ingredient from different recipes can be recognized and combined;
- using the workflow feels easier than writing the list manually.

If the trial succeeds, the next stage is Grocy plus a small integration service
that subtracts home stock before the shopping list is created.

## Backup during the trial

Mealie can create a full site backup from its administration UI. Download a
backup after the initial recipes are entered and keep it outside the Kubernetes
node. A scheduled backup to Synology should be added before Mealie becomes a
permanent service.

Do not move the SQLite database onto NFS. Mealie explicitly recommends
PostgreSQL instead of SQLite when the database itself is stored on NAS.

## External access later

External access is deliberately not part of the first deployment. After the
trial, create a dedicated Cloudflare tunnel and set Mealie's `BASE_URL` to its
final HTTPS hostname. Do not publish Mealie with its default credentials.
