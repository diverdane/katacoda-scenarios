# Continuing as Security Admin

<p align="center">
  <img src="assets/security_admin.jpg">
</p>

## Create Application Database
In this section, we'll create the application database and user, and securely store the user's credentials:

1. Create the application database
2. Create the pets table in that database
3. Create an application user with limited privileges: SELECT and INSERT on the pets table
4. Store these database application-credentials in Kubernetes secrets.

So we can refer to them later, export the database name and application-credentials as environment variables:

`export APPLICATION_DB_NAME=application_db
export APPLICATION_DB_USER=app_user
export APPLICATION_DB_INITIAL_PASSWORD=app_user_password`{{execute}}

Finally, to perform the 4 steps listed above, run:

`docker run --rm -i -e PGPASSWORD=${SECURITY_ADMIN_PASSWORD} postgres:9.6 \
 psql -U ${SECURITY_ADMIN_USER} \
 "postgres://${REMOTE_DB_HOST}:${REMOTE_DB_PORT}/postgres" << EOSQL
CREATE DATABASE ${APPLICATION_DB_NAME};
/* connect to it */
\c ${APPLICATION_DB_NAME};
CREATE TABLE pets (
 id serial primary key,
 name varchar(256)
);
/* Create Application User */
CREATE USER ${APPLICATION_DB_USER} PASSWORD '${APPLICATION_DB_INITIAL_PASSWORD}';
/* Grant Permissions */
GRANT SELECT, INSERT ON public.pets TO ${APPLICATION_DB_USER};
GRANT USAGE, SELECT ON SEQUENCE public.pets_id_seq TO ${APPLICATION_DB_USER};
EOSQL`{{execute}}

You should see:
```
CREATE DATABASE
You are now connected to database "application_db" as user "security_admin_user".
CREATE TABLE
CREATE ROLE
GRANT
GRANT
```

## Create Application Namespace and Store Credentials
The application will be scoped to the application-ns namespace.
To create the namespace run:

`kubectl create namespace application-ns`{{execute}}

You should see:
```
namespace "application-ns" created
```

Next we’ll store the application-credentials in Kubernetes Secrets:

`kubectl --namespace application-ns \
  create secret generic app-backend-credentials \
  --from-literal=host="${REMOTE_DB_HOST}" \
  --from-literal=port="${REMOTE_DB_PORT}" \
  --from-literal=username="${APPLICATION_DB_USER}" \
  --from-literal=password="${APPLICATION_DB_INITIAL_PASSWORD}"`{{execute}}

You should see:
```
secret "app-backend-credentials" created
```

While Kubernetes Secrets are more secure than hard-coded ones, in a real deployment you should secure secrets in a fully-featured vault, like Conjur.

## Create Secretless Broker Configuration ConfigMap
With our database ready and our credentials safely stored, we can now configure the Secretless Broker. We’ll tell it where to listen for connections and how to proxy them.

After that, the developer’s application can access the database without ever knowing the application-credentials.

A Secretless Broker configuration file defines the services that Secretless with authenticate to on behalf of your application.
The Secretless Broker configuration file can be viewed here: [secretless.yml](secretless.yml)

Here’s what this configuration does:

* Defines a service called `pets-pg` that listens for PostgreSQL connections on `localhost:5432`
* Says that the database `host`, `port`, `username` and `password` are stored in Kubernetes Secrets
* Lists the ids of those credentials within Kubernetes Secrets

**Note:** This configuration is shared by all Secretless Broker sidecar containers. There is one Secretless sidecar in every application Pod replica.
**Note:** Since we don't specify an `sslmode` in the Secretless Broker config, it will use the default `require` value.

Next we create a Kubernetes ConfigMap from this secretless.yml:

`kubectl --namespace application-ns \
   create configmap \
   application-secretless-config \
   --from-file=secretless.yml`{{execute}}

You should see:
```
configmap/application-secretless-config created
```

## Create Application Service Account and Grant Entitlements
To grant our application access to the credentials in Kubernetes Secrets, we’ll need to create each of the following:
* ServiceAccount
* Role
* RoleBinding

The YAML manifest to create these resources can be viewed here: [application-entitlements.yml](application-entitlements.yml)
This manifest does the following:
* Creates a ServiceAccount for our application
* Creates a Role with permissions to "get" the backend-credentials secret
* Creates a RoleBinding so our ServiceAccount has this Role

To create these three Kubernetes resources, run this command:
`kubectl --namespace application-ns apply -f application-entitlements.yml`{{execute}}

You should see:
```
role "quick-start-backend-credentials-reader" created
rolebinding "read-quick-start-backend-credentials" created
```

## Up next...
As an Application Developer, you no longer need to worry about all the passwords and database connections! You will deploy an application and leave it up to the Secretless Broker to make the desired connection to the database.
