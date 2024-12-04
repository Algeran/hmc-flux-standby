# Example of Mirantis HMC operator usage with Flux CD

1. Install flux ([official documentation](https://fluxcd.io/flux/installation/bootstrap/github/))

  ```
  export GITHUB_TOKEN=<gh-token>

  flux bootstrap github \
    --token-auth \
    --owner=Algeran \
    --repository=hmc-flux-standby \
    --branch=main \
    --path=management/staging \
    --personal
  ```

2. Create `hmc-system` namespace and label it (required to configure network policies)

  ```
  kubectl create ns hmc-system

  kubectl label ns hmc-system product.mirantis.com=hmc
  ```
3. Apply hmc operator GitRepository and Kustomization object to sync the state:

  ```
  kubectl apply -f management/staging/hmc-system/gitrepository.yaml
  kubectl apply -f management/staging/hmc-system/kustomization.yaml
  ```

  Wait until all operator components will be deployed. Due to the helm release deploys and controls only the operator and cert-manager components and all other parts (CAPI, providers, projectsveltos, etc.) are managed by the operator itself, you need to check readiness of the operator and clustertemplates manually

4. Instal [kubeseal](https://fluxcd.io/flux/guides/sealed-secrets/):

  ```
  kubectl apply -f management/staging/sealed-secrets/gitrepository.yaml
  kubectl apply -f management/staging/sealed-secrets/kustomization.yaml

  kubeseal --fetch-cert \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=flux-system \
    > pub-sealed-secrets.pem
  ```

5. Encrypt the AWS credentials secret and replace it in the repo:

  ```
  kubectl -n hmc-system create secret generic basic-auth \
    --from-literal=AccessKeyID=<AWS_ACCESS_KEY_ID> \
    --from-literal=SecretAccessKey=<AWS_SECRET_KEY> \
    --from-literal=SessionToken=<AWS_SESSION_TOKEN> \
    --dry-run=client \
    -o yaml > aws-cluster-identity-secret.yaml
  
  kubeseal --format=yaml --cert=pub-sealed-secrets.pem \
    < aws-cluster-identity-secret.yaml > aws-cluster-identity-secret-sealed.yaml
  
  mv aws-cluster-identity-secret-sealed.yaml management/staging/hmc-system/components/aws/aws-cluster-identity-secret-sealed.yaml
  ```

6. Apply aws provider identity and credential

  ```
  kubectl apply -f management/staging/providers/gitrepository.yaml
  kubectl apply -f management/staging/providers/kustomization.yaml
  ```
7. Apply clusters configuration

  ```
  kubectl apply -f management/staging/clusters/gitrepository.yaml
  kubectl apply -f management/staging/clusters/kustomization.yaml
  ```