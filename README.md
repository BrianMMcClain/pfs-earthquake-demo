PFS Earthquake Demo
===

Install PFS and configure outbound network access
---
Make sure to refer to the [PFS documentation](https://docs.pivotal.io/pfs/) for instructions on how to install PFS on your Kubernetes cluster. 

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

Get a Google Maps API Key
---

Our Geocode service relies on the Google Maps API to perform the actual Geocoding, so we'll need an API key. Google allows up to 25,000 requests per day, which is well below the volume we'll be running, so there's no need to pay for this API.

The [Google Maps Services Java Lubrary](https://github.com/googlemaps/google-maps-services-java#api-keys) walks through how to generate a new key, but in short:

Each Google Maps Web Service request requires an API key or client ID. API keys are freely available with a Google Account at [developers.google.com/console](https://developers.google.com/console). The
type of API key you need is a **Server key**.

To get an API key:

 1. Visit [developers.google.com/console](https://developers.google.com/console)
  and log in with a Google Account.
 1. Select one of your existing projects, or create a new project.
 1. Enable the API(s) you want to use. The Java Client for Google Maps Services
    accesses the following APIs:
    - Directions API
    - Distance Matrix API
    - Elevation API
    - Geocoding API
    - Maps Static API
    - Places API
    - Roads API
    - Time Zone API
 1. Create a new **Server key**.
 1. If you'd like to restrict requests to a specific IP address, do so now

Prepare to Deploy Services
---

Since the Geocode service and the Events service were written with PFS in mind, there won't be any need for us to clone and build the code locally. Instead, PFS will take care of it for us and then upload the container images to our container registry. We'll just need to set a couple of environment varalbes to make the deployment a little easier. One for our container registry location and one for our registry username. For example, since I uploaded my images to Google's Container Registry, I ran:

```
export REGISTRY=gcr.io
export REGISTRY_USER=$(gcloud config get-value core/project)
```

Make sure to refer to the PFS documentation if you'd like to use a different container registry.

Prepare to Deploy the Event Source
---

We'll be deploying an custom event source soon, and while PFS ships with a few event sources, the ContainerSource is relativly new, so we'll pull in the open source event sources for our demo.

```
kubectl apply --filename https://github.com/knative/eventing-sources/releases/download/v0.2.1/release.yaml
```

Deploy Geocode Service
---

Our [Geocode service](https://github.com/BrianMMcClain/geocoder-pfs) is responsible for taking in the earthquake events and reverse geocode the coordinates to find the closes street address to make it easier for users to read. Additionally, all events will be written to our Postgres database to be pulled by our event service when requested by the frontend.

As we mentioned, PFS will pull and build the code for us, so we don't need to do so locally. Instead, we can run one command to tell PFS everything it needs to know to run our service, filling in your Google API Key:

```
pfs function create geocoder --git-repo https://github.com/BrianMMcClain/geocoder-pfs.git --image $REGISTRY/$REGISTRY_USER/geocoder --verbose --env "PGHOST=geocodedb-postgresql.default.svc.cluster.local" --env "PGPORT=5432" --env "PGDATABASE=geocode" --env "PGUSER=postgres" --env "PGPASSWORD=devPass" --env "GOOGLE_API_KEY=<GOOGLE API KEY>"
```

There's a few arguments here that are worh highlighting:

- **--git-repo** - Path to the git repoistory that has the code for our function
- **--image** - Where PFS should upload the resulting container image
- **--verbose** - Tail the logs and wait for the function to deploy before returning. Not required, but will give us good insight into the process
- **--env** - Environment variables to set for our function in the format "key=value". The PFS CLI can take multiple of these arguments and all will be applied

A great way to track the progress of these deploys is to watch the kubernetes pods that are spun up in the default namespace in another terminal, which can be done with:

```
kubectl get pods -w
```

You'll see a pod spin up with a good handful of containers (usually something in the realm of 8), which is the pod that will build our code, build the container and push it to our registry. After that, the build pod will spin down and we'll see our function spin up.

Deploy Event Source
---

We've also built a custom [event source](https://github.com/knative/docs/tree/master/eventing#sources) that polls the USGS Earthquake data, impletmeneted as a [ContainerSource](https://github.com/knative/docs/tree/master/eventing#containersource). When using a container source, we provide PFS the container that're responsible for the logic of our event source, and in turn PFS provides a URL to send events.

The event source will take a prebuilt container image which we've already built and push up. The configurarion also takes a "sink", which is a reference to a Kubernetes object to send events to. In PFS, we can configure a channel which provides a lot of benefits such as a higher guerentee of delivery, but to keep things simple we've configured our event source to send events directly to our Geocoder service. All of this is defined in the [usgs-event-source.yaml](usgs-event-source.yaml) file, which you can apply directly to your environment:

```
kubectl apply -f usgs-event-source.yaml
```