# Authentication in PHEE - GSoC' 22

Steps followed to implement authentication in PHEE

 - Keycloak Kubernetes deployment
 - Kong API gateway with Konga dashboard
 - Authentication with Kong & Keycloak

## Keycloak Kubernetes Deployment 

### Installation

#### Keycloak Installation: 
Helm charts for Keycloak can be found [here](https://artifacthub.io/packages/helm/bitnami/keycloak). 

 - To install the chart with release name keycloak, run the below
```
helm repo add bitnami https://charts.bitnami.com/bitnami //no need to add if added in last step
helm repo update
helm install keycloak bitnami/keycloak
```
### Running Keycloak
Once Keycloak is installed, wait for the external IP to assign. Once the IP is assigned, Keycloak admin console can be accessed there. On accessing the UI for the first time, it will prompt the user to set up the admin username and password. 
 - Login to Admin Console 
 - Create a Realm - by default there will be a
   master realm to manage keycloak. 
  - Create a User
 
Further Keycloak configuration steps will be performed later. 

## Kong API gateway with Konga dashboard

### Installation

Kong can be integrated with MySQL, postgresSQL, MongoDB or in a DB-Less mode. We will be using Postgres.

#### PostgreSQL Installation

We will use PostgreSQL as our database for Kong. We will install PostgreSQL using helm. Helm charts for PostgreSQL can be found [here](https://artifacthub.io/packages/helm/bitnami/postgresql). 
 - To install the chart with release name postgres, run the below
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install postgres bitnami/postgresql -f postgres-values.yaml
```
The postgres-values.yaml file will have the postgres username, password and DB name. This file can be found under postgres.

#### Kong API gateway Installation

Helm charts for Kong can be found [here](https://artifacthub.io/packages/helm/bitnami/keycloak). 

 - To install the chart with release name keycloak, run the below
```
helm repo add kong https://charts.konghq.com
helm repo update
helm install kong/kong -f kong-values.yaml --version 2.6.4 --skip-crds
```
*Note: The newest version of kong is not compatible with PostgreSQL 14. Make sure to add the version flag while installing kong*

The kong-values.yaml file can be found under kong. Make sure to add database connection strings, enable admin service, which will be used as an endpoint by konga to consume kong admin API.

#### Konga Installation

 - Download the Konga helm chart from [here](https://github.com/pantsel/konga/tree/master/charts/konga). 
 - Go to the directory where konga is installed and run the below
   command.
```
helm install konga .
```
- Inorder to expose konga, use port forwarding or expose using a load balancer service. 
- On logging for the first time, you will be promted to create a user
- Setup a new connection to kong by adding the kong admin service name. 
- If the connection is successful the Konga dasboard should automatically appear. 

#### Kong OIDC plugin Installation
Kong OIDC plugin is only available with Kong-Enterprise. So we will add this plugin manually. This plugin is required in order to communicate with keycloak. 
- Kong OIDC can be installed using luarocks
 - We need to download the kong-OIDC repo and inside the directory, run
 ```
luarocks install kong-oidc --tree .
 ```
 - Create the Config map for Kong OIDC and apply it. 
 ```
kubectl create configmap kong-plugin-oidc --from-file=/kong/plugins/oidc -n default
 ```
 - Add below under ```plugins``` in the kong comnfiguration file
 ```
plugins:
   configMaps:
   - pluginName: oidc
     name: kong-plugin-oidc
 ```
- Then run the below command

```helm upgrade my-kong kong/kong -f values.yaml --version 2.6.4 --skip-crds```
- Once this is done, we should be able to see the OIDC plugin at Konga Dashboard  -> Plugins -> OIDC

## Authentication with Kong & Keycloak

Once we have Keycloak, Kong, Konga GUI, and Kong OIDC plugin properly set up we can start with the authentication flow. Follow the below steps:

### Steps to be followed in Keycloak
This is the continuation of steps from above. 
- Create a client in keycloak. Client refers to a service. 
- Set “Access Type” to “Confidential”, the “Root URL” to the service you are running and the “Valid Redirect URIs” to “/*”  and save. 
- Once saved, go to credentials tab and copy the client secret.

### Steps in Kong
- Add a service in Kong dashboard. 
- Make sure to populate the host, protocoal and port for the service. 

### Configuring Kong OIDC plugin with Keycloak
In Kong we can add plugins to a whole services or specifc routes of a service depending on the use case. 
- Choose the service or route to which needs to be secured.
- Go to plugins and add the OIDC plugin. In the plugin configuration there are three mandatory fields.
	- realm #name of the realm 
	- client ID #name of the client we created (from keycloak) 
	- client secret #saved in above steps
	- discovery - [https://keycloak-external-ip/auth/realms/master/.well-known/openid-configuration](https://keycloak.mockharsh.tk:8180/auth/realms/master/.well-known/openid-configuration)  (my keycloak is running [here](https://831f7d04-us-east.lb.appdomain.cloud))
