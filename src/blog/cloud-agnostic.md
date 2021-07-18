<h1>How we achieved Cloud-agnostic infrastructure</h1>
<div class="center">2021-07-17</div>

Author // [Nocoto Day](http://notcoding.today)  
Status // Final

## Goal

This blog explains how we achieved distributed, reliable, scalable and Cloud-agnostic infrastructure.

Note: Using infrastructure setup does not automagically make your application reliable.
You should follow fundamental state management and distributed computing principles (not discussed in detail in this post).

## Summary

- Treat any Cloud provider as simple VM providers with API to manage these VMs.
- Use [Terraform](https://www.terraform.io/) to manage VMs and DNS.
- Package your binary and flags into [Docker containers](https://www.docker.com/) to deploy to Cloud.
- Use [Nomad](https://www.nomadproject.io/) to turn up jobs - both for Cloud and local environments.

## Why Cloud-agnostic

Fundamentals of being _Cloud-agnostic_ is being able to operate with _any_ public Cloud provider.

Why would you consider Cloud-agnostic infrastructure from the start?

- **Ability to jump to another vendor at any time, save resources.** Just because certain provider meets your needs right now does not mean they will in a few years time. Your tech debt might be big enough that migrating to another vendor may become impossible by the time you need to. This naturally reduces your running costs, since you can lean towards cheapest option at any time.
- **Full control and insight over your infrastructure.** You can optimize and customize your infrastructure in just the way your company needs.
- **Achieve extremely high SLA.** Your infrastructure will be resistant to Cloud provider outages. Because it is very rare to have multiple Cloud outages in the same region at once (unless there are problems in the real world), if setup correctly, beyond five-nine SLA is not impossible.
- **Looks good on your resume.** Whatever your building, at minimum you want it to look good in your resume. As someone who interviews mid-senior to senior Software Engineers, I would generally be more impressed by custom-architecture projects.

What are the caveats?

- **Potentially slower development speed.** Sticking into one Cloud provider and leveraging first-party tools to glue an application together is significantly faster and easier initially.
- **You will spend more time in security.** This depends on the setup. If you are mix-and-matching various Cloud providers, you inevitably have to expose public ports and IPs for inter-service communications. This creates more security holes you need to worry about.

What are some other things to note?

- **Cloud-agnostic impacts staffing and training.** Because your team does not care about Cloud-specific knowledge, this may give you better staffing chances. But this also means you may have to train people from 'scratch' (but fundamental knowledge of any Cloud services should carry over).

## Terraform to manage Clouds

To be Cloud-agnostic, you need a common configuration that applies to multiple Clouds - ie. you want to stop using the web UI to configure your VMs, etc. [Terraform](https://www.terraform.io/) allows us to do this.

You will write a bunch of `HCL` files (imagine JSON with a bit of built-in functions) to configure your Clouds. In the following example, let's use Linode and Vultr to turn up our VMs:

`main.tf`

```tf
terraform {
  required_providers {
    linode = {
      source = "linode/linode"
    }
    vultr = {
      source = "vultr/vultr"
    }
  }
}

provider "vultr" {
  # export VULTR_API_KEY=key
}

provider "linode" {
  # export LINODE_TOKEN=token
}
```

This simply defines which providers are involved. Instead of checking-in your secret keys, please set environment variables in your local machine.

In addition, we have a separate development domain that is managaed via Linode (so that we don't have to type IPs around or hardcode any). You can use any DNS to achieve this, ex. [Cloudflare](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs):

`domains.tf`

```tf
resource "linode_domain" "eggricesoy" {
    type = "master"
    domain = "your_domain"
    soa_email = "your@email.com"
}
```

You can now actually define VMs and associated domain names for each. For example:

`dev.tf`

```tf
resource "linode_instance" "nomad-dev-COUNTRY_CODE-CITY_CODE0" {
  label          = "nomad-dev-COUNTRY_CODE-CITY_CODE0"
  image          = "linode/debian10"
  region         = "your_region"
  type           = "your_machine_type"
}

resource "linode_domain_record" "nomad-dev-COUNTRY_CODE-CITY_CODE0" {
  domain_id = linode_domain.eggricesoy.id
  name = "dev-COUNTRY_CODE-CITY_CODE0"
  record_type = "A"
  target = linode_instance.nomad-dev-COUNTRY_CODE-CITY_CODE0.ip_address
}

resource "vultr_instance" "nomad-dev-COUNTRY_CODE-CITY_CODE1" {
  plan = "your_machine_type"
  region = "your_region"
  # Debian 10
  os_id = "352"
}

resource "linode_domain_record" "nomad-dev-COUNTRY_CODE-CITY_CODE1" {
  domain_id = linode_domain.eggricesoy.id
  name = "dev-COUNTRY_CODE-CITY_CODE1"
  record_type = "A"
  target = vultr_instance.nomad-dev-COUNTRY_CODE-CITY_CODE1.main_ip
}
```

Documentation for each providers and modules can be found at [registry.terraform.io](https://registry.terraform.io/). They are mostly well-written documented and easy to find.

To push infrastructure intent to reality, you run:

```bash
$ terraform apply
```

If you want to take it further, you may want to define start-up scripts for each Cloud provider and run initial scripts to prepare your VMs. I have a script that installs Docker and Nomad, then schedules Nomad on startup. I just do `ssh root@domain "bash -s" < script` however.

## Docker containers for release

Containerise your releases and promote the containers through a scheduled release process.

When you containerise, you want to embed flags to go with the binary together. The huge benefit of doing this is **easy rollbacks when things go wrong**. Flags appear and disappear as source code evolves, and only certain individuals in your team may know the correct set of values (ex. optimization flags). You want to version your flag changes together with the code, at all times.

Each release stage should have constant tests and integrations to verify a successful release. For example:

- dev: build from HEAD and containerise every 2 hours. Push latest dependencies together with the release. Tests basic functionality like 'does it turn up and does basic things in job level'.
- staging: promote the latest successful release at dev every day. Test overall infrastructure actions (ex. add DB replica).
- prod: promote the latest successful release at staging every week.

## Nomad to run containers

[Nomad](https://www.nomadproject.io/) is like [Kubernetes](https://kubernetes.io/) but with actually readable and discoverable tutorials and reference docs. Nomad manages and runs your binaries for you, according to the job configuration.

I personally like Nomad better due to its simplicity of setting up. I can't be bothered configuring and setting up Kube myself. I usually rely on the Cloud provider to configure Kube cluster for me. This means one cluster = one Cloud provider, hence if that Cloud provider explodes, my cluster is lost.

With Nomad, you can mix and match providers across various regions. If setup correctly, your cluster will not go down even if one of the Cloud providers explode.

In Nomad, you have "servers" that do job orchestration and "clients" that actually run Docker containers. There are a couple of things to keep in mind when setting up Nomad cluster:

- A Nomad cluster typically comprises three or five servers (but no more than seven). This is slightly less than ideal but should be enough to cover three continents using two providers each.
- Create a domain for your servers and clients to join, instead of using IP addresses. Ideally, use DNS Round Robin with all of your servers. Remember, use your Terraform config to set this up.
- Use ACLs and policies. Keep it safe.

Configuring jobs in Nomad uses the same language as Terraform (HCL). For example, to run nginx with no rollout policy and in the raw-est form possible:

```nomad
job "nginx" {
  region = "REGION"
  datacenters = ["CITY_CODE"]

  group "nginx" {
    count = 1

    network {
      port "http" {
        to = 80
      }
    }

    service {
      name = "nginx"
      port = "http"
    }

    task "nginx" {
      driver = "docker"

      config {
        image = "nginx"
        ports = ["http"]
      }
    }
  }
}
```

## Other things to keep in mind

As mentioned in the beginning, simply using this infrastructure does not make your service more reliable or scalable. However, this infrastructure will set you strong foundations to run a stable service without blowing your budget on hiring SREs.

After you've set things up following this blog post, you should look into how binary rollouts should happen across the globe and how to manage service-level states.
