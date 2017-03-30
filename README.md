# Docker Broker Deployment

If you're going to deploy 100s of applications then you'll need 100s of databases, message buses and more. This project deploys an [Open Service Broker](https://www.openservicebrokerapi.org/) API-compatible system that can provision dozens of different stateful services: each running inside an isolated Docker container, and each backed by isolated persistent disk from the host machine. And because its all deployed with BOSH, you will have all the tools to deploy everything today and to keep it upgraded, scaled, and security patched over the coming years and decades.

For example, if you're using Cloud Foundry, all your users will have instant access to any number of services that you'd like to offer:

```
$ cf marketplace

service        plans   description
mysql56        free    MySQL 5.6 service for application development and testing
postgresql96   free    PostgreSQL 9.6 service for application development and testing
redis32        free    Redis 3.2 service for application development and testing
```

And here's an animated GIF of provisioning a service, and `ctop` (running at the top) showing the new container coming into life:

![ctop](cf-create-service-ctop.gif)

I haven't yet tried integrating it with Kubernetes via https://github.com/kubernetes-incubator/service-catalog - but I'm sure it works and I'm sure its fabulous. I will look into this soon. In the meantime, here's [@pmorie](https://github.com/pmorie) [playing around with the K8s Service Broker](https://www.youtube.com/watch?v=tRAv5PozgNE). Technically I should demonstrate this service broker in this README... but Paul has awesome hair in the video.

This all sounds great, right. Excellent! Let's boot this thing up!

## Summary of deployment proceedings

This project is everything you need to deploy an Open Service Broker - useful with [Cloud Foundry](http://docs.cloudfoundry.org/services/index.html), and other orchestration systems over time (please let me know when its available for your favourite system!)

You can also interact directly with the API to provision and deprovision stateful services.

Regardless where you will be running your apps - Cloud Foundry, Kubernetes, Open Shift, etc - you will deploy this magnificent system using BOSH, because BOSH is super awesome at managing infrastructure over the many decades you'll want to keep your stateful services running. You might also want to use BOSH to deploy your app platform (see https://github.com/pivotal-cf-experimental/kubo-deployment for Kubernetes).

* What is BOSH? http://bosh.io/docs
* What is the Open Service Broker API? ([announcement blog post](https://www.openservicebrokerapi.org/blog/2016/12/13/why-cloud-foundry-is-making-the-open-service-broker-api-even-more-open))
* Bootstrapping your BOSH director https://github.com/cloudfoundry/bosh-deployment
* [optional] Bootstrapping Cloud Foundry https://github.com/cloudfoundry/cf-deployment
* Bootstrapping Docker Services (this project) https://github.com/cloudfoundry-community/docker-services-deployment

## Preparation

Your BOSH needs to have a cloud-config installed with a `default` option for both `networks` & `vm_types`. See `manifests/boshlite-cloud-config.yml`.

Target your BOSH using environment variable:

```
export BOSH_ENVIRONMENT=<alias or IP>
export BOSH_DEPLOYMENT=docker-broker
```

The example above was created using the following deployment:

```
bosh2 deploy docker-broker.yml \
  --vars-store tmp/creds.yml \
  -o services/op-postgresql96.yml \
  -o services/op-mysql56.yml \
  -o services/op-redis32.yml
```

That's it! BOSH will bring up a VM that will run `docker` daemon and the local service broker [`cf-containers-broker`](https://github.com/cloudfoundry-community/cf-containers-broker/), and will start downloading the three Docker images corresponding to the three `services/*.yml` files included above.

Once deployed, you can dynamically provision new Docker containers using the Service Broker API.

### Integration with Cloud Foundry

If you're an administrator for Cloud Foundry, its nice and easy to update your `docker-broker` to register itself and make your services available to all users.

Add `-o op-cf-integration.yml` to the command you ran above to redeploy the `docker-broker`, and the four `cf_*` variables to describe the admin credentials for the Cloud Foundry (which should be deployed by the same BOSH with the name `cf`):

```
bosh2 deploy docker-broker.yml --vars-store tmp/creds.yml \
  -o op-cf-integration.yml \
  -o services/op-postgresql96.yml \
  -o services/op-mysql56.yml \
  -o services/op-redis32.yml \
  -v cf-api-url=<https://api.mycf.com> \
  -v cf-skip-ssl-validation=false
  -v cf-admin-username=admin \
  -v cf-admin-password=password
```

Then run `bosh2 run-errand broker-registrar` one-off task:

```
bosh2 run-errand broker-registrar
```

The results will include output that finishes with:

```
Enabling access to all plans of service postgresql96 for all orgs as admin...
OK
Enabling access to all plans of service mysql56 for all orgs as admin...
OK
Enabling access to all plans of service redis32 for all orgs as admin...
OK
```

Each of your services will be now available in the service catalog/marketplace (as demonstrated above):

```
cf marketplace
```

You an now provision each of the services, for example:

```
cf create-service redis32 free redis1
```

Either you can bind `redis1` to an application; or you can request service keys to see the credentials yourself:

```
cf create-service-key redis1 redis1-key
cf service-key redis1 redis1-key
```

For a redis service the output might look like below. For other services it will differ.

```yaml
{
 "hostname": "10.244.0.144",
 "password": "yu8qixgw15bmdum1",
 "port": "32768",
 "ports": {
  "6379/tcp": "32768"
 }
}
```

Your ability to directly connect to the service will be determined by networking.
