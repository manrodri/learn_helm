install chart passing configuration using a yaml file

`$ helm install mysite bitnami/drupal --values values.yaml`

```
NAME: mysite
LAST DEPLOYED: Tue Aug 31 15:00:01 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*******************************************************************
*** PLEASE BE PATIENT: Drupal may take a few minutes to install ***
*******************************************************************

1. Get the Drupal URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w mysite-drupal'

  export SERVICE_IP=$(kubectl get svc --namespace default mysite-drupal --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo "Drupal URL: http://$SERVICE_IP/"

2. Get your Drupal login credentials by running:

  echo Username: admin
  echo Password: $(kubectl get secret --namespace default mysite-drupal -o jsonpath="{.data.drupal-password}" | base64 --decode)

```

The `--set` flag takes one or more values directly. They do not need to be stored in a YAML file:

`$ helm install mysite bitnami/drupal --set drupalUsername=admin`

Configuration parameters can be structured.
```
drupalUsername: admin
drupalEmail: admin@example.com
mariadb:
  db:
    name: "my-database"
```

Use of config files is best practice as they can be store and manage in GitHub. Watch out not to store sensitve data in GitHub.



## List your installations

`$ helm list`

```
manuel@dev-server:~/helm$ helm list
NAME  	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART         	APP VERSION
mysite	default  	1       	2021-08-31 15:00:01.076980913 +0000 UTC	deployed	drupal-10.2.33	9.2.4      
manuel@dev-server:~/helm$ 

```

--all-namespaces gives you a list of all your installations in all namespaces.



## Upgrading an Installation

To modify that installation, use helm upgrade.

This is an important distinction to make in the present context because upgrading an installation can consist of two different kinds of changes:

You can upgrade the version of the chart

You can upgrade the configuration of the installation

a *release* is a particular combination of configuration and chart version for an installation.

### Example

For example, say we install the Drupal chart with the ingress turned off.


`$ helm install mysite bitnami/drupal --set ingress.enabled=false`

In this case, we are running an upgrade that will only change the configuration.  Helm will attempt to alter only the bare minimum.

The preceding example will only change the ingress configuration. Nothing changes with the database, or even with the web server running Drupal. For that reason, nothing will be restarted or deleted and re-created. This can occasionally confuse new Helm users, but it is by design. 

When a new version of a chart comes out, you may want to upgrade your existing installation to use the new chart version

```
$ helm repo update (1)
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈ Happy Helming!⎈

$ helm upgrade mysite bitnami/drupal (2)
```


1. Fetch the latest packages from chart repositories.


1. Upgrade the mysite release to use the latest version of bitnami/drupal


### Configuration Values and Upgrades

```$ helm install mysite bitnami/drupal --values values.yaml 
$ helm upgrade mysite bitnami/drupal 
```


1. Install using a configuration file.


1. Upgrade without a configuration file.


The installation will use all of the configuration data supplied in values.yaml, but the upgrade will not. As a result, some settings could be changed back to their defaults. This is usually not what you want.

Helm core maintainers suggest that you provide consistent configuration with each installation and upgrade.

```
$ helm install mysite bitnami/drupal --values values.yaml 
$ helm upgrade mysite bitnami/drupal --values values.yaml 
```

*Note:* You can use helm get values mysite to see what values were used on the last helm install or helm upgrade operation.

The --reuse-values flag will tell Helm to reload the server-side copy of the last set of values, and then use those to generate the upgrade.

```
$ helm upgrade mysite bitnami/drupal --reuse-values
```

### Uninstalling an Installation

To remove a Helm installation, use the helm uninstall command:

```
$ helm uninstall mysite
```
Note that this command does not need a chart name (bitnami/drupal) or any configuration files. It simply needs the name of the installation. 

Like install, list, and upgrade, you can supply a --namespace flag to specify that you want to delete an installation from a specific namespace. Remember you can have installations with the same name in different namespaces

### How Helm Stores Release Information

When we first install a chart with Helm (such as with helm install mysite bitnami/drupal), we create the Drupal application instance, and we also create a special record that contains release information. By default, Helm stores these records as Kubernetes Secrets (though there are other supported storage backends).

```
manuel@dev-server:~/helm$ helm list
NAME  	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART         	APP VERSION
mysite	default  	1       	2021-08-31 15:00:01.076980913 +0000 UTC	deployed	drupal-10.2.33	9.2.4      


manuel@dev-server:~/helm$ kubectl get secrets
NAME                           TYPE                                  DATA   AGE
default-token-6ngqp            kubernetes.io/service-account-token   3      8h
mysite-drupal                  Opaque                                1      46m
mysite-mariadb                 Opaque                                2      46m
mysite-mariadb-token-cjnqm     kubernetes.io/service-account-token   3      46m
sh.helm.release.v1.mysite.v1   helm.sh/release.v1                    1      46m


manuel@dev-server:~/helm$ 

```

We can see multiple release records at the bottom (in my screenshot just one), one for each revision. As you can see, we have created four revisions of mysite by running install and upgrade operations.

```
manuel@dev-server:~/helm$ helm uninstall mysite
release "mysite" uninstalled

manuel@dev-server:~/helm$ helm list
NAME	NAMESPACE	REVISION	UPDATED	STATUS	CHART	APP VERSION

manuel@dev-server:~/helm$ kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-6ngqp   kubernetes.io/service-account-token   3      8h

manuel@dev-server:~/helm$ 
```

 you cannot roll back an uninstall. It is possible, though, to delete the application, but keep the release records:

```
$ helm uninstall --keep-history
```