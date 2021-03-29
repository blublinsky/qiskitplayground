# Qiskit playground operator

## Prereqs

Build with the [SDK operator](https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/).
For installation, follow [Installation guide](https://sdk.operatorframework.io/docs/building-operators/golang/installation/)

## Development

Create project

````
operator-sdk init --domain ibm.com --repo github.io/blublinsky/qiskitplaygrounds
````
Install additional libraries
````
go get github.com/go-logr/logr@v0.3.0
go get github.com/onsi/ginkgo@v1.14.1
go get github.com/onsi/gomega@v1.10.2
go get k8s.io/api/core/v1@v0.19.2
go get github.com/openshift/api/route/v1
````
Create APIs

````
operator-sdk create api --group qiskit --version v1alpha1 --kind QiskitPlayground --resource --controller
````
## Overall approach

Implementation is based on [Jupyter images](https://jupyter-docker-stacks.readthedocs.io/en/latest/using/selecting.html)
A simple [Dockerfile](image/Dockerfile) looks like follows:

````
# Start from Jupyter image
FROM jupyter/scipy-notebook:584e9ab39d22
# Change permissions on data directory (for OpenShift).
# See https://developers.redhat.com/blog/2020/10/26/adapting-docker-and-kubernetes-containers-to-run-on-red-hat-openshift-container-platform/
USER root
RUN chgrp -R 0 /home/jovyan && \
    chmod -R 777 /home/jovyan
# Install Qiskit
USER $NB_UID
RUN pip install qiskit[visualization]
````

Here changing permission is necessary to be able to use this image on OpenShift, [see](https://developers.redhat.com/blog/2020/10/26/adapting-docker-and-kubernetes-containers-to-run-on-red-hat-openshift-container-platform/), which runs the pod as a user defined by project.
There is a prebuild image `blublinsky1/qiskit:0.1`

With this in place build image with the following command (assuming that you are in the project root directory)

````
docker build -t blublinsky1/qiskit:0.1 image
````

The CRD takes the following parameters:

````
	Image string `json:"image,omitempty"`
	ImagePullPolicy apiv1.PullPolicy `json:"imagePullPolicy,omitempty"`
	PVC string `json:"pvc,omitempty"`
	LoadBalancer bool `json:"loadbalancer,omitempty" description:"Define if load balancer service type is supported. By default false"`
  Resources *apiv1.ResourceRequirements `json:"resources,omitempty"`

````
The only required parameter is image, for example:

````
apiVersion: "qiskit.ibm.com/v1alpha1"
kind: QiskitPlayground
metadata:
  name: test
  namespace: jupyter
spec:
  image: "blublinsky1/qiskit:0.1"
````
If not provided Image pull policy defaults to `IfNotPresent`
PVC is a PVC which will be used for persistence of notebooks, for example:

````
apiVersion: "qiskit.ibm.com/v1alpha1"
kind: QiskitPlayground
metadata:
  name: test
  namespace: jupyter
spec:
  image: "blublinsky1/qiskit:0.1"
  pvc: "claim-admin"
````
If PVC is not defined, an internal Pod disk is used

Exposing operator UI, depends on where it runs. If running on OpenShift,
a [route](https://docs.openshift.com/container-platform/4.7/rest_api/network_apis/route-route-openshift-io-v1.html) is created for Playgrond.  
If it is vanilla kubernetes, by default a [ClusterIP](https://rtfm.co.ua/en/kubernetes-clusterip-vs-nodeport-vs-loadbalancer-services-and-ingress-an-overview-with-examples/) type service is created, and in order to
expose a service to user, you need to run `port-forward` on this service or define an `Ingress` for it. This works for `MiniKube` or `Kind`
kubernetes servers. Many of the cloud provider based kubernetes, for example EKS, GKE and Azure support `LoadBalancer` type service. For this type of service, a corresponding load balancer will be created automatically and ingress creation is not required. Defining loadbalancer parameter in the CR will create a loadbalancer service instead of cluster IP

````
apiVersion: "qiskit.ibm.com/v1alpha1"
kind: QiskitPlayground
metadata:
  name: test
  namespace: jupyter
spec:
  image: "blublinsky1/qiskit:0.1"
  loadbalancer: true

````

Finally resource allow to specify resources for the pod, for example:
````
apiVersion: "qiskit.ibm.com/v1alpha1"
kind: QiskitPlayground
metadata:
  name: test
  namespace: jupyter
spec:
  image: "blublinsky1/qiskit:0.1"
  pvc: "claim-admin"
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "128Mi"
      cpu: "500m"
````

To make implementation more reliable, operator is using a deployment with 1 replica,
which means that if the pod is deleted, it will be bring back by deployment.

When playground CR is deleted, it will delete all of the associated resources.
