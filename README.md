PFS Earthquake Demo
===

Install PFS and configure outbound network access
---
Make sure to refer to the PFS documentation for instructions on how to install PFS on your Kubernetes cluster. 

For this demo, we'll also rely on a couple of services outside of our network, so we'll need to [configure outbound network access](https://github.com/knative/docs/blob/master/serving/outbound-network-access.md) for our services. We also need to enable our custom event source to access the [USGS Earthquake Data Feed](https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_hour.geojson) which requires a slightly different approach. This can all be defined via YAML which is included in this repository, so you can apply this configuration with:

```
kubectl apply -f usgs-egress.yaml
```

Set Up Postgres with Helm
---
Our application will rely on a Postgres database to store events as they come in. Luckily, we can easily set this up locally using [Helm](https://helm.sh/). While this won't configure something we'd use in production, it will serve as a great solution for our demo. We'll assume you have the helm CLI installed, but if not refer to Helm's documentation on how to get it.

First we'll set up a service account in our Kubernetes cluster for Helm to use, and then initalize Helm.

```
kubectl create serviceaccount --namespace kube-system tiller

kubectl create clusterrolebinding tiller \
    --clusterrole cluster-admin \
    --serviceaccount=kube-system:tiller

helm init --service-account tiller
```

Next, we can install Postgres and give it a few arguments. Here, we'll set the password to the "postgres" user account to "devPass" and then create a database called "geocode"

```
helm install --name geocodedb --set postgresqlPassword=devPass,postgresqlDatabase=geocode stable/postgresql
```

Once it's up and running this database will be available internally to our Kubernetes cluster at `geocodedb-postgresql.default.svc.cluster.local`. If you'd like to access it locally, you can forward a local port to your Kubernetes cluster and then connect to it like it was running on your machine.

```
kubectl port-forward --namespace default svc/geocodedb-postgresql 5432:5432

PGPASSWORD="devPass" psql --host 127.0.0.1 -U postgres
```

Deploy Geocode Service
---

Our Geocode service is responsible for taking in the earthquake events and reverse geocode the coordinates to find the closes street address to make it easier for users to read. Additionally, all events will be written to our Postgres database to be pulled by our event service when requested by the frontend.