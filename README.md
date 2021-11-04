# Kubebuilder to build your first Kubernetes Operator
[toc]





# TL;DR


# Install Go
```bash
# https://golang.org/doc/install
# Download Go
wget -c https://dl.google.com/go/go1.16.7.linux-amd64.tar.gz

# Remove the old version and install the new version
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.16.7.linux-amd64.tar.gz

# Add /usr/local/go/bin to the PATH environment variable
# Add GOPATH=$HOME/go
# Add the following line to your $HOME/.profile
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
# Apply the change
source ~/.profile

# Verify Go version
go version
```
# Install Kubebuilder
```bash
# download kubebuilder and install locally.
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/
```
# Create Project
```bash
# Initlize go mod
go mod init github.com/wangxin311/kubebuilder-imoocpod

# Initilize kubebuilder framework
kubebuilder init --domain wangxin311.github.com --repo github.com/wangxin311/kubebuilder-imoocpod --skip-go-version-check

# Create Kubevirt API, choose yes
kubebuilder create api --group batch --version v1alpha1 --kind ImoocPod
```
And now kubebuilder has been generated the following files for us.

```bash
root@sv10-control-01:~/kubebuilder# tree
.
├── api
│   └── v1alpha1
│       ├── groupversion_info.go
│       ├── imoocpod_types.go
│       └── zz_generated.deepcopy.go
├── bin
│   └── controller-gen
├── config
│   ├── crd
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_imoocpods.yaml
│   │       └── webhook_in_imoocpods.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   └── manager_config_patch.yaml
│   ├── manager
│   │   ├── controller_manager_config.yaml
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── imoocpod_editor_role.yaml
│   │   ├── imoocpod_viewer_role.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── role_binding.yaml
│   │   └── service_account.yaml
│   └── samples
│       └── batch_v1alpha1_imoocpod.yaml
├── controllers
│   ├── imoocpod_controller.go
│   └── suite_test.go
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── PROJECT

13 directories, 37 files
```
# Opeator Type
In this example, we are going to create a CRD specify the number of busybox pod you'd like to run, when CRD instance updated, the number of POD will be updated accordingly.

CRD type: `api/v1alpha1/imoocpod_types.go`

```go
// ImoocPodSpec defines the desired state of ImoocPod
type ImoocPodSpec struct {
        // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
        // Important: Run "make" to regenerate code after modifying this file

        // Foo is an example field of ImoocPod. Edit imoocpod_types.go to remove/update
        Replicas int `json:"replicas"`
}

// ImoocPodStatus defines the observed state of ImoocPod
type ImoocPodStatus struct {
        // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
        // Important: Run "make" to regenerate code after modifying this file
        Replicas int      `json:"replicas"`
        PodNames []string `json:"podNames"`
}
```
ImoocPodSpec means desired state and  ImoocPodStatus means the actual state.

Generate CRD manifests `config/crd/bases/batch.schwarzeni.github.com_imoocpods.yaml`

```bash
make manifests
```
# Operator logic
```go
root@sv10-control-01:~/kubebuilder/kubebuilder-imoocpod# cat controllers/imoocpod_controller.go
/*
Copyright 2021.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
	"context"
	"fmt"
	"reflect"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"

	batchv1alpha1 "github.com/wangxin311/kubebuilder-imoocpod/api/v1alpha1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
)

// ImoocPodReconciler reconciles a ImoocPod object
type ImoocPodReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=batch.wangxin311.github.com,resources=imoocpods,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=batch.wangxin311.github.com,resources=imoocpods/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=batch.wangxin311.github.com,resources=imoocpods/finalizers,verbs=update
//+kubebuilder:rbac:groups="",resources=pods,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups="",resources=pods/status,verbs=get

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the ImoocPod object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.10.0/pkg/reconcile

var log = logf.Log.WithName("imoocpod")

func (r *ImoocPodReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	//logger := r.Log.WithValues("imoocpod", req.NamespacedName)
	logger := log.WithValues("Request.Namespace", req.Namespace, "Request.Name", req.Name)
	logger.Info("start reconcile")

	// fetch the ImoocPod instance
	instance := &batchv1alpha1.ImoocPod{}
	if err := r.Client.Get(ctx, req.NamespacedName, instance); err != nil {
		if errors.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}

	// 1. fetch pod list by name
	lbls := labels.Set{"app": instance.Name}
	existingPods := &corev1.PodList{}
	if err := r.Client.List(ctx, existingPods, &client.ListOptions{
		Namespace: req.Namespace, LabelSelector: labels.SelectorFromSet(lbls)}); err != nil {
		logger.Error(err, "fetching existing pods failed")
		return ctrl.Result{}, err
	}

	// 2. fetch pod name in pod list
	var existingPodNames []string
	for _, pod := range existingPods.Items {
		if pod.GetObjectMeta().GetDeletionTimestamp() != nil {
			continue
		}
		if pod.Status.Phase == corev1.PodRunning || pod.Status.Phase == corev1.PodPending {
			existingPodNames = append(existingPodNames, pod.GetObjectMeta().GetName())
		}
	}

	// 3. update actual state
	currStatus := batchv1alpha1.ImoocPodStatus{
		Replicas: len(existingPodNames),
		PodNames: existingPodNames,
	}
	if !reflect.DeepEqual(instance.Status, currStatus) {
		instance.Status = currStatus
		if err := r.Client.Status().Update(ctx, instance); err != nil {
			logger.Error(err, "update pod failed")
			return ctrl.Result{}, err
		}
	}

	// 4. pod.Spec.Replicas > running len(pod.replicas), scale up pod -> create pod
	if instance.Spec.Replicas > len(existingPodNames) {
		logger.Info(fmt.Sprintf("creating pod, current and expected num: %d %d", len(existingPodNames), instance.Spec.Replicas))
		pod := newPodForCR(instance)
		if err := controllerutil.SetControllerReference(instance, pod, r.Scheme); err != nil {
			logger.Error(err, "scale up failed: SetControllerReference")
			return ctrl.Result{}, err
		}
		if err := r.Client.Create(ctx, pod); err != nil {
			logger.Error(err, "scale up failed: create pod")
			return ctrl.Result{}, err
		}
	}

	// 5. pod.Spec.Replicas < running len(pod.replicas)，scale down -> delete pod
	if instance.Spec.Replicas < len(existingPodNames) {
		logger.Info(fmt.Sprintf("deleting pod, current and expected num: %d %d", len(existingPodNames), instance.Spec.Replicas))
		pod := existingPods.Items[0]
		existingPods.Items = existingPods.Items[1:]
		if err := r.Client.Delete(ctx, &pod); err != nil {
			logger.Error(err, "scale down faled")
			return ctrl.Result{}, err
		}
	}

	return ctrl.Result{Requeue: true}, nil
}

func newPodForCR(cr *batchv1alpha1.ImoocPod) *corev1.Pod {
	labels := map[string]string{"app": cr.Name}
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

// SetupWithManager sets up the controller with the Manager.
func (r *ImoocPodReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&batchv1alpha1.ImoocPod{}).
		Complete(r)
}
```


# Build and Deploy
```bash
make docker-build docker-push IMG=quay.io/kewang/controller:v3
make deploy IMG=quay.io/kewang/controller:v3
```
Verification

```bash
root@sv10-control-01:~/kubebuilder# kubectl get all -n kubebuilder-system
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/kubebuilder-controller-manager-94fb7f557-4wlp4   2/2     Running   0          22s

NAME                                                     TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/kubebuilder-controller-manager-metrics-service   ClusterIP   10.96.12.30   <none>        8443/TCP   22s

NAME                                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kubebuilder-controller-manager   1/1     1            1           22s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/kubebuilder-controller-manager-94fb7f557   1         1         1       22s
```
# Apply CRD
```bash
kubectl apply -f config/crd/bases/batch.schwarzeni.github.com_imoocpods.yaml
```
# Deploy CRD instance
```yaml
# config/samples/batch_v1alpha1_imoocpod.yaml
apiVersion: batch.wangxin311.github.com/v1alpha1
kind: ImoocPod
metadata:
  name: imoocpod-sample
spec:
  replicas: 5
```


```bash
root@sv10-control-01:~/kubebuilder/kubebuilder-imoocpod# kubectl apply -f config/samples/batch_v1alpha1_imoocpod.yaml
```
```bash
root@sv10-control-01:~/kubebuilder/kubebuilder-imoocpod# kubectl get all
NAME                                       READY   STATUS    RESTARTS   AGE
pod/imoocpod-sample-podd2tp2               1/1     Running   0          7m41s
pod/imoocpod-sample-podmvwpp               1/1     Running   0          7m
pod/imoocpod-sample-podqm8jm               1/1     Running   0          7m41s
pod/imoocpod-sample-podv5tk5               1/1     Running   0          7m41s
pod/imoocpod-sample-podw2554               1/1     Running   0          7m41s
```
