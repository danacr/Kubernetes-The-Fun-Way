# KaaS Server 🧀

## Ephemeral Kubernetes as a Service using Typhoon K8s on Digital Ocean

> [Original repository](https://github.com/danacr/kaas), I also added it as a [submodule](../kaas/README.md), to this repository.

Example request for a development cluster:

```sh
key=$(base64 < key.asc)
json="{\"version\":\"v1.18.1\",\"pubkey\":\"$key\"}"

curl --header "Content-Type: application/json" \
  --request POST \
  --data "$json" \
  https://k8stfw.com/cluster
```

> Note: `key.asc` is your gpg public key, please see the [key.asc](../kaas/test/key.asc) for an example.

In order to run your own KaaS, one must install the appropriate yamls from the `k8s` folder and provide the Kubernetes config file (including the `do_token`) from his manager cluster to the KaaS server.

`KaaS` is a simple http request handler:

```go
func main() {

	http.HandleFunc("/cluster", kaas)

	http.HandleFunc("/favicon.ico", faviconHandler)

	fs := http.FileServer(http.Dir("./static"))
	http.Handle("/", fs)

	log.Printf("Starting kaas 🧀\n")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal(err)
	}

}
```

It checks the currently supported version by [Typhoon K8s](https://github.com/poseidon/typhoon):

```go
func checkversions() (SupportedVersions, error) {
	client := github.NewClient(nil)

	taglist, _, err := client.Repositories.ListTags(context.Background(), "poseidon", "typhoon", nil)
	if err != nil {
		return SupportedVersions{}, err
	}
	var supported SupportedVersions
	for _, tag := range taglist {
		supported.Versions = append(supported.Versions, tag.GetName())
	}
	if err != nil {
		return SupportedVersions{}, err
	}
	return supported, nil
}
```

If it receives a POST request with a valid version, it returns the url where the encrypted kubeconfig file will be hosted and triggers `createcluster()`:

```go
if stringInSlice(cluster.Version, supported.Versions) {
    id, err := uuid.NewUUID()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    cluster.ID = id.String()
    cluster.Minutes = "120"
    cluster.Region = "nyc3"
    cluster.Cfg = "https://storage.googleapis.com/" + cluster.ID + "/cluster-config.gpg"

    err = createcluster(cluster)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    js, err := json.Marshal(cluster.Cfg)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    _, err = w.Write(js)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

} else {
    fmt.Fprintf(w, "Unsupported version")
}
```

`createcluster()` creates a Kubernetes job using [Simple Typhoon K8s](https://github.com/danacr/simple-typhoon-k8s):

```go
job := &batchv1.Job{
    ObjectMeta: metav1.ObjectMeta{
        Name: cluster.ID,
    },
    Spec: batchv1.JobSpec{
        Template: apiv1.PodTemplateSpec{
            Spec: apiv1.PodSpec{
                Containers: []apiv1.Container{
                    {
                        Name:  "terraform",
                        Image: "danacr/stk:latest",
                        Env: []apiv1.EnvVar{
                            apiv1.EnvVar{
                                Name:  "TF_VAR_cluster_id",
                                Value: cluster.ID},
                            apiv1.EnvVar{
                                Name:  "TF_VAR_cluster_region",
                                Value: cluster.Region},
                            apiv1.EnvVar{
                                Name:  "HOW_LONG",
                                Value: cluster.Minutes},
                            apiv1.EnvVar{
                                Name:  "CLUSTER_VERSION",
                                Value: cluster.Version},
                            apiv1.EnvVar{
                                Name:  "pubkey",
                                Value: cluster.PubKey},
                        },
                        VolumeMounts: []apiv1.VolumeMount{
                            apiv1.VolumeMount{
                                Name:      "do-token",
                                MountPath: "/home/terraform/config/do_token",
                                SubPath:   "do_token",
                            },
                            apiv1.VolumeMount{
                                Name:      "service-account",
                                MountPath: "/home/terraform/config/service-account-key.json",
                                SubPath:   "service-account-key.json",
                            },
                        },
                    },
                },
                RestartPolicy: apiv1.RestartPolicyNever,
                Volumes: []apiv1.Volume{
                    apiv1.Volume{
                        Name: "do-token",
                        VolumeSource: apiv1.VolumeSource{
                            Secret: &apiv1.SecretVolumeSource{
                                SecretName: "do-token",
                                Items: []apiv1.KeyToPath{
                                    apiv1.KeyToPath{
                                        Key:  "do_token",
                                        Path: "do_token",
                                    },
                                },
                            },
                        },
                    },
                    apiv1.Volume{
                        Name: "service-account",
                        VolumeSource: apiv1.VolumeSource{
                            Secret: &apiv1.SecretVolumeSource{
                                SecretName: "service-account",
                                Items: []apiv1.KeyToPath{
                                    apiv1.KeyToPath{
                                        Key:  "service-account-key.json",
                                        Path: "service-account-key.json",
                                    },
                                },
                            },
                        },
                    },
                },
            },
        },
    },
}
```
