Command to get the token, username, URL (password field), base64-decoded:

# For the app-workload repo (gitops-orderlay-deployments) — this is almost certainly the one you want
kubectl get secret gitops-ansible-github-secret -n argocd -o jsonpath='{.data.password}' | base64 -d

# For the infra/addons repo (GH-infra-and-k8s-charts-central), if you need that one instead
kubectl get secret argocd-ansible-github-secret -n argocd -o jsonpath='{.data.password}' | base64 -d



kubectl get secret gitops-ansible-github-secret -n argocd -o jsonpath='{.data.url}' | base64 -d

kubectl get secret gitops-ansible-github-secret -n argocd -o jsonpath='{.data.username}' | base64 -d



kubectl get secret gitops-ansible-github-secret -n argocd -o jsonpath='{.data.url}' | base64 -d; echo
kubectl get secret gitops-ansible-github-secret -n argocd -o jsonpath='{.data.username}' | base64 -d; echo
