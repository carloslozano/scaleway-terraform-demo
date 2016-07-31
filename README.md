## TODO

- adjust nomad plan & run examples now that direct access is locked down (scp files, run from jump host /w private IP)

# Terraform on Scaleway

Terraform is a cloud agnostic automation tool to safely and efficiently manage infrastructure as the configuration is evolved.
In its latest version, Terraform ships [Scaleway](https://scaleway.com) support. Terraform is a great tool and combination with Scaleway to version and continuously develop your infratructure with ease.

In this blog post I showcase Terraform new capabilities.  

Specifically I'll setup:

- a Consul cluster. [Consul](https://consul.io) was build for service discovery, orchestration & configuration. To be able to dynamically change running services, consul will be used to provide health checks and service discovery for the Nomad cluster. 
- a Nomad cluster. [Nomad](https://nomadproject.io) is a distributed scheduler which allows us to run arbitrary tasks on a nomad cluster. To achieve higher utilization and ease overall
development nomad helps abstract instances, instead exposing a declarative syntax to specify what
services you want to run. Nomad will ensure that the services are always running and also take care of dynamic re-allocation if instances are unavailable.
- fabio loadbalancer. [fabio](https://github.com/eBay/fabio) was build to support a dynamic service landscape, where instances & services might fail unexpectedly. The native consul integration allows for zero configuration usage, relying on consuls data for configuration and service discovery. Fabio allows us to expose a web api, while not worrying about service relocation in case of instance failures.
- AWS Route 53. To be able to access our service via a publicly accessible DNS entry, I will bridge with AWS Route 53. AWS Route 53 is a mature DNS service, which together with fabio, consul and nomad will allow us to deploy software with certainty of availability.

If you like diving into code: the entire example is available on [github](https://github.com/nicolai86/scaleway-terraform-demo).

## Requirements

I assume that you have terraform >= v0.7.0 installed and are already a Scaleway user. If not, you can create your account in seconds [here](scaleway.com/pricing).

First, set the following environmental variables to allow Terraform to interact with the Scaleway APIs:

```
export SCALEWAY_ACCESS_KEY=<some-key> 
export SCALEWAY_ORGANIZATION=<some-org>
```

Exporting environmental variables lets you run Terraform without specifying any credentials in the Terraform configuration file:

```
# main.tf
provider "scaleway" {}

```

## Preparations 

I use a jump host, so my consul cluster instances are not publicly accessible and 
setup a security group to lock external requests to my Nomad cluster requests out.
Nomad uses ports `4646`, `4647` and `4648` for http, rpc and serf.

I'll allow data center internal traffic, and drop outside traffic on the same port: 

```
# modules/security_group/main.tf
resource "scaleway_security_group" "cluster" {
  name        = "cluster"
  description = "cluster-sg"
}

resource "scaleway_security_group_rule" "accept-internal" {
  security_group = "${scaleway_security_group.cluster.id}"

  action    = "accept"
  direction = "inbound"

  # NOTE this is just a guess - might not work for you.
  ip_range = "10.1.0.0/16"
  protocol = "TCP"
  port     = "${element(var.nomad_ports, count.index)}"
  count    = "${length(var.nomad_ports)}"
}

resource "scaleway_security_group_rule" "drop-external" {
  security_group = "${scaleway_security_group.cluster.id}"

  action    = "drop"
  direction = "inbound"
  ip_range  = "0.0.0.0/0"
  protocol  = "TCP"

  port  = "${element(var.nomad_ports, count.index)}"
  count = "${length(var.nomad_ports)}"

  depends_on = ["scaleway_security_group_rule.accept-internal"]
}
```

Now that our network is no longer publicly accessible, let's start setting up a jump host:

```
# modules/jump_host/main.tf
resource "scaleway_server" "jump_host" {
  name                = "jump_host"
  image               = "${var.image}"
  type                = "${var.type}"
  dynamic_ip_required = true

  tags = ["jump_host"]

  security_group = "${var.security_group}"
}

resource "scaleway_ip" "jump_host" {
  server = "${scaleway_server.jump_host.id}"
}

output "public_ip" {
  value = "${scaleway_ip.jump_host.ip}"
}
```

As you can see I'm using the `scaleway_ip` resource to request a public IP which can 
outlive the instance it's attached to. This way you can re-create the jump host without
loosing the IP you're using.

We'll be using our jump host module like this: 

```
# main.tf
provider "scaleway" {}

module "security_group" {
  source = "./modules/security_group"
}

module "jump_host" {
  source = "./modules/jump_host"

  security_group = "${module.security_group.id}"
}
```

Next, let's setup our consul clusters:

## Setting up consul

We'll take the [official consul terraform AWS module](https://github.com/hashicorp/consul/tree/master/terraform/) and adjust it for our needs:

First, the core of our setup is the `scaleway_server` resource. To slightly simplify the setup we'll default to ubuntu 16.04 and `systemd`. The image & commercial server type have been moved into a separate variables file.
Lastly, we have to modify the `install.sh` script to install the [official consul ARM binary](https://releases.hashicorp.com/consul/0.6.4/):

```
# modules/consul/main.tf
resource "scaleway_server" "server" {
  count               = "${var.server_count}"
  name                = "consul-${count.index + 1}"
  image               = "${var.image}"
  type                = "${var.type}"
  dynamic_ip_required = true

  tags = ["consul"]

  connection {
    type         = "ssh"
    user         = "root"
    host         = "${self.private_ip}"
    bastion_host = "${var.bastion_host}"
    bastion_user = "root"
    agent        = true
  }

  provisioner "file" {
    source      = "${path.module}/scripts/rhel_system.service"
    destination = "/tmp/consul.service"
  }

  provisioner "remote-exec" {
    inline = [
      "echo ${var.server_count} > /tmp/consul-server-count",
      "echo ${scaleway_server.server.0.private_ip} > /tmp/consul-server-addr",
    ]
  }

  provisioner "remote-exec" {
    scripts = [
      "${path.module}/scripts/install.sh",
      "${path.module}/scripts/service.sh",
    ]
  }
}
```

Since we'll be using terraform with `remote-exec` we have to request a public IP via `dynamic_ip_required = true`.
To accomodate the bundled installation scripts we'll packaged the entire setup into one terraform module called `consul`.  

Also note the use of the `connection` attribute to instruct terraform to use our jump host to
connect to any consul instance.

Now, lets use our new consul module:

```
# main.tf
provider "scaleway" {}

module "security_group" {
  source = "./modules/security_group"
}

module "jump_host" {
  source = "./modules/jump_host"

  security_group = "${module.security_group.id}"
}

module "consul" {
  source = "./modules/consul"

  security_group = "${module.security_group.id}"
  bastion_host   = "${module.jump_host.public_ip}"
}
```

Now, to get a running consul cluster we have to let terraform do its job:

```
$ terraform get       # tell terraform to lookup referenced consul module 
$ terraform apply
```

This will take some minutes. Once done, verify that our cluster consists of two servers.
First, find out the public IP of our jumphost:

```
$ terraform output -module=jump_host
  public_ip = 212.47.227.252
```

Next, let's lookup the private IPs of our consul servers, and query the member list:

```
$ terraform show | grep private_ip
  private_ip = 10.1.40.120
  private_ip = 10.1.17.22
$ ssh root@212.47.227.252 'consul members -rpc-addr=10.1.40.120:8400'
Node      Address           Status  Type    Build  Protocol  DC
consul-1  10.1.40.120:8301  alive   server  0.6.4  2         dc1
consul-2  10.1.17.22:8301   alive   server  0.6.4  2         dc1
```

Note that it doesn't matter which server we talk to.

> TODO describe how to lookup IMAGE uuid. Right now I'm working around the issue: create a server, describe via API, use image uuid from response. scw images --no-trunc doesn't work for ARM… :?

Lets continue with setting up our nomad cluster!

## Setting up nomad

> TODO write up how to generate nomad binary. Basically start Scaleway server, install go arm, gcc & git; go get nomad, modify `scripts/build.sh` to remove linux/arm from excluded targets; compile; download binary

Nomad doesn't supply prebuild ARM binaries. Luckily we can compile a working binary on a Scaleway ARM server easily. I've included nomad v0.4.0 in this repository, and we'll be using this binary for the rest of this post.

The basic nomad setup looks very similar to our consul setup: again we're using Ubuntu 16.04 as base image; nomad will be supervised by `systemd`. The notable differences are:

- the nomad binary is uploaded from our local machine
- the nomad configuration file is generated inline
- nomad will use the existing consul cluster. this allows nomad to bootstrap itself easily

```
resource "scaleway_server" "server" {
  count               = "${var.server_count}"
  name                = "nomad-${count.index + 1}"
  image               = "${var.image}"
  type                = "${var.type}"
  dynamic_ip_required = true
  tags                = ["cluster"]

  provisioner "file" {
    source      = "${path.module}/scripts/rhel_system.service"
    destination = "/tmp/nomad.service"
  }

  provisioner "file" {
    source      = "${path.module}/scripts/nomad_v0.4_linux_arm"
    destination = "/usr/local/bin/nomad"
  }

  provisioner "remote-exec" {
    inline = <<CMD
cat > /tmp/server.hcl <<EOF
datacenter = "dc1"

bind_addr = "${self.private_ip}"

advertise {
  # We need to specify our host's IP because we can't
  # advertise 0.0.0.0 to other nodes in our cluster.
  serf = "${self.private_ip}:4648"
  rpc = "${self.private_ip}:4647"
  http= "${self.private_ip}:4646"
}

# connect to consul for cluster management
consul {
  address = "${var.consul_cluster_ip}:8500"
}

# every node will be running as server as well as client…
server {
  enabled = true
  # … but only one node bootstraps the cluster
  bootstrap_expect = ${element(split(",", "1,0"), signum(count.index))}
}

client {
  enabled = true

  # enable raw_exec driver. explanation will follow
  options = {
    "driver.raw_exec.enable" = "1"
  }
}
EOF
CMD
  }

  provisioner "remote-exec" {
    scripts = [
      "${path.module}/scripts/install.sh",
      "${path.module}/scripts/service.sh"
    ]
  }
}
```

Again we've extract this into a module to bundle both the binary and the required scripts.

Our main terraform file hides the entire setup complexity for us:

```
# main.tf
provider "scaleway" {}

module "security_group" {
  source = "./modules/security_group"
}

module "consul" {
  source = "./modules/consul"

  security_group = "${module.security_group.id}"
}

module "nomad" {
  source = "./modules/nomad"

  consul_cluster_ip = "${module.consul.server_ip}"
  security_group    = "${module.security_group.id}"
}
```

Again, let's use terraform to get our nomad cluster up and running:

```
$ terraform get    # tell terraform to lookup referenced consul module 
$ terraform plan   # we should see two new nomad servers
$ terraform apply
```

This will take some minutes again. And again, let's verify the setup was actually successful. We should see two nomad nodes, where one node is marked as leader:

```
$ ssh root@163.172.160.218 'nomad server-members -address=http://10.1.38.33:4646'
Name            Address     Port  Status  Leader  Protocol  Build  Datacenter  Region
nomad-1.global  10.1.36.94  4648  alive   false   2         0.4.0  dc1         global
nomad-2.global  10.1.38.33  4648  alive   true    2         0.4.0  dc1         global
```

Also, let's verify that nomad registered with our consul cluster.

```
$ ssh root@212.47.227.252 'curl -s 10.1.42.50:8500/v1/catalog/services' | jq 'keys'
[
  'consul',
  'nomad',
  'nomad-client'
]
```

If you see `consul`, `nomad` and `nomad-client` in the output everything is working so far.

Before we proceed we need to verify that the ARM binary of nomad properly reports resources, otherwise we can't schedule jobs in our cluster:

```
$ ssh root@163.172.171.232 'nomad node-status -self -address=http://10.1.38.151:4646'
ID     = c8c8a567
Name   = nomad-1
Class  = <none>
DC     = dc1
Drain  = false
Status = ready
Uptime = 4m55s

Allocated Resources
CPU        Memory       Disk        IOPS
0/5332000  0 B/2.0 GiB  0 B/16 EiB  0/0

Allocation Resource Utilization
CPU        Memory
0/5332000  0 B/2.0 GiB

Host Resource Utilization
CPU             Memory           Disk
844805/5332000  354 MiB/2.0 GiB  749 MiB/46 GiB
```

> this did not work with nomad v0.4.0 and ubuntu 14.04. But it works with ubuntu 16.04.

Everything is looking great. Let's run some software!

## Running fabio

Fabio is a reserve proxy open sourced by [ebay](github.com/eBay/fabio) which supports consul out of the box. This is great because it will take care to route traffic internally to the correct node which runs a specific service.

Since there are also no prebuild ARM binaries available I've also included a compiled binary for this in the repository. The fabio compilation process is pretty much the same for nomad; see above.

Now that we have an ARM binary we need to schedule fabio on every nomad node. This
can be done easily using nomads `system` type.

Please note that you have to modify the `nomad/fabio.nomad` file slightly, entering a nomad cluster ip:

```
# nomad/fabio.nomad
job "fabio" {
  datacenters = ["dc1"]

  # run on every nomad node
  type = "system"

  update {
    stagger = "5s"
    max_parallel = 1
  }

  group "fabio" {
    task "fabio" {
      driver = "raw_exec"

      config {
        command = "fabio_v1.2_linux_arm"
        # replace 10.1.42.50 with an IP from your cluster!
        args = ["-proxy.addr=:80", "-registry.consul.addr", "10.1.42.50:8500", "-ui.addr=:9999"]
      }

      artifact {
        source = "https://github.com/nicolai86/scaleway-terraform-demo/raw/master/binaries/fabio_v1.2_linux_arm"

        options {
          checksum = "md5:9cea33d5531a4948706f53b0e16283d5"
        }
      }

      resources {
        cpu = 20
        memory = 64
        network {
          mbits = 1

          port "http" {
            static = 80
          }
          port "ui" {
            static = 9999
          }
        }
      }
    }
  }
}
```

We're not running consul locally on every nomad node so we have to specify `-registry.consul.addr`. Please replace the IP `10.1.42.50` with any IP from your consul cluster. We could get around this manual manipulation e.g. by using [envconsul](https://github.com/hashicorp/envconsul) or installing consul on every nomad node and using DNS.

Anyway, let's use nomad to run fabio on every node:

```
$ nomad run -address=http://163.172.171.232:4646 nomad/fabio.nomad
==> Monitoring evaluation "7116f4b8"
    Evaluation triggered by job "fabio"
    Allocation "1ae8159c" created: node "c8c8a567", group "fabio"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "7116f4b8" finished with status "complete"
```

Fabio should have self registered with consul, let's verify that:

```
$ ssh root@163.172.157.49 'curl -s 163.172.157.49:8500/v1/catalog/services' | jq 'keys'
[
  'consul',
  'nomad',
  'nomad-client',
  'fabio'
]
```

Great! Now we need to run some applications on our fresh cluster:

## Is go 1.7 out yet?

I've taken the [is go 1.2 out yet](https://github.com/nf/go12) API and adjusted it to check for go 1.7 instead of go 1.2.

To allow fabio to pick up our app we need to include a service definition. I'll be 
using one of my domains for this, `randschau.eu`, since I want to bridge this to AWS 
Route 53 later on:

```
service {
  name = "isgo17outyet"
  tags = ["urlprefix-isgo17outyet.randschau.eu/"]
  port = "http"
  check {
    type = "http"
    name = "health"
    interval = "15s"
    timeout = "5s"
    path = "/"
  }
}
```

See the `nomad` directory for the complete `isgo17outyet.nomad` file.

```
$ nomad run -address=http://163.172.171.232:4646 -verbose nomad/isgooutyet.nomad
==> Monitoring evaluation "4c8ca49f-32a3-db17-92a0-d19f1ca8e3e8"
    Evaluation triggered by job "isgo17outyet"
    Allocation "29036d82-7cc6-6742-8db9-176fe83a7a3a" created: node "c8c8a567-a59b-b322-6428-7d6dc66af8d9", group "isgo17outyet"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "4c8ca49f-32a3-db17-92a0-d19f1ca8e3e8" finished with status "complete"
```

We can verify that our app is actually healthy by checking the consul health checks:

```
$ ssh root@163.172.157.49 'curl 163.172.157.49:8500/v1/health/checks/isgo17outyet' | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   298  100   298    0     0   2967      0 --:--:-- --:--:-- --:--:--  2980
[
  {
    "Node": "consul-1",
    "CheckID": "66d7110ca9d9d844415a26592fbb872888895021",
    "Name": "health",
    "Status": "passing",
    "Notes": "",
    "Output": "",
    "ServiceID": "_nomad-executor-29036d82-7cc6-6742-8db9-176fe83a7a3a-isgo17outyet-isgo17outyet-urlprefix-isgo17outyet.randschau.eu/",
    "ServiceName": "isgo17outyet",
    "CreateIndex": 187,
    "ModifyIndex": 188
  }
]
```

Now we need to route `isgo17outyet.randschau.eu` to any nomad node to be able to access it via dns. For this we'll be using AWS Route 53:

## DNS routing

We slightly modify our `main.tf`:

```
provider "scaleway" {}

module "security_group" {
  source = "./modules/security_group"
}

module "consul" {
  source = "./modules/consul"

  security_group = "${module.security_group.id}"
}

module "nomad" {
  source = "./modules/nomad"

  consul_cluster_ip = "${module.consul.server_ip}"
  security_group    = "${module.security_group.id}"
}

provider "aws" {
  region = "eu-west-1"
}

resource "aws_route53_record" "service" {
  zone_id = "${var.aws_hosted_zone_id}"
  name = "isgo17outyet"
  type = "A"
  ttl = "30"
  records = [
      "${split(",", module.nomad.public_ips)}"
  ]
}
```

Apply the changes with terraform:

```
$ terraform apply -var 'aws_hosted_zone_id=<my-hosted-zone-id>'
```

Once this is done we should immediatly be able to access our api via our AWS Route 53 
domain:

```
$  curl -v -H 'isgo17outyet.randschau.eu' 
*   Trying 163.172.171.232...
* Connected to 163.172.171.232 (163.172.171.232) port 9999 (#0)
> GET / HTTP/1.1
> Host: isgo17outyet.randschau.eu
> User-Agent: curl/7.43.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Length: 120
< Content-Type: text/html; charset=utf-8
< Date: Sat, 23 Jul 2016 11:46:17 GMT
<

<!DOCTYPE html><html><body><center>
  <h2>Is Go 1.7 out yet?</h2>
  <h1>

    No.

  </h1>
</center></body></html>
* Connection #0 to host 163.172.171.232 left intact
```

## closing thoughts

This was just a demo on how to use the new Scaleway provider with terraform,
as well as how to connect it with other cloud providers like AWS.

We've skipped over using `scaleway_ip`, e.g. to have IPs which can outlive specific servers. This would be a great option for a jump host.

Also if you want to persist data separate from your server I suggest giving `scaleway_volume` & `scaleway_volume_attachment` a spin.

That's it for now. Happy hacking : )
