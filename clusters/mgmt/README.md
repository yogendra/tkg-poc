# Managment Cluster Setup

1. Switch context
   ```
   kubectl config use-context poc-mgmt-admin@poc-mgmt
   ```
2. Deploy TMC extension controller

kubectl apply -f tmc-extension-manager.yaml
