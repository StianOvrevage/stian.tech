

# Connecting

KUBECONFIG=~/.kube/config:k8s-do1-kubeconfig.yaml
kubectl config view --flatten > tmp
mv tmp ~/.kube/config

# Initial cluster setup

kubectl apply -f rbac-config.yaml
helm init --service-account tiller

# Install countly


helm upgrade --install mongodb stable/mongodb --set mongodbRootPassword=kjfedbkeFKJN --set mongodbUsername=countly --set mongodbPassword=akjsfbSAD23wefk --set mongodbDatabase=countly --set persistence.storageClass=do-block-storage --set service.type=LoadBalancer

