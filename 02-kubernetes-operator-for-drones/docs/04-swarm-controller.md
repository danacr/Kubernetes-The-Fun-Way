# Swarm Controller

Since our controller knows what a drone is, the only way to start them now is by manually requesting drone, but what if we could use an aggregator of some sorts?

This is why I chose to create the Swarm resource and controller:

```go
// SwarmSpec defines the desired state of Swarm
type SwarmSpec struct {
	HowMany *int32 `json:"howmany,omitempty"`
}

// SwarmStatus defines the observed state of Swarm
type SwarmStatus struct {
	FlyingDrones int32 `json:"flyingdrones,omitempty"`
}
```

> Note: The Swarm Custom Resource allows us to request a number of drones to be launched and to observe that state using the Status field

The swarm controller simply checks for the current number of drones and launches new ones or lands existing ones.

```go
func (r *SwarmReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	ctx := context.Background()
	log := r.Log.WithValues("Swarm", req.NamespacedName)

	// your logic here
	log.Info("fetching swarm resource")
	swarm := experimentsv1.Swarm{}
	if err := r.Client.Get(ctx, req.NamespacedName, &swarm); err != nil {
		log.Error(err, "failed to get swarm")
		// Ignore NotFound errors as they will be retried automatically if the
		// resource is created in future.
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	log.Info("Do we have enough drones?")

	drones := experimentsv1.DroneList{}
	if err := r.List(ctx, &drones); err != nil {
		return ctrl.Result{}, err
	}

	if int32(len(drones.Items)) < *swarm.Spec.HowMany {
		log.Info("Not enough, must create drones")

		name := strings.ReplaceAll(namesgenerator.GetRandomName(0), "_", "-")

		drone := experimentsv1.Drone{
			ObjectMeta: metav1.ObjectMeta{
				Name:      name,
				Namespace: req.Namespace,
			},
		}
		if err := r.Client.Create(ctx, &drone); err != nil {
			log.Error(err, "failed to create drone")
			return ctrl.Result{}, err
		}

	}
	if int32(len(drones.Items)) > *swarm.Spec.HowMany {
		log.Info("Too many, must kill")
		r.Delete(ctx, &experimentsv1.Drone{
			ObjectMeta: ctrl.ObjectMeta{
				Name:      drones.Items[0].Name,
				Namespace: req.Namespace,
			},
		})
	}

	log.Info("updating swarm status")
	if err := r.List(ctx, &drones); err != nil {
		return ctrl.Result{}, err
	}
	swarm.Status.FlyingDrones = int32(len(drones.Items))
	err := r.Update(ctx, &swarm)
	if err != nil {
		log.Error(err, "failed to update swarm status")
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}
```

> Note: There appears to be a bug at the moment which causes drones to refuse flying on the first try, so the drone-pod container has to be rescheduled in order to start the drone.

Once this operator is deployed on the cluster, you can request Drones from Kubernetes the same way you would request pods :)

## Demo:

[![Kubernetes Operators](https://img.youtube.com/vi/JPVgxnsvOs0/maxresdefault.jpg)](https://www.youtube.com/watch?v=JPVgxnsvOs0)
