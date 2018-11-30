# LaravelDay Helm chart

## Installation

```bash
helm install stable/redis --name cache --namespace laravelday-2018
helm install stable/mysql --name database --namespace laravelday-2018
helm upgrade laravelday . --namespace laravelday-2018 --install
```

## Uninstall

```bash
helm delete cache --purge
helm delete database --purge
helm delete laravelday --purge
```

### Dependencies

```bash
helm install stable/redis --name cache --namespace laravelday-2018
helm install stable/mysql --name database --namespace laravelday-2018
helm install --name jenkins --namespace cicd --set Master.AdminPassword=admin --set rbac.install=true stable/jenkins
```