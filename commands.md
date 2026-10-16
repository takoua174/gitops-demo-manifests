# TP5 GitOps / ArgoCD : commandes pour la démonstration en direct

> Application ArgoCD : `gitops-demo-dev` · Namespace : `gitops-dev` · Deployment : `gitops-demo` · Dossier : `overlays/dev`
> Remplacer `<user>` par ton nom d'utilisateur GitHub.

---

## 0. Avant de commencer (à faire 10 minutes avant)

**Terminal A : tunnel vers l'interface ArgoCD (laisser ouvert)**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

**Terminal B : suivre les pods en direct (laisser ouvert)**
```bash
kubectl get pods -n gitops-dev -w
```

**Terminal C : dossier du dépôt de manifests (pour les commandes git)**
```bash
cd gitops-demo-manifests
git pull origin main
```

**Navigateur :** `https://localhost:8080` → application `gitops-demo-dev` ouverte.

**Se connecter au CLI (si besoin) :**
```bash
argocd login localhost:8080 --username admin --insecure
```

**Vérifier l'état de départ :**
```bash
argocd app get gitops-demo-dev
kubectl get deploy,pods,svc -n gitops-dev
```
Attendu : `Sync Status: Synced`, `Health Status: Healthy`.

**Accélérer la détection pendant la démo (au besoin) :**
```bash
argocd app get gitops-demo-dev --refresh
```
ou bouton **Refresh** dans l'interface.

---

## DÉMO 1 : Déploiement continu automatisé

**Objectif :** un changement dans Git arrive dans le cluster sans `kubectl apply`.

**1. État initial**
```bash
kubectl get deploy gitops-demo -n gitops-dev
```

**2. Modifier le nombre de réplicas dans Git**

Ouvrir `overlays/dev/patch-replicas.yaml` et passer `replicas: 2` à `replicas: 4`.
(Avec Git Bash / Linux : `sed -i 's/replicas: 2/replicas: 4/' overlays/dev/patch-replicas.yaml`)

**3. Commit et push**
```bash
git add overlays/dev/patch-replicas.yaml
git commit -m "feat(dev): passer a 4 replicas"
git push origin main
```

**4. Observer**
- Interface ArgoCD : `OutOfSync` → `Syncing` → `Synced` / `Healthy`
- Terminal B : deux nouveaux pods apparaissent
```bash
argocd app get gitops-demo-dev --refresh
kubectl get deploy gitops-demo -n gitops-dev
```
Attendu : `4/4`.

**À dire :** « Je n'ai lancé aucun kubectl apply. ArgoCD a détecté le commit et synchronisé. »

---

## DÉMO 2 : Drift et Self-Healing

**Objectif :** une modification manuelle du cluster est annulée automatiquement.

**Scénario A : supprimer le Deployment**
```bash
kubectl delete deployment gitops-demo -n gitops-dev
kubectl get deploy -n gitops-dev -w
```
Attendu : ArgoCD détecte l'écart (`OutOfSync` / `Missing`) et **recrée** le Deployment.

**Scénario B : changer les réplicas à la main**
```bash
kubectl scale deployment gitops-demo -n gitops-dev --replicas=10
kubectl get deploy gitops-demo -n gitops-dev
```
Attendu : retour automatique à la valeur de Git (4).

**Scénario C (optionnel) : modifier le Service**
```bash
kubectl edit svc gitops-demo -n gitops-dev
```
Changer le port, enregistrer : ArgoCD annule la modification.

**Bonus : montrer la différence détection / correction**
```bash
argocd app set gitops-demo-dev --self-heal=false
kubectl scale deployment gitops-demo -n gitops-dev --replicas=10
argocd app get gitops-demo-dev
```
Attendu : `OutOfSync`, 10 réplicas conservés (écart détecté mais non corrigé).
```bash
argocd app set gitops-demo-dev --self-heal
```
Attendu : retour automatique à 4 réplicas.

**À dire :** « La détection signale l'écart. Le self-heal le corrige sans intervention humaine. Git reste la seule source de vérité. »

---

## DÉMO 3 : Sync Status versus Health Status

**Objectif :** montrer `Synced` mais `Degraded`.

**1. Mettre une image inexistante dans Git**

Dans `overlays/dev/kustomization.yaml`, remplacer le tag de l'image par `9.9.9-inexistant`.
(Avec kustomize : `cd overlays/dev && kustomize edit set image ghcr.io/<user>/gitops-demo-app=ghcr.io/<user>/gitops-demo-app:9.9.9-inexistant && cd ../..`)

**2. Commit et push**
```bash
git add overlays/dev/kustomization.yaml
git commit -m "test(dev): image inexistante (demo sync vs health)"
git push origin main
```

**3. Observer**
```bash
argocd app get gitops-demo-dev --refresh
kubectl get pods -n gitops-dev
```
Attendu : un pod en `ErrImagePull` puis `ImagePullBackOff`, les anciens pods toujours `Running`.

```bash
kubectl describe pod <nom-du-pod-en-erreur> -n gitops-dev
```
(Regarder la section `Events` en bas.)

**Dans l'interface :** `Sync Status: Synced` et `Health Status: Degraded` (ou `Progressing` au début).

**À dire :** « Synced veut dire que le cluster correspond à Git. Healthy veut dire que ça fonctionne. Git demandait une image inexistante : le cluster l'applique fidèlement, mais le pod ne démarre pas. »

---

## DÉMO 4 : Historique et Rollback

**Objectif :** revenir rapidement à une version stable.

**1. Voir l'historique**
```bash
argocd app history gitops-demo-dev
```
Noter l'**ID** de la dernière révision stable (ex : `13`) et son hash de commit.

Dans l'interface : onglet **History and Rollback**.

**2. Rollback via ArgoCD (désactiver l'auto-sync d'abord)**
```bash
argocd app set gitops-demo-dev --sync-policy none
argocd app rollback gitops-demo-dev <ID>
```
(Ou dans l'interface : choisir la révision stable → **Rollback**.)

Vérifier :
```bash
argocd app get gitops-demo-dev
kubectl get pods -n gitops-dev
```
Attendu : `Synced` / `Healthy`, pods `Running`.

**3. Réactiver l'auto-sync**
```bash
argocd app set gitops-demo-dev --sync-policy automated --self-heal --auto-prune
```

**4. Corriger aussi Git (méthode GitOps recommandée)**

Sans cette étape, Git contient toujours l'image défectueuse.
```bash
git log --oneline -5
git revert <hash-du-commit-fautif> --no-edit
git push origin main
```
Attendu : ArgoCD resynchronise, `Synced` / `Healthy`.

**À dire :** « Le rollback ArgoCD est immédiat. Avec l'auto-sync, la correction durable passe par un git revert, qui garde l'historique. »

---

## Vérification finale

```bash
argocd app get gitops-demo-dev
kubectl get deploy,pods -n gitops-dev
git log --oneline -10
```
Attendu : `Synced` / `Healthy`.

---

## Remise à zéro entre deux essais

1. Remettre dans Git : image `1.2.0` et `replicas: 4`, puis commit et push.
2. Réactiver l'auto-sync et le self-heal :
```bash
argocd app set gitops-demo-dev --sync-policy automated --self-heal --auto-prune
```
3. Vérifier :
```bash
argocd app get gitops-demo-dev --refresh
```

---

## Si quelque chose bloque

| Problème | Solution |
|---|---|
| ArgoCD ne détecte pas le commit | Bouton **Refresh** ou `argocd app get gitops-demo-dev --refresh` |
| Rollback annulé aussitôt | L'auto-sync est encore actif : `argocd app set gitops-demo-dev --sync-policy none` |
| Interface inaccessible | Relancer le `kubectl port-forward` du terminal A |
| `argocd` ne répond plus | `argocd login localhost:8080 --username admin --insecure` |
| `ImagePullBackOff` sur une vraie image | Le package GHCR n'est pas public |
| Mot de passe admin oublié | `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" \| base64 -d` |