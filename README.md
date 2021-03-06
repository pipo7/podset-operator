# podset-operator - Follow below instructions ***Error in controller.go to be corrected later
The software versions might change so upgrade system and corresponding versions .

Link : https://www.katacoda.com/openshift/courses/operatorframework/go-operator-podset
Pre-requisites are:
-------------------
git
gcc
make
go version 1.15  ( Tried with go version 1.16 but it does not work with it )
docker version 17.03+.
kubectl and access to a Kubernetes cluster of a compatible version.
See :https://sdk.operatorframework.io/docs/building-operators/golang/installation/

Versions:
go version is 1.15.2
on CentOS8 
operator-sdk version: "v1.0.1", commit: "4169b318b578156ed56530f373d328276d040a1b", kubernetes version: "v1.18.2", go version: "go1.13.15 linux/amd64", GOOS: "linux", GOARCH: "amd64"
kubectl version 1.18+

Ensure that your GOPROXY is set to "https://proxy.golang.org|direct"

#Ensure make is installed 
$ sudo apt-get update -y && sudo apt-get install -y make

Below replace o with kubectl as we are using kubernetes and not Openshift

#Let's begin by creating a new project called myproject:
$ oc new-project myproject

#Let's now create a new directory for our project:
$ mkdir -p $HOME/projects/podset-operator &&  cd $HOME/projects/podset-operator

#Initialize a new Go-based Operator SDK project for the PodSet Operator:
$ operator-sdk init --domain=example.com --repo=github.com/redhat/podset-operator
OR
$ kubebuilder init --domain=example.com --repo=github.com/pipo7/podset-operator

#Run below if the dependency error comes , it may not come if it already uses latest modules
go get github.com/go-logr/logr@v0.1.0  && go get github.com/onsi/ginkgo@v1.11.0 && go get github.com/onsi/gomega@v1.8.1


#Add a new Custom Resource Definition (CRD) API called PodSet with APIVersion app.example.com/v1alpha1 and Kind PodSet. This command will also create our boilerplate controller logic and Kustomize configuration files.
$ operator-sdk create api --group=app --version=v1alpha1 --kind=PodSet --resource --controller
OR
$ kubebuilder create api \
 --group app \
 --version v1alpha1 \
 --controller \
 --resource \
 --kind PodSet
 

#We should now see the /api, config, and /controllers directories.
#Let's begin by inspecting the newly generated api/v1alpha1/podset_types.go file for our PodSet API:

cat api/v1alpha1/podset_types.go
#In Kubernetes, every functional object (with some exceptions, i.e. ConfigMap) includes spec and status. 
#Kubernetes functions by reconciling desired state (Spec) with the actual cluster state. We then record what is observed (Status).
#Also observe the +kubebuilder comment markers found throughout the file. 
#operator-sdk makes use of a tool called controler-gen (from the controller-tools project) for generating utility code and Kubernetes YAML. 
More information on markers for config/code generation can be found here.

#Let's now modify the PodSetSpec and PodSetStatus of the PodSet Custom Resource (CR) at api/v1alpha1/podset_types.go
#It should look like the file below:

package v1alpha1
import (
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// PodSetSpec defines the desired state of PodSet
type PodSetSpec struct {
        // Replicas is the desired number of pods for the PodSet
        // +kubebuilder:validation:Minimum=1
        // +kubebuilder:validation:Maximum=12
        Replicas int32 `json:"replicas,omitempty"`
}

// PodSetStatus defines the current status of PodSet and we want PodNames also so we add that
type PodSetStatus struct {
	//Names of the pods
        PodNames        []string        `json:"podNames"`
	// current state available replicas of pods
    AvailableReplicas    int32    `json:"availableReplicas"`
}
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
 
// PodSet is the Schema for the podsets API
// +kubebuilder:printcolumn:JSONPath=".spec.replicas",name=Desired,type=string
// +kubebuilder:printcolumn:JSONPath=".status.availableReplicas",name=Available,type=string
type PodSet struct {
        metav1.TypeMeta   `json:",inline"`
        metav1.ObjectMeta `json:"metadata,omitempty"`

        Spec   PodSetSpec   `json:"spec,omitempty"`
        Status PodSetStatus `json:"status,omitempty"`
}
// +kubebuilder:object:root=true
// PodSetList contains a list of PodSet
type PodSetList struct {
        metav1.TypeMeta `json:",inline"`
        metav1.ListMeta `json:"metadata,omitempty"`
        Items           []PodSet `json:"items"`
}
func init() {
        SchemeBuilder.Register(&PodSet{}, &PodSetList{})
}
#You can easily update this file by running the following command:
$ \cp /tmp/podset_types.go api/v1alpha1/podset_types.go  # this step works in katacoda excercise.

#After modifying the *_types.go file, always run the following command to update the zz_generated.deepcopy.go file:
#go to main podset folder and then run :
$ make generate

#Now we can run the make manifests command to generate our customized CRD and additional object YAMLs.
$ make manifests

#Thanks to our comment markers, observe that we now have a newly generated CRD yaml that reflects the spec.replicas and status.podNames OpenAPI v3 schema validation and customized print columns.
$ cat config/crd/bases/app.example.com_podsets.yaml

#Deploy your PodSet Custom Resource Definition to the live OpenShift Cluster:
$ oc apply -f config/crd/bases/app.example.com_podsets.yaml
##OR you can also run make install , whcih will aslo do same "kustomize build config/crd | kubectl apply -f -"
make install

#Confirm the CRD was successfully created:
$ oc get crd podsets.app.example.com -o yaml
$ kubectl get crds podsets.app.example.com -o yaml

#kubectl get crds podsets.app.example.com
NAME                      CREATED AT
podsets.app.example.com   2021-11-09T08:05:21Z


#Let's now observe the default controllers/podset_controller.go file:

$ cat controllers/podset_controller.go
#This default controller requires additional logic so we can trigger our reconciler whenever 
#kind: PodSet objects are added, updated, or deleted. We also want to trigger the reconciler whenever Pods owned 
#by a given PodSet are added, updated, and deleted as well. To accomplish this. we modify the controller's SetupWithManager method.
#Modify the PodSet controller logic at controllers/podset_controller.go:

package controllers

import (
        "context"
        "reflect"

        "github.com/go-logr/logr"
        "k8s.io/apimachinery/pkg/runtime"
        ctrl "sigs.k8s.io/controller-runtime"
        "sigs.k8s.io/controller-runtime/pkg/client"
	//Change below to your github repo path
        //So  appv1alpha1 "github.com/redhat/podset-operator/api/v1alpha1" will change to 
	appv1alpha1 "github.com/pipo7/podset-operator/api/v1alpha1"

        "k8s.io/apimachinery/pkg/labels"
        "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
        "sigs.k8s.io/controller-runtime/pkg/reconcile"

        "k8s.io/apimachinery/pkg/api/errors"
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

        corev1 "k8s.io/api/core/v1"
)


// PodSetReconciler reconciles a PodSet object
type PodSetReconciler struct {
        client.Client
        Log    logr.Logger
        Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=app.example.com,resources=podsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=app.example.com,resources=podsets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=v1,resources=pods,verbs=get;list;watch;create;update;patch;delete


func (r *PodSetReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
        _ = context.Background()
        _ = r.Log.WithValues("podset", req.NamespacedName)

        // Fetch the PodSet instance
        instance := &appv1alpha1.PodSet{}
        err := r.Get(context.TODO(), req.NamespacedName, instance)
        if err != nil {
                if errors.IsNotFound(err) {
                        // Request object not found, could have been deleted after reconcile request.
                        // Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
                        // Return and don't requeue
                        return reconcile.Result{}, nil
                }
                // Error reading the object - requeue the request.
                return reconcile.Result{}, err
        }
        // List all pods owned by this PodSet instance
        podSet := instance
        podList := &corev1.PodList{}
    	lbs := map[string]string{
                "app":     podSet.Name,
                "version": "v0.1",
        }
        labelSelector := labels.SelectorFromSet(lbs)
        listOps := &client.ListOptions{Namespace: podSet.Namespace, LabelSelector: labelSelector}
        if err = r.List(context.TODO(), podList, listOps); err != nil {
                return reconcile.Result{}, err
        }

        // Count the pods that are pending or running as available
        var available []corev1.Pod
        for _, pod := range podList.Items {
                if pod.ObjectMeta.DeletionTimestamp != nil {
                        continue
                }
                if pod.Status.Phase == corev1.PodRunning || pod.Status.Phase == corev1.PodPending {
                        available = append(available, pod)
                }
        }
        numAvailable := int32(len(available))
        availableNames := []string{}
        for _, pod := range available {
                availableNames = append(availableNames, pod.ObjectMeta.Name)
        }

    // Update the status if necessary
        status := appv1alpha1.PodSetStatus{
                PodNames: availableNames,
                AvailableReplicas: numAvailable,
        }
        if !reflect.DeepEqual(podSet.Status, status) {
                podSet.Status = status
                err = r.Status().Update(context.TODO(), podSet)
                if err != nil {
                        r.Log.Error(err, "Failed to update PodSet status")
                        return reconcile.Result{}, err
                }
        }

        if numAvailable > podSet.Spec.Replicas {
                r.Log.Info("Scaling down pods", "Currently available", numAvailable, "Required replicas", podSet.Spec.Replicas)
                diff := numAvailable - podSet.Spec.Replicas
                dpods := available[:diff]
                for _, dpod := range dpods {
                        err = r.Delete(context.TODO(), &dpod)
                        if err != nil {
                                r.Log.Error(err, "Failed to delete pod", "pod.name", dpod.Name)
    return reconcile.Result{}, err
                        }
                }
                return reconcile.Result{Requeue: true}, nil
        }

        if numAvailable < podSet.Spec.Replicas {
                r.Log.Info("Scaling up pods", "Currently available", numAvailable, "Required replicas", podSet.Spec.Replicas)
                // Define a new Pod object
                pod := newPodForCR(podSet)
                // Set PodSet instance as the owner and controller
                if err := controllerutil.SetControllerReference(podSet, pod, r.Scheme); err != nil {
                        return reconcile.Result{}, err
                }
                err = r.Create(context.TODO(), pod)
                if err != nil {
                        r.Log.Error(err, "Failed to create pod", "pod.name", pod.Name)
                        return reconcile.Result{}, err
                }
                return reconcile.Result{Requeue: true}, nil
        }

        return reconcile.Result{}, nil
}


// newPodForCR returns a busybox pod with the same name/namespace as the cr
func newPodForCR(cr *appv1alpha1.PodSet) *corev1.Pod {
        labels := map[string]string{
                "app":     cr.Name,
                "version": "v0.1",
        }
        return &corev1.Pod{
                ObjectMeta: metav1.ObjectMeta{
                        GenerateName: cr.Name + "-pod",
                        Namespace:    cr.Namespace,
                        Labels:       labels,
                },
                Spec: corev1.PodSpec{
                        Containers: []corev1.Container{
                                {
                                        Name:    "busybox",
                                        Image:   "busybox",
                                        Command: []string{"sleep", "3600"},
                                },
                        },
                },
        }
}
//SetupWithManager defines how the controller will watch for resources
func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
        return ctrl.NewControllerManagedBy(mgr).
                For(&appv1alpha1.PodSet{}).
                Owns(&corev1.Pod{}).
                Complete(r)
}

#You can easily update this file by running the following command:
$ \cp /tmp/podset_controller.go controllers/podset_controller.go


##go mod tidy ensures that the go.mod file matches the source code in the module. It adds any missing module requirements necessary to build the current module's packages and dependencies, and it removes requirements on modules that don't provide any relevant packages. It also adds any missing entries to go.sum and removes unnecessary entries.
go mod tidy

#Once the CRD is registered, there are two ways to run the Operator:
#As a Pod inside a Kubernetes cluster
#As a Go program outside the cluster using Operator-SDK. This is great for local development of your Operator.
#For the sake of this tutorial, we will run the Operator as a Go program outside the cluster using Operator-SDK and our kubeconfig credentials

$ WATCH_NAMESPACE=myproject make run 
#OR
$ make run &
##if you get any go library depdency then please add that...
If you get ERROR " the package is not in /usr/local/go/src/..." then copy the package there . However this error should not come please check go module paths and settting in ~/profile
#~/go/src$ ls
#podset-operator
#$ cp -r podset-operator/ /usr/local/go/src/


I got errors for 
go get k8s.io/api/core/v1@v0.17.2

#Creating the new CR now nased on CRD
#In a new terminal, inspect the Custom Resource manifest:

$ cd $HOME/projects/podset-operator
$ cat config/samples/app_v1alpha1_podset.yaml

#Ensure your kind: PodSet Custom Resource (CR) is updated with spec.replicas
apiVersion: app.example.com/v1alpha1
kind: PodSet
metadata:
  name: podset-sample
spec:
  replicas: 3
You can easily update this file by running the following command:
$ \cp /tmp/app_v1alpha1_podset.yaml config/samples/app_v1alpha1_podset.yaml


Ensure you are currently scoped to the myproject Namespace:
$ oc project myproject

Deploy your PodSet Custom Resource to the live OpenShift Cluster:
$ oc create -f config/samples/app_v1alpha1_podset.yaml

Verify the Podset exists:
$ oc get podset

Verify the PodSet operator has created 3 pods:
$ oc get pods

kubectl apply -f config/samples/app_v1alpha1_podset.yaml
podset.app.example.com/podset-sample created
ps@NOIHSM1DOP-PS42:~/go/src/podset-operator$ kubectl get podset
NAME            DESIRED   AVAILABLE
podset-sample   3         3
ps@NOIHSM1DOP-PS42:~/go/src/podset-operator$ kubectl get podsets
NAME            DESIRED   AVAILABLE
podset-sample   3         3

kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/podset-sample-pod9jbtf   2/2     Running   0          46s
pod/podset-sample-poddkp2r   2/2     Running   0          49s
pod/podset-sample-podzdrrc   2/2     Running   0          49s


Verify that status shows the name of the pods currently owned by the PodSet:
$ oc get podset podset-sample -o yaml

Increase the number of replicas owned by the PodSet:
$ oc patch podset podset-sample --type='json' -p '[{"op": "replace", "path": "/spec/replicas", "value":5}]'

$ kubectl patch podset podset-sample --type='json' -p '[{"op": "replace", "path": "/spec/replicas", "value":5}]'
podset.app.example.com/podset-sample patched

Verify that we now have 5 running pods
$ oc get pods
$ kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/podset-sample-pod9jbtf   2/2     Running   0          2m27s
pod/podset-sample-poddkp2r   2/2     Running   0          2m30s
pod/podset-sample-podhc82j   2/2     Running   0          25s
pod/podset-sample-podq7tqt   2/2     Running   0          25s
pod/podset-sample-podzdrrc   2/2     Running   0          2m30s

Our PodSet controller creates pods containing OwnerReferences in their metadata section. 
This ensures they will be removed upon deletion of the podset-sample CR.
Observe the OwnerReference set on a Podset's pod:

$ oc get pods -o yaml | grep ownerReferences -A10
$ kubectl get po -o yaml | grep ownerReferences -A10

Delete the podset-sample Custom Resource:
$ oc delete podset podset-sample
Thanks to OwnerReferences, all of the pods should be deleted:

$ oc get pods
