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

Regardless where you will be running your apps - Cloud Foundry, Kubernetes, Open Shift, etc - you will deploy this magnificent system using BOSH, because BOSH is super awesome at managing infrastructure over the many decades you'll want to keep your stateful services running. You might also want to use BOSH to deploy your app platform (see https://github.com/cloudfoundry-incubator/kubo-deployment for Kubernetes).

* What is BOSH? http://bosh.io/docs
* What is the Open Service Broker API? ([announcement blog post](https://www.openservicebrokerapi.org/blog/2016/12/13/why-cloud-foundry-is-making-the-open-service-broker-api-even-more-open))
* Bootstrapping your BOSH director https://github.com/cloudfoundry/bosh-deployment
* [optional] Bootstrapping Cloud Foundry https://github.com/cloudfoundry/cf-deployment
* Bootstrapping Docker Services (this project) https://github.com/cloudfoundry-community/docker-broker-deployment

## Preparation

Target your BOSH using environment variable:

```
export BOSH_ENVIRONMENT=<alias or IP>
export BOSH_DEPLOYMENT=docker-broker
```

The example above was created using the following deployment:

```
git clone https://github.com/cloudfoundry-community/docker-broker-deployment
bosh deploy docker-broker-deployment/docker-broker.yml \
  -o docker-broker-deployment/services/op-postgresql96.yml \
  -o docker-broker-deployment/services/op-mysql56.yml \
  -o docker-broker-deployment/services/op-redis32.yml \
  -o <(./docker-broker-deployment/pick-from-cloud-config.sh)
```

NOTE: remember to add `--vars-store` if your BOSH environment does not have credhub/config-server.

That's it! BOSH will bring up a cluster of servers that each run the `docker` daemon. They also each have a local agent [`cf-containers-broker`](https://github.com/cloudfoundry-community/cf-containers-broker/), and will start downloading the three Docker images corresponding to the three `services/*.yml` files included above. The deployment also includes the coordinating service broker `subway`.

To see the list of servers running, use `bosh instances`:

```
Instance                                          Process State  AZ  IPs
docker/0cc28eb1-0fd1-4929-a529-b90aed2933e2       running        z1  10.244.0.151
docker/14c56cc6-05dc-4eba-b1ee-776fb6e345d1       running        z3  10.244.0.157
docker/b3f2a25d-adf6-410b-8057-160d723fbd65       running        z2  10.244.0.156
sanity-test/7299137a-89c2-4c3e-bf76-d33255171d57  -              z1  10.244.0.158
subway/f1c65dd9-7ab6-4192-98cc-749711cd0ce5       running        z1  10.244.0.155
```

Once deployed, you can dynamically provision new Docker containers using the Service Broker API.

To confirm that each included service - `postgresql96`, `redis32` and `mysql56` - is working, run the `sanity-test` errand:

```
bosh run-errand sanity-test
```

The `subway` instance is the Service Broker API for provisioning/binding/unbinding/unprovisioning, and is running on port `8000`.

### Integration with Cloud Foundry

If you're an administrator for Cloud Foundry, its nice and easy to update your `docker-broker` to register itself and make your services available to all users.

Add `-o op-cf-integration.yml` to the command you ran above to redeploy the `docker-broker`, and the four `cf_*` variables to describe the admin credentials for the Cloud Foundry (which should be deployed by the same BOSH with the name `cf`):

```
export BOSH_DEPLOYMENT=docker-broker

# name of cf deployment in BOSH, e.g. 'cf'
cf_deployment=cf

system_domain=$(bosh -d $cf_deployment manifest | bosh int - --path /instance_groups/name=api/jobs/name=cloud_controller_ng/properties/system_domain)
skip_verify=$(bosh -d $cf_deployment manifest | bosh int - --path /instance_groups/name=api/jobs/name=cloud_controller_ng/properties/ssl/skip_cert_verify)
admin_password=$(bosh -d $cf_deployment manifest | bosh int - --path /instance_groups/name=uaa/jobs/name=uaa/properties/uaa/scim/users/name=admin/password)

bosh deploy docker-broker-deployment/docker-broker.yml \
  -o docker-broker-deployment/op-cf-integration.yml \
  -o docker-broker-deployment/services/op-postgresql96.yml \
  -o docker-broker-deployment/services/op-mysql56.yml \
  -o docker-broker-deployment/services/op-redis32.yml \
  -v cf-api-url=https://api.$system_domain \
  -v cf-skip-ssl-validation=$skip_verify \
  -v cf-admin-username=admin \
  -v "cf-admin-password=$admin_password" \
  -v broker-route-name=docker-broker \
  -v broker-route-uri=docker-broker.$system_domain \
  -o <(./docker-broker-deployment/pick-from-cloud-config.sh -o op-cf-integration.yml)
```

Update the `-v` variables for your Cloud Foundry and system domain.

Again to confirm that the API is available via the new route and the system is still working, run the `sanity-test` errand again:

```
bosh run-errand sanity-test
```

Next, run the `bosh run-errand broker-registrar` one-off errand to register the service broker with your Cloud Foundry (using the admin credentials included in `-v` variables above):

```
bosh run-errand broker-registrar
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

### Single VM

You can have a single vm deployment using the `docker-broker-solo.yml` manifest:

```
bosh deploy docker-broker-deployment/docker-broker-solo.yml \
  -o docker-broker-deployment/services/op-redis32.yml \
  -o <(./docker-broker-deployment/pick-from-cloud-config.sh docker-broker-solo.yml)
```

The other operator files might not yet work with this single instance deployment.
