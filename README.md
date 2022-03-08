# Process to copy contents from one PVC to another

I created this to allow me to migrate from different storage backends for a given project. The process would be:

 * Create a new PVC using the preferred storage backend
 * Run sync process job

## Usage
### Set env variables
```
export INSTANCEID=xyz
export NAMESPACE=mas-$INSTANCEID-visualinspection
```
### Build image
```
docker build . -f docker/Dockerfile -t rsync
```
### Publish image to local OCP registry
* Expose route to internal registry (you only need to do this once per cluster!)
```
oc get routes -n openshift-image-registry
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
oc get routes -n openshift-image-registry
# route will show up after a bit
export ROUTE=$(oc get routes -n openshift-image-registry | xargs | awk '{print $9}')
```
* Login to internal registry
```
docker login -u kubeadmin -p $(oc whoami -t) $ROUTE
```
* Tag/publish to internal registry
```
export REGISTRY=$ROUTE/$NAMESPACE
docker tag rsync:latest $REGISTRY/rsync
docker push $REGISTRY/rsync
#save the checksum
CHECKSUM=sha256:xxxx
```
### Run job
```
#Update other variables based on INSTANCEID
export SRC_PVC=$INSTANCEID-data-pvc
export DST_PVC=$INSTANCEID-data-pvc-clone2
sed "s|\$NAMESPACE|${NAMESPACE}|;s|\$CHECKSUM|${CHECKSUM}|" job-template.yaml > myjob.yaml

#Create the rsync job
oc process -f myjob.yaml SOURCE_PVC=$SRC_PVC DEST_PVC=$DST_PVC | oc create -f -
```
