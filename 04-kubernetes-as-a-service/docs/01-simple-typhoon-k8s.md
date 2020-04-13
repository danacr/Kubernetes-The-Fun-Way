# Simple Typhoon K8s

Tiny bash setup script that creates Typhoon K8s clusters

> [Original repository](https://github.com/danacr/simple-typhoon-k8s), I also added it as a submodule, to this repository.

`create.sh` starts by generating and adding an ssh identity to your digital ocean account:

```sh
eval `ssh-agent` # create the process

# Get ssh fingerprint
ssh-keygen -f /home/terraform/.ssh/id_rsa -q -N ""
ssh-add /home/terraform/.ssh/id_rsa
ssh-add -L

export TF_VAR_ssh_fingerprint=$(ssh-add -l -E md5| awk '{print $2}'|cut -d':' -f 2-)
export TF_VAR_do_token=$(< config/do_token)

# Add ssh pub key to do
export auth="Authorization: Bearer "$TF_VAR_do_token
export payload="{\"name\":\"$TF_VAR_cluster_id\",\"public_key\":\"$(cat /home/terraform/.ssh/id_rsa.pub)\"}"
curl -X POST -H "Content-Type: application/json" -H "$auth" -d "$payload" "https://api.digitalocean.com/v2/account/keys"
```

`stk` is the go binary that manages interactions with the Google Cloud Platform API, such as creating a GCS bucket:

```go
func creategcs() error {
	if err := serviceAccount(); err != nil {
		return err
	}
	ctx := context.Background()
	client, err := storage.NewClient(ctx)
	if err != nil {
		return err
	}
	bucket := client.Bucket(os.Getenv("TF_VAR_cluster_id"))

	ctx, cancel := context.WithTimeout(ctx, time.Second*10)
	defer cancel()
	if err := bucket.Create(ctx, "k8stfw", &storage.BucketAttrs{
		StorageClass: "STANDARD",
		Location:     "us-east1",
	}); err != nil {
		return err
	}
	return nil
}
```

`create.sh` then modifies the `main.tf` file and starts the cluster provisioning:

```sh
envsubst < main.tf.bak | tee main.tf

terraform init
terraform plan -out create.plan
terraform apply -auto-approve create.plan
```

Once the cluster is created, `stk encrypt` will gpg encrypt the generated kubeconfig:

```go
func cfgencrypt() error {
	// Decode public key
	decoded, err := base64.StdEncoding.DecodeString(os.Getenv("pubkey"))
	if err != nil {
		return err
	}
	// Read in public key
	recipient, err := readEntity(decoded)
	if err != nil {
		return err
	}

	f, err := os.Open("cluster-config")
	if err != nil {
		return err
	}
	defer f.Close()

	dst, err := os.Create("cluster-config.gpg")
	if err != nil {
		return err
	}
	defer dst.Close()
	err = encrypt([]*openpgp.Entity{recipient}, nil, f, dst)
	if err != nil {
		return err
	}
	return nil
}
```

Then upload it to the GCS bucket it created earlier and set the file ACL to public:

```go
func uploadcfg() error {
	if err := cfgencrypt(); err != nil {
		return err
	}
	if err := serviceAccount(); err != nil {
		return err
	}
	ctx := context.Background()
	client, err := storage.NewClient(ctx)
	if err != nil {
		return err
	}
	bucket := client.Bucket(os.Getenv("TF_VAR_cluster_id"))

	ctx, cancel := context.WithTimeout(ctx, time.Second*50)
	defer cancel()

	f, err := os.Open("cluster-config.gpg")
	if err != nil {
		return err
	}
	defer f.Close()

	wc := bucket.Object("cluster-config.gpg").NewWriter(ctx)
	if _, err = io.Copy(wc, f); err != nil {
		return err
	}
	if err := wc.Close(); err != nil {
		return err
	}
	acl := bucket.Object("cluster-config.gpg").ACL()
	if err := acl.Set(ctx, storage.AllUsers, storage.RoleReader); err != nil {
		return err
	}

	return nil
}
```

The script will then wait for as long as the cluster needs to be alive, after which it will delete the cluster resources:

```sh
if [ ! -z "$HOW_LONG" ]
then
    sleep "$HOW_LONG"m
    terraform destroy -auto-approve
    stk delete
fi
```

`stk delete` will remove all the files in the GCS bucket and then delete the bucket itself:

```go
func deletegcs(bucket *storage.BucketHandle, ctx context.Context) error {

	ctx, cancel := context.WithTimeout(ctx, time.Second*10)
	defer cancel()
	it := bucket.Objects(ctx, nil)

	for {
		attrs, err := it.Next()
		if err == iterator.Done {
			break
		}
		if err != nil {
			return err
		}
		err = bucket.Object(attrs.Name).Delete(ctx)
		if err != nil {
			return err
		}

	}
	if err := bucket.Delete(ctx); err != nil {
		return err
	}
	return nil
}
```

Next: [KaaS Server](02-kaas-server.md)
