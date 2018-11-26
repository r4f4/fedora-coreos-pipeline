The `manifests/` directory contains OpenShift manifests that will set up a Jenkins pipeline
using the [Kubernetes Jenkins plugin](https://github.com/jenkinsci/kubernetes-plugin).

### Create a Jenkins instance with a persistent volume backing store:

```
$ oc new-app --template=jenkins-persistent --param=NAMESPACE=fedora-coreos --param=MEMORY_LIMIT=2Gi --param=VOLUME_CAPACITY=2Gi
```

Notice the `NAMESPACE` parameter. This makes the Jenkins master use the
image from our namespace, which we'll create in the next step. (The
reason we create the app first is that otherwise OpenShift will
automatically instantiate Jenkins with default parameters when creating
the Jenkins pipeline).

### Create the buildconfigs from the template

```
$ oc new-app --file=manifests/bc.yaml
```

If working on a private branch, you may override the
`REPO_URL` and `REPO_REF` parameters:

```
$ oc new-app --file=manifests/bc.yaml \
  --param=REPO_URL=https://github.com/jlebon/fedora-coreos-ci \
  --param=REPO_REF=my-feature-branch
```

Right now this template creates two buildconfigs:

1. The Jenkins master imagestream
2. The Jenkins pipeline build

We can now start a build of the Jenkins master:

```
oc start-build --follow fedora-coreos-jenkins
```

Once the Jenkins master image is built, Jenkins should start up (verify
with `oc get pods`).

It is a good idea to
[update all plugins](TROUBLESHOOTING.md#issue-for-plugins-not-being-up-to-date)
and 
[set the kubernetes plugin to use DNS for service names](TROUBLESHOOTING.md#issue-for-jenkins-dns-names).

### Create an image stream for the coreos-assembler container.

It's nice to use an image stream so that we can pull from the local container
registry rather than from a remote registry each time.

Also add `--scheduled` to the image so it gets updated when the remote
tag in the remote registry gets updated.

```
$ oc import-image coreos-assembler:master --from=quay.io/coreos-assembler/coreos-assembler:master --confirm
$ oc tag quay.io/coreos-assembler/coreos-assembler:master coreos-assembler:master --scheduled
```

### Create an image stream for the Jenkins slave container 

This is so that we don't have to pull it from the remote registry each time.
This is used for the `jnlp` container in the kubernetes plugin.

```
$ oc import-image jenkins-slave-base-centos7:latest --from=docker.io/openshift/jenkins-slave-base-centos7 --confirm
$ oc tag docker.io/openshift/jenkins-slave-base-centos7:latest jenkins-slave-base-centos7:latest --scheduled
```

### Create a PVC where we will store the cache and build artifacts

```
$ oc create -f manifests/pvc.yaml
```

### [PROD ONLY] Update the "secret" token values in the webooks to be unique

```
$ oc set triggers bc/fedora-coreos-pipeline --from-github
$ oc set triggers bc/fedora-coreos-pipeline --from-webhook
```

### [PROD ONLY] Set up webhooks/automation

Grab the URLs of the webhooks from `oc describe` and set up webhook
in github.

- `oc describe bc/fedora-coreos-pipeline` and grab the `Webhook GitHub` URL
- From the GitHub web console for the configs repository.
- Select Add Webhook from Settings → Webhooks & Services.
- Paste the webook URL output into the Payload URL field.
- Change the Content Type from GitHub’s default `application/x-www-form-urlencoded` to `application/json`.
- Click Add webhook.

### Start build using the CLI

```
$ oc start-build fedora-coreos-pipeline
```

Use the web interface to view logs from builds.

### [OPTIONAL] Set up simple-httpd

When hacking locally, it might be useful to look at the contents of the
PV to see the builds since one isn't rsync'ing to an artifact server.
One alternative to creating a "sleeper" pod with the PV mounted is to
expose a simple httpd server:

```
$ oc create -f manifests/simple-httpd.yaml
```

You'll then be able to browse the contents of the PV at:

```
http://simple-httpd-fedora-coreos.$CLUSTER_URL
```

(Or simply check the output of `oc get route simple-httpd`).