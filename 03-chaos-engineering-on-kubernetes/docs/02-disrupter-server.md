# K8s Disrupter Server

## Triggering Virtual Machine destruction using Go

> [Original repository](https://github.com/danacr/k8s-disrupter-server), I also added it as a submodule, to this repository.

We need to create a small program that will listen to events and trigger disruptions when the right POST request is received.

```go
func main() {
	var err error
	if err = serviceAccount(); err != nil {
		log.Fatal(err)
	}
	http.HandleFunc("/", disrupt)
	http.HandleFunc("/favicon.ico", faviconHandler)
	fmt.Printf("Starting Disrupter\n")
	if err = http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal(err)
	}
}
```

> Note: you need to provide a service account which has the right permissions on GCP

Then we need to create a list of instances that correspond to our cluster instance group. The "google.golang.org/api/compute/v1" package is great for that

```go
func getinstances() ([]*compute.InstanceWithNamedPorts, error) {
	ctx := context.Background()

	c, err := google.DefaultClient(ctx, compute.CloudPlatformScope)
	if err != nil {
		return nil, err
	}

	computeService, err := compute.New(c)
	if err != nil {
		return nil, err
	}

	// Project ID for this request.
	project := "kill-the-cluster"

	// The name of the zone where the instance group is located.
	zone := "europe-west6-a" //

	// The name of the instance group from which you want to generate a list of included instances.
	instanceGroup := "gke-chaotic-cluster-preemptible-pool-ae9c7dfb-grp"

	rb := &compute.InstanceGroupsListInstancesRequest{}
	var instancelist []*compute.InstanceWithNamedPorts
	req := computeService.InstanceGroups.ListInstances(project, zone, instanceGroup, rb)
	if err := req.Pages(ctx, func(page *compute.InstanceGroupsListInstances) error {
		instancelist = append(instancelist, page.Items...)
		return nil
	}); err != nil {
		return nil, err
	}
	return instancelist, nil
}
```

Using https://cloud.google.com/compute/docs/reference/rest/v1/instances/delete we can delete the instance using its name

```go
func deleteinstance(instance string) (string, error) {
	ctx := context.Background()

	c, err := google.DefaultClient(ctx, compute.CloudPlatformScope)
	if err != nil {
		log.Fatal(err)
	}

	computeService, err := compute.New(c)
	if err != nil {
		return "", err
	}

	// Project ID for this request.
	project := "kill-the-cluster"

	// The name of the zone where the instance group is located.
	zone := "europe-west6-a" //

	resp, err := computeService.Instances.Delete(project, zone, instance).Context(ctx).Do()
	if err != nil {
		return "", err
	}

	// TODO: Change code below to process the `resp` object:
	result := fmt.Sprintf("%#v\n", resp)
	return result, nil
}
```

Wen we receive the post request from the [K8s Disrupter](../k8s-disrupter), select one of the running instances and delete it:

```go
	case "POST":
		phone := Device{}
		err := json.NewDecoder(r.Body).Decode(&phone)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		instancelist, err := getinstances()
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		var goodbye string
		for _, instanceWithNamedPorts := range instancelist {
			if instanceWithNamedPorts.Status == "RUNNING" {
				goodbye = instanceWithNamedPorts.Instance
			}
		}
		goodbye = goodbye[strings.LastIndex(goodbye, "/")+1:]

		result, err := deleteinstance(goodbye)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		log.Println(result)

		http.StatusText(200)
```

Next: [Survivor](03-survivor.md)
