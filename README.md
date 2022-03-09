# Process to copy contents from one PVC to another within MVI

This allow to migrate from different storage backends for a given project. The process would be:

 * Create a new PVC using the preferred storage backend
 * Build & Publish rsync image
 * Run rsync process job

## Prereqs
### Set environment
```
export INSTANCEID=xyz
export NAMESPACE=mas-$INSTANCEID-visualinspection
oc project $NAMESPACE
```
### Build image
```
docker build . -f docker/Dockerfile -t rsync
```
### Publish image to local OCP registry
* Expose route to internal registry
```
oc get routes -n openshift-image-registry
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
oc get routes -n openshift-image-registry
# route will show up after a bit
export ROUTE=$(oc get routes -n openshift-image-registry | xargs | awk '{print $9}')
```
* Publish to internal registry
```
export REGISTRY=$ROUTE/$NAMESPACE
docker login -u kubeadmin -p $(oc whoami -t) $ROUTE
docker tag rsync:latest $REGISTRY/rsync
docker push $REGISTRY/rsync
#save the checksum
CHECKSUM=sha256:xxxx
sed "s|\$NAMESPACE|${NAMESPACE}|;s|\$CHECKSUM|${CHECKSUM}|" job-template.yaml > myjob.yaml
```

## Usage
### Run rsync job
```
export SRC_PVC=$INSTANCEID-data-pvc
export DST_PVC=$INSTANCEID-data-pvc-clone
oc process -f myjob.yaml SOURCE_PVC=$SRC_PVC DEST_PVC=$DST_PVC | oc create -f -
```
