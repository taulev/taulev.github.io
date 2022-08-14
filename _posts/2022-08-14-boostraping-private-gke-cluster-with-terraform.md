Kubernetes is an orchestration tool which automates deployment, scaling, and management of containerized applications. GKE (Google Kubernetes Engine) is a manged Kubernetes platform offering by Google Cloud, it takes away the pain of managing Kubernetes control plane. In this post, I will be sharing the steps required to easily bootstrap a private GKE cluster with Terraform. 


&nbsp;
## Creating a service account
First, you will need to create a service account within Google Cloud Platform (GCP). Open the GCP console and navigate to **IAM & Admin -> Service Accounts -> Create service account (at the top)** and fill out the details. 

There are multiple ways for service account to authenticate with GCP, but for convenience we will be using service account keys. In the same service account window, click on 3 dots under actions and select **Manage keys**. In the keys window, click **Add key -> Create new key -> Select JSON -> Create**, the key will be automatically downloaded.

&nbsp;
## Configuring Terraform providers
First create a root folder where you will hold Terraform files, in this post the folder name we will be using is `gke`. In the `gke` folder, create the following files:
- `config.tf` - will contain provider information
- `main.tf` - will have our resource definitions

You should also move the service account key file to this folder, I name my service account key file `account.json`. Your directory structure should look like:
```
gke
├── account.json
├── config.tf
├── main.tf
```

Open the `config.tf` file and fill it with the following text:
```
terraform {
  required_providers {
    google = {
      version = "~> 4.31.0"
    }
  }
}
```
We specify that we will be using Google provider of version *4.31.x*, `~>` specifies that the rightmost component of the version can be incremented.

&nbsp;
## Bootstrapping the GKE cluster
Now that we have our provider configured, we are ready to define Terraform resources, that allows us to create a GKE cluster with one `terraform apply` command.

&nbsp;
#### Enabling required GCP APIs
By default, the GKE API is not enabled and without it, Terraform won't be able to provision a GKE cluster. This could be enabled manually, but we can also enable it via Terraform. Open `main.tf` and add the following lines (**you should change "your-project-id" with your actual project id**):
```
locals {
  api = [
    "cloudresourcemanager.googleapis.com",
    "container.googleapis.com",
  ]
}

resource "google_project_service" "apis" {
  count = length(local.api)

  service                    = local.api[count.index]
  project                    = "your-project-id"
  disable_dependent_services = true
}
```

Here we define a local variable `api` which is a list of APIs that we will be enabling, and we later access it via the `local` keyword (as can be seen in `service` value). `cloudresourcemanager.googleapis.com` enables Cloud Resource Manager API, which allows us to interact with GCP resources, while `container.googleapis.com` enables Kubernetes Engine API, which allows us to provision GKE clusters. We are using `count` to create multiple `google_project_service` resources with a single definition.

&nbsp;
#### Creating a VPC network
For the GKE cluster, we will also create a private VPC network. We will be using the predefined [Google Network](https://registry.terraform.io/modules/terraform-google-modules/network/google/5.2.0) module. A module is basically a collection of resources that are used together. Append your `main.tf` with:
```
module "gke_network" {
  source  = "terraform-google-modules/network/google"
  version = "~> 5.2.0"

  network_name = "gke-network"
  project_id   = "your-project-id"
  subnets = [
    {
      subnet_name           = "gke-subnetwork"
      subnet_ip             = "10.1.0.0/16"
      subnet_private_access = "true"
      subnet_region         = "europe-west3"
    },
  ]
  secondary_ranges = {
    "gke-subnetwork" = [
      {
        ip_cidr_range = "10.2.0.0/16"
        range_name    = "pod-ip-range"
      },
      {
        ip_cidr_range = "10.3.0.0/16"
        range_name    = "service-ip-range"
      },
    ]
  }

  depends_on = [google_project_service.apis]
}
```
With `source` we tell Terraform where to look for the module and everything else is pretty self-explanatory - we create a VPC and a subnetwork within that VPC, then we specify secondary IP ranges which will be used for Kubernetes Pods and Services (for real life clusters you should carefully consider subnet ranges as per [docs](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips#defaults_limits)). Finally, we use `depends_on` to ensure that Terraform won't attempt to provision these resources until APIs are enabled.

&nbsp;
#### Provisioning GKE cluster
Finally, we are ready to provision our GKE cluster. Again, we will be using a [Terraform module](https://registry.terraform.io/modules/terraform-google-modules/kubernetes-engine/google/22.1.0) to make our life a lot easier. Add these lines to `main.tf`:
```
module "gke" {
  source  = "terraform-google-modules/kubernetes-engine/google//modules/private-cluster"
  version = "~> 22.1.0"

  enable_private_nodes = true
  ip_range_pods        = "pod-ip-range"
  ip_range_services    = "service-ip-range"
  master_authorized_networks = [
    { cidr_block = "1.2.3.4/32", display_name = "First IP that will have access to Control Plane" },
    { cidr_block = "5.6.7.8/32", display_name = "Second IP that will have access to Control Plane" },
  ]
  name           = "gke-cluster"
  network        = module.gke_network.network_name
  network_policy = true
  node_pools = [
    {
      name         = "my-node-pool"
      machine_type = "e2-small"
      min_count    = 1
      max_count    = 3
      disk_size_gb = 30
    },
  ]
  project_id               = "your-project-id"
  region                   = "europe-west3"
  regional                 = false
  remove_default_node_pool = true
  subnetwork               = "gke-subnetwork"
  zones                    = ["europe-west3-a"]

  depends_on = [google_project_service.apis, module.gke_network]
}

module "auth" {
  source  = "terraform-google-modules/kubernetes-engine/google//modules/auth"
  version = "~> 22.1.0"

  cluster_name = module.gke.name
  location     = module.gke.location
  project_id   = "your-project-id"
  
  depends_on = [google_project_service.apis, module.gke]
}

resource "local_file" "kubectlconfig" {
  content  = module.auth.kubeconfig_raw
  filename = "gke-cluster-config"
}
```

Things worth mentioning:
1. `enable_private_nodes = true` ensures that created nodes do not have external IP address. As we are creating a private cluster, we want to limit public endpoint accessibility, we could disable it, but instead we will filter IPs which can access this public endpoint. We achieve that with `master_authorized_networks`.
2. You can use `node_pools` to define your list of node pools that should be created. Here we create only one pool with e2-small instances. We allow a maximum of 3 nodes for autoscaler to scale if we lack resources.
3. As this is a tutorial, we are creating a zonal cluster by setting `regional = false` and `zones = ["europe-west3-a"]`. Zones specify in which zones the cluster can be hosted. Zonal cluster has only 1 Control plane, so if you want high availability you should change `regional` to true, and then you can omit `zones` variable as it is not required for regional cluster.
4. `module "auth"` and `resource "local_file" "kubectlconfig"` are optional, however they are convenient as they will create you a config file that allows you to authenticate with your GKE cluster.

&nbsp;
#### terraform init && terraform apply
Our `main.tf` is ready and all that is left for us to do it initialize terraform providers/modules and apply the configuration. Before you do that, you need to set the `GOOGLE_APPLICATION_CREDENTIALS` environment variable to point to our `account.json` file. You can achieve this with:
`export GOOGLE_APPLICATION_CREDENTIALS=/path/to/gke/account.json`

Once the environment variable is set, run `terraform init` followed by `terraform apply`. After running `terraform apply` you will be provided with a list of resources that will be provisioned and will be asked for confirmation - type `yes` press `Enter` on your keyboard and go grab a drink as the provisioning will take some time, usually it is around 15-20 minutes.


&nbsp;
## Final Words
And here you have it - an easy way to provision a Private Cluster in GKE using Terraform! It is worth mentioning that the configuration here is simplified - usually you would want to define variables in `variables.tf` and use those variables instead of direct values as in this post, also you would probably want to separate resources in different files, in this case you could move API enabling to `api.tf` and VPC creating to `network.tf`.


#### Complete main.tf
```
locals {
  api = [
    "cloudresourcemanager.googleapis.com",
    "container.googleapis.com",
  ]
}

resource "google_project_service" "apis" {
  count = length(local.api)

  service                    = local.api[count.index]
  project                    = "your-project-id"
  disable_dependent_services = true
}

module "gke_network" {
  source  = "terraform-google-modules/network/google"
  version = "~> 5.2.0"

  network_name = "gke-network"
  project_id   = "your-project-id"
  subnets = [
    {
      subnet_name           = "gke-subnetwork"
      subnet_ip             = "10.1.0.0/16"
      subnet_private_access = "true"
      subnet_region         = "europe-west3"
    },
  ]
  secondary_ranges = {
    "gke-subnetwork" = [
      {
        ip_cidr_range = "10.2.0.0/16"
        range_name    = "pod-ip-range"
      },
      {
        ip_cidr_range = "10.3.0.0/16"
        range_name    = "service-ip-range"
      },
    ]
  }

  depends_on = [google_project_service.apis]
}

module "gke" {
  source  = "terraform-google-modules/kubernetes-engine/google//modules/private-cluster"
  version = "~> 22.1.0"

  enable_private_nodes = true
  ip_range_pods        = "pod-ip-range"
  ip_range_services    = "service-ip-range"
  master_authorized_networks = [
    { cidr_block = "1.2.3.4/32", display_name = "First IP that will have access to Control Plane" },
    { cidr_block = "5.6.7.8/32", display_name = "Second IP that will have access to Control Plane" },
  ]
  name           = "gke-cluster"
  network        = module.gke_network.network_name
  network_policy = true
  node_pools = [
    {
      name         = "my-node-pool"
      machine_type = "e2-small"
      min_count    = 1
      max_count    = 3
      disk_size_gb = 30
    },
  ]
  project_id               = "your-project-id"
  region                   = "europe-west3"
  regional                 = false
  remove_default_node_pool = true
  subnetwork               = "gke-subnetwork"
  zones                    = ["europe-west3-a"]

  depends_on = [google_project_service.apis, module.gke_network]
}

module "auth" {
  source  = "terraform-google-modules/kubernetes-engine/google//modules/auth"
  version = "~> 22.1.0"

  cluster_name = module.gke.name
  location     = module.gke.location
  project_id   = "your-project-id"
  
  depends_on = [google_project_service.apis, module.gke]
}

resource "local_file" "kubectlconfig" {
  content  = module.auth.kubeconfig_raw
  filename = "gke-cluster-config"
}
```
