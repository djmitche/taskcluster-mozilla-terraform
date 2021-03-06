# Taskcluster Mozilla Terraform

This is the Taskcluster team's internal terraform configuration for setting up
the team's clusters. It uses [taskcluster-terraform](https://github.com/taskcluster/taskcluster-terraform) to do most of the work. This is a good project to cargo-cult if you
wish to set up your own cluster.

## Prerequisites

To run terraform, you will need:

* Docker
* Configured passwordstore access to team secrets
* AWS credentials
* Azure credentials
* A Google Cloud account

## Usage

First, don't forget to update submodules.
In the root of this repo, run:

```shell
git submodule init
git submodule update
```

You'll need to re-run the `update` part when git indicates it's out of date.

Because of the peculiar configuration of terraform used here, the supported way to apply the configuration in the repository is to run terraform in the provided docker image.
To do so, run

```shell
./terraform-runner.sh <deployment>
```

where `<deployment>` is the name of the deployment you want to address (e.g., `taskcluster-staging-net`).
If you don't have one yet, skip down to "New Deployments", below.

This script do a fancy dance to set up access to all of the cloud services, and so on.
Just follow its instructions.
This is mostly one-time work for each deployment.
You will need to extract the appropriate secrets file for the deployment from the team passwordstore repository, and paste that in when directed to do so.

Most of the credentials (including these secrets) are cached from run to run in a docker volume, limiting the amount of logging-in you will need to do.

*CAUTION*: that docker volume thus contains powerful cleartext secrets!
Docker volumes are readable by anyone with permission to execute `docker run` on a host.
**DO NOT RUN THIS TOOL ON A SHARED OR UNTRUSTED SYSTEM**!
To delete the volume find it in `docker volume ls` and delete it by name with `docker volume rm`.
You'll need to re-enter all the secret stuff on your next run.

Once setup is complete, the script drops you in a shell at `/repo`.
That's a bind mount of the repository where you ran `./terraform-runner.sh`.
You can run `terraform` as much as you'd like in that docker container.
You can also use the `kubectl`, `gcloud`, `az`, and `aws` tools from this environment to examine and administer the cluster.

All other work (editing files, `git` operations, etc.) should occur outside of the docker container, as usual.
You must install submodules with `git submodule init` and `git submodule update`. If you wish to udpate to a newer version of the remote, add `--remote` to the second command.

### Changing Settings

If you change settings in a deployment configuration file, simply run `setup` again in the docker container.
Similarly, if your cloud credentials expire, run `setup`.

If you need to change secrets, you can edit the file within the docker container at `~/secrets.sh`.
Be sure to keep this in sync with the file in passwordstore.

If you need to change credentials (perhaps you signed into the wrong AWS account?), follow the `docker volume rm` steps above.

### New Deployments

To create a new deployment, make a new directory `deployments/<deployment>/` and create a `main.sh` in it.
See the README in `deployments` for more information.
Ensure that DPL is distinct from any other deployment, or risk creating chaos!

You can use whatever rootUrl you would like, but for a dev environment `https://<somename>.taskcluster-dev.net` is recommended.
The DNS for this zone (as well as for taskcluster-staging.net) is managed in Route53 in the team's staging AWS account.

You will also want to create a secrets file in passwordstore, named after your deployment.
You can copy from another one and change the necessary bits.

Then run `./terraform-runner.sh <your-deployment>`.
Enter all the necessary stuff.

Once you get to a command prompt, run `terraform init` to initialize terraform.

Then run the `terraform import` command that was included in the output, something like:

```shell
terraform import aws_dynamodb_table.dynamodb_tfstate_lock $DPL-tfstate
```

Once that's done, run `terraform init` and `terraform apply -target module.gke`.
Once that succeeds, proceed with `terraform apply` as for an existing deployment.

This is necessary to set up the GKE environment before trying to create Kubernetes resources.
Terraform's dependencies are not expressive enough to capture this.

#### Common Issues

If you are not already logged into Azure in your browser, the link provided by `terraform-runner.sh` will not work.
Instead, follow the link in passwordstore, login, then follow the link provided by `terraform-runner.sh`.

Google's Cloud Console is not compatible with multiple Google accounts (I know, right?).
To use the console with your Mozilla account only, set up the Firefox Containers add-on to always open `https://console.cloud.google.com`.
Then, in that same container, visit that URL and sign in to your Mozilla account.
When prompted to click links in the console to authenticate to gcloud, do so in a tab assigned to this container.

If you are prompted to accept the Googly terms of service, go to `https://console.cloud.google.com` and do so, then run `terraform` again.
Note that you must be careful to login to the console with your work account -- unlike other Google properties, it is not "sticky".
Firefox multi-account containers are helpful.

Sometimes Google's APIs time out.
If that happens, just re-run `terraform apply`.

You might also get an error about billing states -- ".. while in an inactive billing state".
Just re-try running it a few times, as it seems to settle after a few tens of minutes.
If you get an error about the project already existing after this settles, try running
```
terraform import module.gke.google_container_cluster.primary us-east1/taskcluster
```
and then re-running apply.

If GCP says your project name is already taken, that's a shame -- project ID's are global.
Set `TF_VAR_gcp_project` to something unique, perhaps by appending `-2` to your deployment name.

#### DNS/TLS Setup

Once everything is applied, you should see an output named `cluster_ip`.
Due to the "convergent" nature of Kubernetes, it may take a few minutes before this output appears.
It's up to you to configure the DNS for your root URL domain name to point to this IP.
For a `taskcluster-dev.net` subdomain, just add an A record to the Route53 domain in the team's staging AWS account.

Once you do so, you will find that loading the *http* version of your rootUrl will get you some results.
However, *https* may take some time to start working (and https is technically required, so the cluster won't work correctly until this is done).
The cert-manager service is operating in the background to set up a certificate with LetsEncrypt, and once it does so, https URLs will work and http URLs will redirect to https.

Your deployment is ready to go!

### Existing Deployments

The first time you run terraform for a deployment, you will need to run `terraform init` to install all of the various modules.
Once that succeeds, `terraform plan` and `terraform apply` as usual.
If you have not modified anything in the `gke` module, you can go a little faster by adding `-target module.taskcluster`.

## Workers

In general, deploying workers is out of scope for this repo.
In order to get some basic task-execution functionality, these terraform scripts run the [gce-provider-concept](https://github.com/imbstack/taskcluster-gce-provider-concept), which starts some GCE instances running workers.
Without configuration, this service will crash -- and that's fine if you don't need workers.

If you do need workers, you'll need an image (as GCE does not allow public images).
The `workers` directory in this repository has a packer script to create a GCE worker image that is compatible with the temporary gce_provider service.
To build an image, switch to the `workers` directory and run `packer build --var gcp_project_id=<your gcp project> generic-worker.json`.
The build process will take a while (it involves starting an instance, then shutting it down and capturing a snapshot).
When it is done, you will see something like

```
--> googlecompute: A disk image was created: taskcluster-generic-worker-debian-9-1545065836
```

Take that image name and add it to your deployment configuration:
```
export TF_VAR_gce_provider_image_name="taskcluster-generic-worker-debian-9-1545065836
```

The resulting workers have `provisionerId/workerType` `gce-provider/gce-worker-test`.

## Approved Changes

Review is not required for anything under `deployments/<deployment>` for your own dev deployment.
Review never hurts, but for trivial stuff there's no need.

## Terraform-runner Docker Build

To build the docker image, run `./build.sh`.
Note that this image is completely nondeterministic and will pull the latest version of everything.
Caveat ædificator.
