# Learning About a Release

To start, let’s revisit the five phases of a Helm installation from the previous section. They were:


* Load the chart
* Parse the values.
* Execute the templates.
* Render the YAML.
* Send it to Kubernetes.

During the last phase, though, Helm sends that data to Kubernetes. And then the two communicate back and forth until the release is either accepted or rejected. Moreover, since many individuals may be working on the same copy of that particular application installation, Helm needs to monitor the state in such a way that multiple users can see that information.

Helm provides this feature with release records.

## Release Records

 we also saw how helm install creates a special type of Kubernetes Secret that holds release information

 ```
$ kubectl get secret
NAME                           TYPE                                  DATA   AGE
default-token-g777k            kubernetes.io/service-account-token   3      6m
mysite-drupal                  Opaque                                1      2m20s
mysite-mariadb                 Opaque                                2      2m20s
sh.helm.release.v1.mysite.v1   helm.sh/release.v1

 ```

Helm automatically generated this secret (sh.helm.release.v1.mysite.v1)  to track version 1 of our mysite installation (which is a Drupal site).

Each release record contains enough information to re-create the Kubernetes objects for that revision (an important thing for helm rollback). 

If we look at the release:

```
apiVersion: v1
data:
  release: SDRzSUFBQU... # Lots of Base64-encoded data removed
kind: Secret
metadata:
  creationTimestamp: "2020-08-11T18:37:26Z"
  labels: 
    modifiedAt: "1597171046"
    name: mysite
    owner: helm
    status: deployed
    version: "3"
  name: sh.helm.release.v1.mysite.v3
  namespace: default
  resourceVersion: "1991"
  selfLink: /api/v1/namespaces/default/secrets/sh.helm.release.v1.mysite.v3
  uid: cbb8b457-e331-467b-aa78-1e20360b5be6
type: helm.sh/release.v1


```

The labels section of the Kubernetes metadata contains information about this release. In short, this secret is stored inside of Kubernetes so that different users of the same cluster have access to the same release information.


During the life cycle of a release, it can pass through several different statuses:

* pending-install
Before sending the manifests to Kubernetes, Helm claims the installation by creating a release (marked version 1) whose status is set to pending-install.
* deployed
As soon as Kubernetes accepts the manifest from Helm, Helm updates the release record, marking it as deployed.
* pending-upgrade
When a Helm upgrade is begun, a new release is created for an installation (e.g., v2), and its status is set to pending-upgrade.
* superseded
When an upgrade is run, the last deployed release is updated, marked as superseded, and the newly upgraded release is changed from pending-upgrade to deployed.
* pending-rollback
* uninstalling
* uninstalled
* failed

## Find Details of a Release with helm get

While helm list provides a summary view of installations, the helm get set of commands provide deeper information about a particular release. There are five helm get subcommands (hooks, manifests, notes, values, and all).

```
$ helm get values wordpress
USER-SUPPLIED VALUES:
image:
  pullPolicy: NoSuchPolicy

```

```
$ helm get values wordpress --revision 2
USER-SUPPLIED VALUES:
image:
  tag: latest
```

When the --all flag is specified, Helm will get the complete computed set of values, sorted alphabetically. 

## History and Rollbacks

We can investigate the release history of WordPress to see what happened. To do this, we will use helm history:

```
$ helm history wordpress
REVISION UPDATED       STATUS     CHART             APP VER  DESCRIPTION
1        Wed Aug 12... superseded wordpress-9.3.11  5.4.2    Install complete
2        Wed Aug 12... deployed  	wordpress-9.3.11  5.4.2    Upgrade complete
3        Wed Aug 12... failed    	wordpress-9.3.11  5.4.2    Upgrade \
  "wordpress" failed: cannot patch "wordpress" with kind Deployment: \
  Deployment.apps "wordpress" is invalid: \
  spec.template.spec.containers[0].imagePullPolicy: Unsupported value: \
  "NoSuchPolicy": supported values: "Always", "IfNotPresent", "Never"
```

when it was upgraded again, that upgrade failed. The helm history command even gives us the error message that Kubernetes returned when marking the release failed.

```
$ helm rollback wordpress 2
Rollback was a success! Happy Helming!
```

Now we can once again use helm history to see what has happened:

```
REVISION  UPDATED       STATUS      CHART             APP VER  DESCRIPTION
1         Wed Aug 12... superseded  wordpress-9.3.11  5.4.2    Install complete
2         Wed Aug 12... superseded  wordpress-9.3.11  5.4.2    Upgrade complete
3         Wed Aug 12... failed      wordpress-9.3.11  5.4.2    Upgrade \
  "wordpress" failed: cannot patch "wordpress" with kind Deployment: \
  Deployment.apps "wordpress" is invalid: \
  spec.template.spec.containers[0].imagePullPolicy: Unsupported value: \
  "NoSuchPolicy": supported values: "Always", "IfNotPresent", "Never"
4         Wed Aug 12... deployed    wordpress-9.3.11  5.4.2    Rollback to 2
```

Rollbacks can on occasion cause some unexpected behavior, especially if the Kubernetes resources have been hand-edited by users. Helm and Kubernetes will attempt to preserve those hand-edits if they do not conflict with the rollback. Helm core maintainers recommend against hand-editing resources. If all edits are made through Helm, then you can use Helm tools effectively and with no guesswork.

