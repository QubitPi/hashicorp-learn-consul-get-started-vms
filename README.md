Learn HashiCorp Consul
======================

<!-- TOC -->
* [Learn HashiCorp Consul](#learn-hashicorp-consul)
  * [Deploy Consul on VMs](#deploy-consul-on-vms)
    * [Service Map](#service-map)
    * [Prerequisites](#prerequisites)
    * [Cloning GitHub Repository](#cloning-github-repository)
    * [Infrastructure Overview](#infrastructure-overview)
  * [Test scenarios](#test-scenarios)
    * [Available environments](#available-environments)
    * [Spin-up environment](#spin-up-environment)
    * [Scenario Output](#scenario-output)
  * [The demo application](#the-demo-application)
  * [Scenarios](#scenarios)
    * [01_consul](#01consul)
    * [02_service_discovery](#02servicediscovery)
    * [03_service_mesh](#03servicemesh)
    * [04_service_mesh_access [DRAFT]](#04servicemeshaccess-draft)
    * [05_service_mesh_monitoring](#05servicemeshmonitoring)
  * [Separate scripts](#separate-scripts)
<!-- TOC -->

> [!TIP]
>
> For most of us, Consul is not an easy-to-pick-up compared to Hashicorp Packer or Terraform. The purpose of this repo 
> is to learn Consul by a mini-project with absolute zero experience with Consul before. When we complete this tutoria,
> we should be having a pretty solid working sense of what it is and reading its documentations afterwards shall be
> relatively easy.

Consul is a service networking solution that enables us to manage secure network connectivity between services and 
across on-premise and multi-cloud environments and runtimes. Consul offers service discovery, service mesh, traffic 
management, and automated updates to network infrastructure device. Check out the
[What is Consul?](https://qubitpi.github.io/hashicorp-consul/consul/docs/intro) page to learn more.

In the following step-by-step instructions, we will deploy a demo application, configure it to use Consul service 
discovery, secure it with service mesh, allow external traffic into the service mesh, and enhance observability into 
our service mesh. During the process, we will learn how to leverage Consul to securely connect our services running on 
any environment. Specifically, using this repo, we will learn to

1. configure, deploy, and bootstrap a Consul server on a virtual machine (VM). After deploying Consul, we will interact 
   with Consul using the UI, CLI, and API
2. deploy Consul client agents to virtual machine (VM) workloads. Then, we will register the services to the Consul 
   catalog and set up a distributed monitoring system using Consul health checks.
3. zero trust security in our network by implementing Consul service mesh. This will enable secure service-to-service 
   communication and allow us to leverage Consul's full suite of features.
4. add a Consul API Gateway in our service mesh and secure external network access to applications and services running 
   in our Consul service mesh
5. configure and use Consul to observe traffic within our service mesh. This enables us to quickly understand how 
   services interact with each other and effectively debug our services' traffic.
6. link our on-premises Consul datacenter to HCP Consul Central, the hosted management plane service available to 
   organizations using HCP Consul.

Deploy Consul on VMs
--------------------

In this section, we will:

- Deploy our VM environment on AWS EC2 using Terraform
- Configure a Consul server
- Start a Consul server instance
- Configure your terminal to communicate with the Consul datacenter
- Bootstrap Consul ACL system and create tokens for Consul management
- Interact with Consul API, KV store and UI

> [!TIP]
> 
> AWS service charges might apply in the following procedures. Don't worry, however, as there is a "one-click" step at
> the end of this section that rollbacks all deployed AWS resources

This repo uses HashiCups, a demo coffee shop application made up of several microservices running on VMs

![HashiCups UI](docs/img/hashicups-ui.png)

The application is composed by a mix of microservices and monolith services running on four different VMs:

1. `NGINX`: an NGINX server configured as reverse proxy for the application. This is the public facing part of our 
   application.
2. `FRONTEND`: the application frontend.
3. `API`: the application API layer. It is composed by multiple services but all of them are running on the same host, 
   waiting to be migrated into microservices.
4. `DATABASE`: a PostgreSQL instance.

![00_hashicups](docs/img/gs_vms-diagram-00.png)

### Service Map

The table shows more details on the services structure.

| Service    | PORT  | Upstream                    |
|------------|-------|-----------------------------|
| database   | 5432  | []                          |
| api        | 8081  | [database]                  |
| - payments | 8080  | []                          |
| - product  | 9090  | *[database]                 |
| - public   | *8081 | [api.product, api.payments] |
| frontend   | 3000  | [api.public]                |
| nginx      | 80    | [frontend, api.public]      |

* `Service`: name of the service. In the environments names are composed using the following naming scheme:

  ```
  hashicups-<service-name>[-<sub-service-name>]
  ```

* `Port`: the port on which the service is running. The *api* meta-service is using multiple ports, one per 
  sub-service. The one marked with a `*` is the only one that requires external access.
* `Upstream`: represent the dependencies each service has. If one of the services mentioned in this column is not 
  working properly, then the service will not work either.

### Prerequisites

In order to proceed, we will first need:

- An AWS account configured for
  [use with Terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs#authentication-and-configuration).
  For example, we set IAM credentials to authenticate the Terraform AWS provider as a pair of environment variables:

  ```console
  export AWS_ACCESS_KEY_ID=
  export AWS_SECRET_ACCESS_KEY=
  ```

- [aws-cli >= 2.0](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [terraform >= 1.0](https://qubitpi.github.io/hashicorp-terraform/terraform/install)
- [consul >= 1.15.0](https://qubitpi.github.io/hashicorp-consul/consul/install)

### Cloning GitHub Repository

Clone the [GitHub repository](https://github.com/QubitPi/hashicorp-learn-consul-get-started-vms) containing the 
configuration files and resources.

```console
git clone https://github.com/hashicorp-education/learn-consul-get-started-vms
```

Change into the directory that contains the complete configuration files:

```console
cd learn-consul-get-started-vms/self-managed/infrastructure/aws
```

### Infrastructure Overview

- [ami.tf](./self-managed/infrastructure/aws/ami.tf): All of our VM's runs on Debian OS


Deploy:

```
terraform apply --auto-approve -var-file=./conf/00_hashicups.tfvars
```

![00_hashicups](docs/img/gs_vms-diagram-00.png)

* HashiCups application installed on 4 VMs
* One empty VM ready to host Consul server
* One *Bastion Host* VM with all tools pre-installed to interact with the environment.


## Test scenarios

Using the pre-configured environments you can spin-up scenarios to be used in
tutorials or as test environments.

### Available environments

The environments are defined inside the `./self-managed/infrastructure/aws/conf` 
folder:

```
tree ./self-managed/infrastructure/aws/conf
```

```
./self-managed/infrastructure/aws/conf
├── 00_hashicups.tfvars
├── 01_consul.tfvars
├── 02_service_discovery.tfvars
├── 03_service_mesh.tfvars
├── 04_service_mesh_access.tfvars
└── 05_service_mesh_monitoring.tfvars

0 directories, 6 files
```

### Spin-up environment

> **Info:** only AWS cloud provider scenarios are available ATM.

Use one of the configurations available as parameter for `terraform`:

```
cd ./infrastructure/aws
```

First, initialize the Terraform project.

```
terraform init
```

Optionally, reset your environment.

```
terraform destroy --auto-approve && \
    terraform apply --auto-approve -var-file=./conf/00_hashicups.tfvars
```

### Scenario Output

> **Info:** the tool is currently under active development, the info in this section 
might be outdated. Refer to the actual output of your `terraform apply` command.

```
terraform output
```

```plaintext
connection_string = "ssh -i certs/id_rsa.pem admin@`terraform output -raw ip_bastion`"
ip_bastion = "35.161.114.102"
remote_ops = "export BASTION_HOST=35.161.114.102"
retry_join = "provider=aws tag_key=ConsulJoinTag tag_value=auto-join-j26f"
ui_consul = "https://35.87.2.143:8443"
ui_grafana = "http://35.161.114.102:3000"
ui_hashicups = "http://34.216.72.145"
```

* `connection_string`: is the command you can use to connect via SSH to the 
Bastion Host. By changing the `ip_bastion` to one of the `ip_*` output names you
can connect to the respective host.

* `ip_bastion`: Public IP of the *Bastion Host*. This is a VM containing all 
necessary tools to interact with the environment. The tool uses this host as a 
entrypoint into the environment and as a centralized scenario orchestrator. Read 
more on [Bastion Host](./docs/BastionHost.md).

* `ip_*`: Public IP of the `*` host. Used for UI and SSH connections.

* `ui_consul`: UI for Consul Control Plane. Some scenario do not start Consul
servers or do not deploy an HCP Consul cluster. In those scenarios the URL will 
not work or might be empty. This is expected for those scenarios that test the
Consul daemon lifecycle.

* `ui_hashicups`: UI for the Grafana monitoring tool. The tool is running from
the Bastion Host VM.

* `ui_hashicups`: UI for the test application, HashiCups, deployed in all 
scenarios. For most scenarios the UI corresponds to the `nginx` host.

## The demo application

All the scenarios include a demo application, *HashiCups*, used to demonstrate 
Consul features in a setting that can mimic real-life deployments.

Read more in the [HashiCups Demo Application](./docs/HashiCups.md) page.

## Scenarios

Here a brief description of the available scenarios.



### 01_consul

Deploy:

```
terraform apply --auto-approve -var-file=./conf/01_consul.tfvars
```

![01_consul](docs/img/gs_vms-diagram-01.png)

* All steps of scenario `00_hashicups`.
* Consul server configured for TLS and Gossip encryption and with restrictive ACL defaults.
* Consul server running.
* Consul ACL system initialized.
* Restricted token applied to the Consul server agent.

### 02_service_discovery

Deploy:

```
terraform apply --auto-approve -var-file=./conf/02_service_discovery.tfvars
```

![02_service_discovery](docs/img/gs_vms-diagram-02.png)

* All steps of scenario `01_consul`.
* `for each service`
    * Consul client configured for TLS and Gossip encryption and with restrictive ACL defaults.
    * Consul client tokens generated and added to the Consul configuration.
    * Service discovery configuration files created and added the the Consul configuration.
    * Consul client running

### 03_service_mesh

Deploy:

```
terraform apply --auto-approve -var-file=./conf/03_service_mesh.tfvars
```

![03_service_mesh](docs/img/gs_vms-diagram-03.png)

* All steps of scenario `02_service_discovery`.
* `for each service`
    * Consul service mesh configuration files created for the service and added to the Consul configuration replacing the service discovery configuration.
    * Consul client agent restarted to apply the configuration change
    * Envoy sidecar proxy running
    * Service process configuration change to use only `localhost` references.
    * Service process restarted to listen on `localhost` interface only.
* Permissive `* > *` intention configured,to allow traffic.


### 04_service_mesh_access [DRAFT]

Deploy:

```
terraform apply --auto-approve -var-file=./conf/04_service_mesh_access.tfvars
```

![04_service_mesh_access](docs/img/gs_vms-diagram-04.png)

* All steps of scenario `03_service_mesh`.
* Consul server configuration changed for service mesh access (API Gateway).

### 05_service_mesh_monitoring

Deploy:

```
terraform apply --auto-approve -var-file=./conf/05_service_mesh_monitoring.tfvars
```


![04_service_mesh_monitoring](docs/img/gs_vms-diagram-05.png)


* All steps of scenario `04_service_mesh_access`.
* Consul server configuration changed for monitoring.
* Consul server agent restarted to apply the configuration change
* `for each service`
    * Grafana agent configuration generated
    * Grafana agent process started

## Separate scripts

During the environment setup, some steps of the UX are performed using external scripts. 
Those steps are usually the operations more dependent on your specific deployment.

We detached those steps into supporting scripts to:
* provide easier reference for those steps
* avoid code drift for the core parts of the workflow

The scripts are located under the `./ops/scenarios/99_supporting_scripts/` folder:

```
tree ./ops/scenarios/99_supporting_scripts/
./ops/scenarios/99_supporting_scripts/
├── generate_consul_client_config.sh
├── generate_consul_server_config.sh
├── generate_consul_server_tokens.sh
└── generate_consul_service_config.sh

0 directories, 4 files

```

You can use the scripts in this folder as a source of inspiration for writing your custom configuration scripts.

Read more on the different scripts on the [Separate Scripts](./docs/SeparateScripts.md) page.