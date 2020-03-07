# Chaos Engineering on Kubernetes

#### Google Cloud Meetup Zürich [recording](https://www.youtube.com/watch?v=5JLOkjbkNg4) and [slides](https://docs.google.com/presentation/d/1VVZ1QPbae4Pnqr-sKEO4knL-2mwz-ijWt-AhJGu6EYQ/edit?usp=sharing)

The purpose is to test the resiliance of applications running in ephemeral environments, such as GKE on preemptible instances.

![Chaos](images/chaos.gif)

> Please note, do not run this in production

## Target Audience

The target audience for episode #3 is someone who is familiar with Kubernetes. React and Go knowledge is useful for the code samples.

## Cluster Details

- [GKE with a Preemptible Node Pool](https://cloud.google.com/kubernetes-engine/docs/how-to/preemptible-vms)

## IaaS

- [Google CloudSQL for Postgres](https://cloud.google.com/sql/docs/postgres) to store the state of the application
- [Google App Engine](https://cloud.google.com/appengine) to host the [disrupter server](k8s-disrupter/README.md)

## Steps

- [Mobile App](docs/01-mobile-app.md)
- [Disrupter Server](docs/02-disrupter-server.md)
- [Survivor](docs/03-survivor.md)
