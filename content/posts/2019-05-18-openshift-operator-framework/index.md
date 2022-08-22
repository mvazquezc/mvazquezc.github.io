---
title:  "Writing Operators using the Operator Framework SDK"
#description: "Learn how Kubernetes operators are created"
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
date: 2019-05-18
lastmod: 2021-12-01
draft: false
author: "Mario"
tags: [ "okd", "origin", "containers", "kubernetes", "operators", "controllers", "operator framework", "operator sdk", "openshift", "ocp" ]
url: "/writing-operators-using-operator-framework/"
---

# Operators, operators everywhere

As you may have noticed, **Kubernetes operators** are becoming more an more popular those days. In this post we are going to explain the basics around Operators and we will develop a simple Operator using the **Operator Framework SDK**.

# What is an Operator

An operator aims to automate actions usually performed manually while lessening the likelihood of error and simplifying complexity.

We can think of an operator as a method of packaging, deploying and managing a Kubernetes enabled application. Kubernetes enabled applications are deployed on Kubernetes and managed using the Kubernetes APIs and tooling.

Kubernetes APIs can be extended in order to enable new types of Kubernetes enabled applications. We could say that Operators are the runtime that manages such applications.

A simple Operator would define how to deploy an application, whereas an advanced one will also take care of day-2 operations like backup, upgrades, etc.

Operators use the **Controller pattern**, but not all Controllers are Operators. We could say it's an Operator if it's got:

* Controller Pattern
* API Extension
* Single-App Focus

Feel free to read more about operators on the [Operator FAQ by CoreOS](https://coreos.com/operators/)


# Kubernetes Controllers

In the Kubernetes world, the **Controllers** take care of routine tasks to ensure cluster's observed state, matches cluster's desired state.

Each Controller is responsible for a particular resource in Kubernetes. The Controller runs a control loop that watches the shared state of the cluster through the Kubernetes API server and makes changes attempting to move the current state towards the desired state.

Some examples:

* Replication Controller
* Cronjob Controller

## Controller Components

There are two main components in a controller: `Informer/SharedInformer` and `WorkQueue`.

### Informer

In order to retrieve information about an object, the Controller sends a request to the Kubernetes API server. However, querying the API repeatedly can become expensive when dealing with thousands of objects.

On top of that, the Controller doesn't really need to send requests continuously. It only cares about CRUD events happening on the objects it's managing.

Informers are not much used in the current Kubernetes, instead SharedInformers are used.

### SharedInformer

A Informer creates a local cache for a set of resources used by itself. In Kubernetes there are multiple controllers running and caring about multiple kinds of resources though.

Having a shared cache among Controllers instead of one cache for each Controller sounds like a plan, that's a SharedInformer.

### WorkQueue

The `SharedInformer` can't track what each Controller is up to, so the Controller must provide its own queuing and retrying mechanism.

Whenever a resource changes, the SharedInformer's Event Handler puts a key into the `WorkQueue` so the Controller will take care of that change.

## How a Controller Works

### Control Loop

Every controller has a **Control Loop** which basically does:

1. Processes every single item from the WorkQueue
2. Pops an item and do whatever it needs to do with that item
3. Pushes the item back to the WorkQueue if required
4. Updates the item status to reflect the new changes
5. Starts over

**Code Examples**

* [https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L180](https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L180)
* [https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L187](https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L187)

### WorkQueue

1. Stuff is put into the WorkQueue
2. Stuff is take out from the WorkQueue in the Control Loop
3. WorkQueue doesn't store objects, it stores `MetaNamespaceKeys`

A MetaNamespaceKey is a key-value reference for an object. It has the namespace for the resource and the name for the resource.

**Code Examples**

* [https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L111](https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L111)
* [https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L187](https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L187)

### SharedInformer

As we said before, is a shared data cache which distributes the data to all the `Listers` interested in knowing about changes happening to specific objects.

The most important part of the `SharedInformer` are the `EventHandlers`. Using an `EventHandler` is how you register your interest in specific object updates like addition, creation, updation or deletion.

When an update occurs, the object will be put into the WorkQueue so it gets processed by the Controller in the Control Loop.

`Listers` are an important part of the `SharedInformers` as well. `Listers` are designed specifically to be used within Controllers as they have access to the cache.

**Listers vs Client-go**

Listers have access to the cache whereas Client-go will hit the Kubernetes API server (which is expensive when dealing with thousands of objects).

**Code Examples**

* [https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L252](https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L252)
* [https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L274](https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L274)

## SyncHandler A.K.A Reconciliation Loop

The first invocation of the `SyncHandler` will always be getting the `MetaNamespaceKey` for the resource it needs to work with.

With the `MetaNamespaceKey` the object is gathered from the cache, but well.. it's not really an object, but a pointer to the cached object.

With the object reference we can read the object, in case the object needs to be updated, then the object have to be DeepCopied. `DeepCopy` is an expensive operation, making sure the object will be modified before calling `DeepCopy` is a good practice.

With the object reference / DeepCopy we are ready to apply our business logic.

**Code Examples**

* [https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L243](https://github.com/kubernetes/sample-controller/blob/release-1.18/controller.go#L243)

## Kubernetes Controllers

Some information about controllers:

* Cronjob controller is probably the smallest one out there
* [Sample Controller](https://github.com/kubernetes/sample-controller) will help you getting started with Kubernetes Controllers

# Writing your very first Operator using the Operator Framework SDK

We will create a very simple Operator using the [Operator Framework SDK](https://github.com/operator-framework/operator-sdk).

The Operator will be in charge of deploying a simple [GoLang application](https://github.com/mvazquezc/reverse-words).

## Requirements

At the moment of this writing the following versions were used:

* golang-1.16.8
* Operator Framework SDK v1.0.0
* Kubernetes 1.21

## Installing the Operator Framework SDK

~~~sh
RELEASE_VERSION=v1.15.0
# Linux
sudo curl -L https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk_linux_amd64 -o /usr/local/bin/operator-sdk
sudo chmod +x /usr/local/bin/operator-sdk
~~~

## Initializing the Operator Project

First, a new new project for our Operator will be initialized.

~~~sh
mkdir -p ~/operators-projects/reverse-words-operator && cd $_
export GO111MODULE=on
export GOPROXY=https://proxy.golang.org
export GH_USER=<github_user>
operator-sdk init --domain=linuxera.org --repo=github.com/$GH_USER/reverse-words-operator
~~~

## Create the Operator API Types

As previously discussed, Operators extend the Kubernetes API, the API itself is organized in groups and versions. Our Operator will define a new Group, object Kind and its versioning.

In the example below we will define a new API Group called `apps` under domain `linuxera.org`, a new object Kind `ReverseWordsApp` and its versioning `v1alpha1`.

~~~sh
operator-sdk create api --group=apps --version=v1alpha1 --kind=ReverseWordsApp --resource=true --controller=true
~~~

Now it's time to define the structure of our new Object. The Spec properties that we will be using are:

* `replicas`: Will be used to define the number of replicas for our application
* `appVersion`: Will be used to define which version of the application is deployed

In the Status we will use:

* `appPods`: Will track the pods associated to our current ReverseWordsApp instance
* Different conditions

Below the code for our Types:

~~~go
/*


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

package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// ReverseWordsAppSpec defines the desired state of ReverseWordsApp
type ReverseWordsAppSpec struct {
	Replicas   int32  `json:"replicas"`
	AppVersion string `json:"appVersion,omitempty"`
}

// ReverseWordsAppStatus defines the observed state of ReverseWordsApp
type ReverseWordsAppStatus struct {
	AppPods    []string           `json:"appPods"`
	Conditions []metav1.Condition `json:"conditions"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

// ReverseWordsApp is the Schema for the reversewordsapps API
type ReverseWordsApp struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ReverseWordsAppSpec   `json:"spec,omitempty"`
	Status ReverseWordsAppStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// ReverseWordsAppList contains a list of ReverseWordsApp
type ReverseWordsAppList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []ReverseWordsApp `json:"items"`
}

func init() {
	SchemeBuilder.Register(&ReverseWordsApp{}, &ReverseWordsAppList{})
}

// Conditions
const (
	// ConditionTypeReverseWordsDeploymentNotReady indicates if the Reverse Words Deployment is not ready

	ConditionTypeReverseWordsDeploymentNotReady string = "ReverseWordsDeploymentNotReady"

	// ConditionTypeReady indicates if the Reverse Words Deployment is ready
	ConditionTypeReady string = "Ready"
)
~~~

You can download the Types file:

~~~sh
curl -Ls https://linuxera.org/writing-operators-using-operator-framework/reversewordsapp_types.go -o ~/operators-projects/reverse-words-operator/api/v1alpha1/reversewordsapp_types.go
~~~

Replicas will be defined as an `int32` and will reference the Spec property `replicas`. For the status AppPods will be defined as a `stringList` and will reference the Status property `appPods`.

With above changes in-place we need to add new dependencies and re-generate some boilerplate code to take into account the latest changes in our types.

~~~sh
go mod tidy
make manifests
make generate
~~~

## Code your Operator business logic

An empty controller (well, not that empty) has been created into our project, now it's time to modify it so it actually deploys our application the way we want.

Our application consists of a Deployment and a Service, so our Operator will deploy the Reverse Words App as follows:

1. A Kubernetes Deployment object will be created
2. A Kubernetes Service object will be created

Below code (commented) for our Controller:

~~~go
/*


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
	"reflect"

	"github.com/go-logr/logr"
	appsv1alpha1 "github.com/mvazquezc/reverse-words-operator/api/v1alpha1"
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/meta"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/apimachinery/pkg/util/intstr"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	ctrllog "sigs.k8s.io/controller-runtime/pkg/log"
)

// ReverseWordsAppReconciler reconciles a ReverseWordsApp object
type ReverseWordsAppReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

// Finalizer for our objects
const reverseWordsAppFinalizer = "finalizer.reversewordsapp.apps.linuxera.org"

// +kubebuilder:rbac:groups=apps.rha.lab,resources=pacmangames,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps.rha.lab,resources=pacmangames/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps.rha.lab,resources=pacmangames/finalizers,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch
// +kubebuilder:rbac:groups=core,resources=serviceaccounts,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=rbac.authorization.k8s.io,resources=clusterroles,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=rbac.authorization.k8s.io,resources=clusterrolebindings,verbs=get;list;watch;create;update;patch;delete

func (r *ReverseWordsAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := ctrllog.FromContext(ctx)

	// Fetch the ReverseWordsApp instance
	instance := &appsv1alpha1.ReverseWordsApp{}
	err := r.Get(ctx, req.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			log.Info("ReverseWordsApp resource not found. Ignoring since object must be deleted")
			return ctrl.Result{}, nil
		}
		// Error reading the object - requeue the request.
		log.Error(err, "Failed to get ReverseWordsApp")
		return ctrl.Result{}, err
	}

	// Check if the CR is marked to be deleted
	isInstanceMarkedToBeDeleted := instance.GetDeletionTimestamp() != nil
	if isInstanceMarkedToBeDeleted {
		log.Info("Instance marked for deletion, running finalizers")
		if contains(instance.GetFinalizers(), reverseWordsAppFinalizer) {
			// Run the finalizer logic
			err := r.finalizeReverseWordsApp(log, instance)
			if err != nil {
				// Don't remove the finalizer if we failed to finalize the object
				return ctrl.Result{}, err
			}
			log.Info("Instance finalizers completed")
			// Remove finalizer once the finalizer logic has run
			controllerutil.RemoveFinalizer(instance, reverseWordsAppFinalizer)
			err = r.Update(ctx, instance)
			if err != nil {
				// If the object update fails, requeue
				return ctrl.Result{}, err
			}
		}
		log.Info("Instance can be deleted now")
		return ctrl.Result{}, nil
	}

	// Add Finalizers to the CR
	if !contains(instance.GetFinalizers(), reverseWordsAppFinalizer) {
		if err := r.addFinalizer(log, instance); err != nil {
			return ctrl.Result{}, err
		}
	}

	// Reconcile Deployment object
	result, err := r.reconcileDeployment(instance, log)
	if err != nil {
		return result, err
	}
	// Reconcile Service object
	result, err = r.reconcileService(instance, log)
	if err != nil {
		return result, err
	}

	// The CR status is updated in the Deployment reconcile method

	return ctrl.Result{}, nil
}

func (r *ReverseWordsAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&appsv1alpha1.ReverseWordsApp{}).
		Owns(&appsv1.Deployment{}).
		Owns(&corev1.Service{}).
		Complete(r)
}

func (r *ReverseWordsAppReconciler) reconcileDeployment(cr *appsv1alpha1.ReverseWordsApp, log logr.Logger) (ctrl.Result, error) {
	// Define a new Deployment object
	deployment := newDeploymentForCR(cr)

	// Set ReverseWordsApp instance as the owner and controller of the Deployment
	if err := ctrl.SetControllerReference(cr, deployment, r.Scheme); err != nil {
		return ctrl.Result{}, err
	}

	// Check if this Deployment already exists
	deploymentFound := &appsv1.Deployment{}
	err := r.Get(context.Background(), types.NamespacedName{Name: deployment.Name, Namespace: deployment.Namespace}, deploymentFound)
	if err != nil && errors.IsNotFound(err) {
		log.Info("Creating a new Deployment", "Deployment.Namespace", deployment.Namespace, "Deployment.Name", deployment.Name)
		err = r.Create(context.Background(), deployment)
		if err != nil {
			return ctrl.Result{}, err
		}
		// Requeue the object to update its status
		return ctrl.Result{Requeue: true}, nil
	} else if err != nil {
		return ctrl.Result{}, err
	} else {
		// Deployment already exists
		log.Info("Deployment already exists", "Deployment.Namespace", deploymentFound.Namespace, "Deployment.Name", deploymentFound.Name)
	}

	// Ensure deployment replicas match the desired state
	if !reflect.DeepEqual(deploymentFound.Spec.Replicas, deployment.Spec.Replicas) {
		log.Info("Current deployment replicas do not match ReverseWordsApp configured Replicas")
		// Update the replicas
		err = r.Update(context.Background(), deployment)
		if err != nil {
			log.Error(err, "Failed to update Deployment.", "Deployment.Namespace", deploymentFound.Namespace, "Deployment.Name", deploymentFound.Name)
			return ctrl.Result{}, err
		}
	}
	// Ensure deployment container image match the desired state, returns true if deployment needs to be updated
	if checkDeploymentImage(deploymentFound, deployment) {
		log.Info("Current deployment image version do not match ReverseWordsApp configured version")
		// Update the image
		err = r.Update(context.Background(), deployment)
		if err != nil {
			log.Error(err, "Failed to update Deployment.", "Deployment.Namespace", deploymentFound.Namespace, "Deployment.Name", deploymentFound.Name)
			return ctrl.Result{}, err
		}
	}

	// Check if the deployment is ready
	deploymentReady := isDeploymentReady(deploymentFound)

	// Create list options for listing deployment pods
	podList := &corev1.PodList{}
	listOpts := []client.ListOption{
		client.InNamespace(deploymentFound.Namespace),
		client.MatchingLabels(deploymentFound.Labels),
	}
	// List the pods for this ReverseWordsApp deployment
	err = r.List(context.Background(), podList, listOpts...)
	if err != nil {
		log.Error(err, "Failed to list Pods.", "Deployment.Namespace", deploymentFound.Namespace, "Deployment.Name", deploymentFound.Name)
		return ctrl.Result{}, err
	}
	// Get running Pods from listing above (if any)
	podNames := getRunningPodNames(podList.Items)
	if deploymentReady {
		// Update the status to ready
		cr.Status.AppPods = podNames
		meta.SetStatusCondition(&cr.Status.Conditions, metav1.Condition{Type: appsv1alpha1.ConditionTypeReverseWordsDeploymentNotReady, Status: metav1.ConditionFalse, Reason: appsv1alpha1.ConditionTypeReverseWordsDeploymentNotReady})
		meta.SetStatusCondition(&cr.Status.Conditions, metav1.Condition{Type: appsv1alpha1.ConditionTypeReady, Status: metav1.ConditionTrue, Reason: appsv1alpha1.ConditionTypeReady})
	} else {
		// Update the status to not ready
		cr.Status.AppPods = podNames
		meta.SetStatusCondition(&cr.Status.Conditions, metav1.Condition{Type: appsv1alpha1.ConditionTypeReverseWordsDeploymentNotReady, Status: metav1.ConditionTrue, Reason: appsv1alpha1.ConditionTypeReverseWordsDeploymentNotReady})
		meta.SetStatusCondition(&cr.Status.Conditions, metav1.Condition{Type: appsv1alpha1.ConditionTypeReady, Status: metav1.ConditionFalse, Reason: appsv1alpha1.ConditionTypeReady})
	}
	// Reconcile the new status for the instance
	cr, err = r.updateReverseWordsAppStatus(cr, log)
	if err != nil {
		log.Error(err, "Failed to update ReverseWordsApp Status.")
		return ctrl.Result{}, err
	}
	// Deployment reconcile finished
	return ctrl.Result{}, nil
}

// updateReverseWordsAppStatus updates the Status of a given CR
func (r *ReverseWordsAppReconciler) updateReverseWordsAppStatus(cr *appsv1alpha1.ReverseWordsApp, log logr.Logger) (*appsv1alpha1.ReverseWordsApp, error) {
	reverseWordsApp := &appsv1alpha1.ReverseWordsApp{}
	err := r.Get(context.Background(), types.NamespacedName{Name: cr.Name, Namespace: cr.Namespace}, reverseWordsApp)
	if err != nil {
		return reverseWordsApp, err
	}

	if !reflect.DeepEqual(cr.Status, reverseWordsApp.Status) {
		log.Info("Updating ReverseWordsApp Status.")
		// We need to update the status
		err = r.Status().Update(context.Background(), cr)
		if err != nil {
			return cr, err
		}
		updatedReverseWordsApp := &appsv1alpha1.ReverseWordsApp{}
		err = r.Get(context.Background(), types.NamespacedName{Name: cr.Name, Namespace: cr.Namespace}, updatedReverseWordsApp)
		if err != nil {
			return cr, err
		}
		cr = updatedReverseWordsApp.DeepCopy()
	}
	return cr, nil

}

// addFinalizer adds a given finalizer to a given CR
func (r *ReverseWordsAppReconciler) addFinalizer(log logr.Logger, cr *appsv1alpha1.ReverseWordsApp) error {
	log.Info("Adding Finalizer for the ReverseWordsApp")
	controllerutil.AddFinalizer(cr, reverseWordsAppFinalizer)

	// Update CR
	err := r.Update(context.Background(), cr)
	if err != nil {
		log.Error(err, "Failed to update ReverseWordsApp with finalizer")
		return err
	}
	return nil
}

// finalizeReverseWordsApp runs required tasks before deleting the objects owned by the CR
func (r *ReverseWordsAppReconciler) finalizeReverseWordsApp(log logr.Logger, cr *appsv1alpha1.ReverseWordsApp) error {
	// TODO(user): Add the cleanup steps that the operator
	// needs to do before the CR can be deleted. Examples
	// of finalizers include performing backups and deleting
	// resources that are not owned by this CR, like a PVC.
	log.Info("Successfully finalized ReverseWordsApp")
	return nil
}

func (r *ReverseWordsAppReconciler) reconcileService(cr *appsv1alpha1.ReverseWordsApp, log logr.Logger) (ctrl.Result, error) {
	// Define a new Service object
	service := newServiceForCR(cr)

	// Set ReverseWordsApp instance as the owner and controller of the Service
	if err := controllerutil.SetControllerReference(cr, service, r.Scheme); err != nil {
		return ctrl.Result{}, err
	}

	// Check if this Service already exists
	serviceFound := &corev1.Service{}
	err := r.Get(context.Background(), types.NamespacedName{Name: service.Name, Namespace: service.Namespace}, serviceFound)
	if err != nil && errors.IsNotFound(err) {
		log.Info("Creating a new Service", "Service.Namespace", service.Namespace, "Service.Name", service.Name)
		err = r.Create(context.Background(), service)
		if err != nil {
			return ctrl.Result{}, err
		}
		// Service created successfully - don't requeue
		return ctrl.Result{}, nil
	} else if err != nil {
		return ctrl.Result{}, err
	} else {
		// Service already exists
		log.Info("Service already exists", "Service.Namespace", serviceFound.Namespace, "Service.Name", serviceFound.Name)
	}
	// Service reconcile finished
	return ctrl.Result{}, nil
}

// Returns a new deployment without replicas configured
// replicas will be configured in the sync loop
func newDeploymentForCR(cr *appsv1alpha1.ReverseWordsApp) *appsv1.Deployment {
	labels := map[string]string{
		"app": cr.Name,
	}
	replicas := cr.Spec.Replicas
	// Minimum replicas will be 1
	if replicas == 0 {
		replicas = 1
	}
	appVersion := "latest"
	if cr.Spec.AppVersion != "" {
		appVersion = cr.Spec.AppVersion
	}
	// TODO:Check if application version exists
	containerImage := "quay.io/mavazque/reversewords:" + appVersion
	probe := &corev1.Probe{
		Handler: corev1.Handler{
			HTTPGet: &corev1.HTTPGetAction{
				Path: "/health",
				Port: intstr.FromInt(8080),
			},
		},
		InitialDelaySeconds: 5,
		TimeoutSeconds:      2,
		PeriodSeconds:       15,
	}
	return &appsv1.Deployment{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "apps/v1",
			Kind:       "Deployment",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      "dp-" + cr.Name,
			Namespace: cr.Namespace,
			Labels:    labels,
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: &replicas,
			Selector: &metav1.LabelSelector{
				MatchLabels: labels,
			},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: labels,
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{
						{
							Image: containerImage,
							Name:  "reversewords",
							Ports: []corev1.ContainerPort{
								{
									ContainerPort: 8080,
									Name:          "reversewords",
								},
							},
							LivenessProbe:  probe,
							ReadinessProbe: probe,
						},
					},
				},
			},
		},
	}
}

// Returns a new service
func newServiceForCR(cr *appsv1alpha1.ReverseWordsApp) *corev1.Service {
	labels := map[string]string{
		"app": cr.Name,
	}
	return &corev1.Service{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "v1",
			Kind:       "Service",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      "service-" + cr.Name,
			Namespace: cr.Namespace,
			Labels:    labels,
		},
		Spec: corev1.ServiceSpec{
			Type:     corev1.ServiceTypeLoadBalancer,
			Selector: labels,
			Ports: []corev1.ServicePort{
				{
					Name: "http",
					Port: 8080,
				},
			},
		},
	}
}

// isDeploymentReady returns a true bool if the deployment has all its pods ready
func isDeploymentReady(deployment *appsv1.Deployment) bool {
	configuredReplicas := deployment.Status.Replicas
	readyReplicas := deployment.Status.ReadyReplicas
	deploymentReady := false
	if configuredReplicas == readyReplicas {
		deploymentReady = true
	}
	return deploymentReady
}

// getRunningPodNames returns the pod names for the pods running in the array of pods passed in
func getRunningPodNames(pods []corev1.Pod) []string {
	// Create an empty []string, so if no podNames are returned, instead of nil we get an empty slice
	var podNames []string = make([]string, 0)
	for _, pod := range pods {
		if pod.GetObjectMeta().GetDeletionTimestamp() != nil {
			continue
		}
		if pod.Status.Phase == corev1.PodPending || pod.Status.Phase == corev1.PodRunning {
			podNames = append(podNames, pod.Name)
		}
	}
	return podNames
}

// checkDeploymentImage returns wether the deployment image is different or not
func checkDeploymentImage(current *appsv1.Deployment, desired *appsv1.Deployment) bool {
	for _, curr := range current.Spec.Template.Spec.Containers {
		for _, des := range desired.Spec.Template.Spec.Containers {
			// Only compare the images of containers with the same name
			if curr.Name == des.Name {
				if curr.Image != des.Image {
					return true
				}
			}
		}
	}
	return false
}

// contains returns true if a string is found on a slice
func contains(list []string, s string) bool {
	for _, v := range list {
		if v == s {
			return true
		}
	}
	return false
}
~~~

You can download the controller code, remember to change the GitHub ID before bulding the operator:

~~~sh
# Remember to change import: appsv1alpha1 "github.com/mvazquezc/reverse-words-operator/api/v1alpha1"

curl -Ls https://linuxera.org/writing-operators-using-operator-framework/reversewordsapp_controller.go -o ~/operators-projects/reverse-words-operator/controllers/reversewordsapp_controller.go
~~~

## Setup Watch namespaces

By default, the controller will watch all namespaces, in this case we want it to watch only the namespace where it runs, in order to do so we need to update the Controler options in the `main.go` file.

~~~go
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

package main

import (
	"flag"
	"fmt"
	"os"

	// Import all Kubernetes client auth plugins (e.g. Azure, GCP, OIDC, etc.)
	// to ensure that exec-entrypoint and run can make use of them.
	_ "k8s.io/client-go/plugin/pkg/client/auth"

	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	clientgoscheme "k8s.io/client-go/kubernetes/scheme"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/healthz"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"

	appsv1alpha1 "github.com/mvazquezc/reverse-words-operator/api/v1alpha1"
	"github.com/mvazquezc/reverse-words-operator/controllers"
	//+kubebuilder:scaffold:imports
)

var (
	scheme   = runtime.NewScheme()
	setupLog = ctrl.Log.WithName("setup")
)

func init() {
	utilruntime.Must(clientgoscheme.AddToScheme(scheme))

	utilruntime.Must(appsv1alpha1.AddToScheme(scheme))
	//+kubebuilder:scaffold:scheme
}

func main() {
	var metricsAddr string
	var enableLeaderElection bool
	var probeAddr string
	flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
	flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
	flag.BoolVar(&enableLeaderElection, "leader-elect", false,
		"Enable leader election for controller manager. "+
			"Enabling this will ensure there is only one active controller manager.")
	opts := zap.Options{
		Development: true,
	}
	opts.BindFlags(flag.CommandLine)
	flag.Parse()

	ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))
	watchNamespace, err := getWatchNamespace()
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:                 scheme,
		MetricsBindAddress:     metricsAddr,
		Port:                   9443,
		HealthProbeBindAddress: probeAddr,
		LeaderElection:         enableLeaderElection,
		LeaderElectionID:       "1ef59d40.linuxera.org",
		Namespace:              watchNamespace, // namespaced-scope when the value is not an empty string
	})
	if err != nil {
		setupLog.Error(err, "unable to start manager")
		os.Exit(1)
	}

	if err = (&controllers.ReverseWordsAppReconciler{
		Client: mgr.GetClient(),
		Scheme: mgr.GetScheme(),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "ReverseWordsApp")
		os.Exit(1)
	}
	//+kubebuilder:scaffold:builder

	if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up health check")
		os.Exit(1)
	}
	if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up ready check")
		os.Exit(1)
	}

	setupLog.Info("starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
}

// getWatchNamespace returns the Namespace the operator should be watching for changes
func getWatchNamespace() (string, error) {
	// WatchNamespaceEnvVar is the constant for env variable WATCH_NAMESPACE
	// which specifies the Namespace to watch.
	// An empty value means the operator is running with cluster scope.
	var watchNamespaceEnvVar = "WATCH_NAMESPACE"

	ns, found := os.LookupEnv(watchNamespaceEnvVar)
	if !found {
		return "", fmt.Errorf("%s must be set", watchNamespaceEnvVar)
	}
	return ns, nil
}
~~~

You can download the `main.go`, remember to change the GitHub ID before bulding the operator:

~~~sh
# Remember to change import: appsv1alpha1 "github.com/mvazquezc/reverse-words-operator/api/v1alpha1"

curl -Ls https://linuxera.org/writing-operators-using-operator-framework/main.go -o ~/operators-projects/reverse-words-operator/main.go
~~~

## Specify permissions and generate RBAC manifests

Our controller needs some RBAC permissions to interact with the resources it manages. These has been specified via RBAC Markers in our controller code:

~~~go
// +kubebuilder:rbac:groups=apps.linuxera.org,resources=reversewordsapps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps.linuxera.org,resources=reversewordsapps/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;

func (r *ReverseWordsAppReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
~~~

The ClusterRole manifest at config/rbac/role.yaml is generated from the above markers via controller-gen with the following command:

~~~sh
go mod tidy
make manifests
~~~

## Build the Operator

First, we will build the operator and once the image is built, we will push it to the [Quay Registry](https://quay.io/).

Before we start building the operator, we need access to a Kubernetes cluster. If you don't have one you can use [Kind](https://github.com/kubernetes-sigs/kind/), [Minikube](https://github.com/kubernetes/minikube) or my prefered one, [KCli](https://github.com/karmab/kcli)

In order to get a local cluster with KCli, just run this command:

~~~sh
kcli create kube generic -P masters=1 -P workers=1 -P master_memory=4096 -P numcpus=2 -P worker_memory=4096 -P sdn=calico -P version=1.18 -P ingress=true -P ingress_method=nginx -P metallb=true -P domain=linuxera.org operatorscluster
~~~

Now that we have the cluster up and running we will build and push the operator.

> **NOTE**: If you use podman instead of docker you can edit the Makefile and change docker commands by podman commands

~~~sh
export USERNAME=<quay-username>
make docker-build docker-push IMG=quay.io/$USERNAME/reversewords-operator:v0.0.1
~~~

## Deploy the Operator

1. Create the required CRDs in the cluster

    ~~~sh
    make install
    ~~~
2. Deploy the operator

    > **NOTE**: While developing you can run the operator locally (you need a valid kubeconfig) by running `make run`

    In order to deploy the different operator pieces, Kustomize is used. There is a Kustomization file (`~/operators-projects/reverse-words-operator/config/default/kustomization.yaml`) where you can define some defaults for your operator, like the namePrefix for the different objects or the namespace where it will be deployed. 

    1. Edit the default kustomization file `~/operators-projects/reverse-words-operator/config/default/kustomization.yaml` and specify the namespace where your operator should run by modifying the `namespace` property

        ~~~sh
        export NAMESPACE=operators-test
        sed -i "s/namespace: .*/namespace: $NAMESPACE/g" ~/operators-projects/reverse-words-operator/config/default/kustomization.yaml
        ~~~   
    
    2. Create the namespace and Deploy the operator

        ~~~sh
        kubectl create ns $NAMESPACE
        export USERNAME=<quay_username>
        make deploy IMG=quay.io/$USERNAME/reversewords-operator:v0.0.1
        ~~~
    3. Patch the controller deployment so it only watches the namespace where it's running
   
        ~~~sh
        kubectl -n $NAMESPACE patch deployment reverse-words-operator-controller-manager -p '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"kube-rbac-proxy"},{"name":"manager"}],"containers":[{"env":[{"name":"WATCH_NAMESPACE","valueFrom":{"fieldRef":{"fieldPath":"metadata.namespace"}}}],"name":"manager"}]}}}}'
        ~~~
6. We should see our operator pod up and running

    ~~~
	2021-12-20T19:19:06.102Z	INFO	controller-runtime.metrics	metrics server is starting to listen	{"addr": "127.0.0.1:8080"}
	2021-12-20T19:19:06.104Z	INFO	setup	starting manager
	I1220 19:19:06.109145       1 leaderelection.go:248] attempting to acquire leader lease operators-test/1ef59d40.linuxera.org...
	2021-12-20T19:19:06.110Z	INFO	starting metrics server	{"path": "/metrics"}
	I1220 19:19:37.027525       1 leaderelection.go:258] successfully acquired lease operators-test/1ef59d40.linuxera.org
	2021-12-20T19:19:37.027Z	DEBUG	events	Normal	{"object": {"kind":"ConfigMap","namespace":"operators-test","name":"1ef59d40.linuxera.org","uid":"04a51285-5f4f-4aef-a4f9-fc0794fbe32d","apiVersion":"v1","resourceVersion":"1687"}, "reason": "LeaderElection", "message": "reverse-words-operator-controller-manager-84f854f9-5w8wh_5bdcb9e5-8cf7-4129-8908-32734f3b56f6 became leader"}
	2021-12-20T19:19:37.028Z	DEBUG	events	Normal	{"object": {"kind":"Lease","namespace":"operators-test","name":"1ef59d40.linuxera.org","uid":"90ced595-977a-4208-9506-e219c7bfd31a","apiVersion":"coordination.k8s.io/v1","resourceVersion":"1688"}, "reason": "LeaderElection", "message": "reverse-words-operator-controller-manager-84f854f9-5w8wh_5bdcb9e5-8cf7-4129-8908-32734f3b56f6 became leader"}
	2021-12-20T19:19:37.028Z	INFO	controller.reversewordsapp	Starting EventSource	{"reconciler group": "apps.linuxera.org", "reconciler kind": "ReverseWordsApp", "source": "kind source: /, Kind="}
	2021-12-20T19:19:37.028Z	INFO	controller.reversewordsapp	Starting EventSource	{"reconciler group": "apps.linuxera.org", "reconciler kind": "ReverseWordsApp", "source": "kind source: /, Kind="}
	2021-12-20T19:19:37.028Z	INFO	controller.reversewordsapp	Starting EventSource	{"reconciler group": "apps.linuxera.org", "reconciler kind": "ReverseWordsApp", "source": "kind source: /, Kind="}
	2021-12-20T19:19:37.028Z	INFO	controller.reversewordsapp	Starting Controller	{"reconciler group": "apps.linuxera.org", "reconciler kind": "ReverseWordsApp"}
	2021-12-20T19:19:37.130Z	INFO	controller.reversewordsapp	Starting workers	{"reconciler group": "apps.linuxera.org", "reconciler kind": "ReverseWordsApp", "worker count": 1}
    ~~~
7. Now it's time to create ReverseWordsApp instances

    ~~~sh
    cat <<EOF | kubectl -n $NAMESPACE create -f -
    apiVersion: apps.linuxera.org/v1alpha1
    kind: ReverseWordsApp
    metadata:
        name: example-reversewordsapp
    spec:
        replicas: 1
    EOF

    cat <<EOF | kubectl -n $NAMESPACE create -f -
    apiVersion: apps.linuxera.org/v1alpha1
    kind: ReverseWordsApp
    metadata:
        name: example-reversewordsapp-2
    spec:
        replicas: 2
    EOF
    ~~~
8. We should see two deployments and services being created, and if wee look at the status of our object we should see the pods backing the instance

    ~~~sh
    kubectl -n $NAMESPACE get reversewordsapps example-reversewordsapp -o yaml

	apiVersion: apps.linuxera.org/v1alpha1
	kind: ReverseWordsApp
	metadata:
	  creationTimestamp: "2021-12-20T19:20:21Z"
	  finalizers:
	  - finalizer.reversewordsapp.apps.linuxera.org
	  generation: 1
	  name: example-reversewordsapp
	  namespace: operators-test
	  resourceVersion: "2020"
	  selfLink: /apis/apps.linuxera.org/v1alpha1/namespaces/operators-test/reversewordsapps/example-reversewordsapp
	  uid: 8f4c696a-b6b4-41f7-b4b2-647d1f8a55b6
	spec:
	  replicas: 1
	status:
	  appPods:
	  - dp-example-reversewordsapp-5786d986c5-rvgf7
	  conditions:
	  - lastTransitionTime: "2021-12-20T19:20:46Z"
	    message: ""
	    reason: ReverseWordsDeploymentNotReady
	    status: "False"
	    type: ReverseWordsDeploymentNotReady
	  - lastTransitionTime: "2021-12-20T19:20:46Z"
	    message: ""
	    reason: Ready
	    status: "True"
	    type: Ready
    ~~~

9. We can test our application now

    ~~~sh
    LB_ENDPOINT=$(kubectl -n $NAMESPACE get svc --selector='app=example-reversewordsapp' -o jsonpath='{.items[*].status.loadBalancer.ingress[*].ip}')
    
    curl -X POST -d '{"word":"PALC"}' http://$LB_ENDPOINT:8080
    {"reverse_word":"CLAP"}
    ~~~
10. Cleanup

    ~~~sh
    kubectl -n $NAMESPACE delete reversewordsapp example-reversewordsapp example-reversewordsapp-2
    kubectl delete -f config/crd/bases/apps.linuxera.org_reversewordsapps.yaml 
    kubectl delete ns operators-test
    ~~~
11. That's it!

# In the next episode:

* [We will look at how to use OLM to release our operator](https://linuxera.org/integrating-operators-olm/)
* We will see a K8s controllers deep dive

# Sources

* [Operators by CoreOS](https://coreos.com/operators/)
* [A deep dive into Kubernetes Controllers](https://engineering.bitnami.com/articles/a-deep-dive-into-kubernetes-controllers.html)
* [Writing Kube Controllers for Everyone](https://www.youtube.com/watch?v=AUNPLQVxvmw)
