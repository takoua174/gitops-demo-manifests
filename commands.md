demo 1:
kubectl port-forward -n gitops-dev svc/gitops-demo 3000:80
curl.exe http://localhost:3000/

demo2:
kubectl delete deployment gitops-demo -n gitops-dev
kubectl get deploy -n gitops-dev -w

kubectl scale deployment gitops-demo -n gitops-dev --replicas=10
kubectl get deploy -n gitops-dev

argocd app set gitops-demo-dev --self-heal=false
kubectl scale deployment gitops-demo -n gitops-dev --replicas=10
kubectl get deploy -n gitops-dev
argocd app set gitops-demo-dev --self-heal=true

demo3:
