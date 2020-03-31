# nachogym
## Setup Development Environment
### Setup k8s on Mac for development purposes
- Install docker for mac here: https://docs.docker.com/docker-for-mac/install/
- Enable kubernetes: https://docs.docker.com/docker-for-mac/#kubernetes
- Install kubernetes dashboard: https://github.com/kubernetes/dashboard
- Setup kubernetes dashboard user
  - From deploy/mac, run:
    - kubectl apply -f dashboard-adminuser.yaml
    - kubectl apply -f dashboard-rbac-rolebinding.yaml
  - On the CLI, run:
    - kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}') 
  - From Chrome (note, as of this writing does not work from Firefox), go to:
    - http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
    - Select Token, and paste the token from the above kubectl command into the Enter token * field

### Install and run postgres on k8s (https://github.com/bitnami/charts/)
- helm repo add bitnami https://charts.bitnami.com/bitnami
- helm install postgresql bitnami/postgresql
- PostgreSQL can be accessed via port 5432 on the following DNS name from within your cluster:

    postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)

To connect to your database run the following command:

    kubectl run postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:11.7.0-debian-10-r43 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgresql -U postgres -d postgres -p 5432

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432

### Setup Postgres with Keycloak User and DB

From the CLI:
createuser keycloak
psql
ALTER USER keycloak WITH ENCRYPTED password 'keycloak';
CREATE DATABASE keycloak WITH ENCODING='UTF8' OWNER=keycloak;
\q

### Install and run keycloak on k8s
- helm repo add codecentric https://codecentric.github.io/helm-charts
- helm install keycloak --set keycloak.persistence.dbVendor=postgres --set keycloak.persistence.dbName=keycloak --set keycloak.persistence.dbHost=postgresql --set keycloak.persistence.dbPort=5432 --set keycloak.persistence.dbUser=keycloak --set keycloak.persistence.dbPassword=keycloak --set keycloak.persistence.deployPostgres=false codecentric/keycloak
- export POD_NAME=$(kubectl get pods --namespace default -l app.kubernetes.io/instance=keycloak -o jsonpath="{.items[0].metadata.name}")
- kubectl port-forward --namespace default $POD_NAME 8080

### Add admin user to Keycloak

kubectl exec -it keycloak-0 -- /bin/bash
cd /opt/jboss/keycloak/bin
. ./add-user-keycloak.sh -u admin -p admin
. ./jboss-cli.sh --connect command=:reload
exit

