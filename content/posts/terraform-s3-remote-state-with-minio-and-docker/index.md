---
title: "Terraform S3 remote state with Minio and Docker"
date: "2019-02-27"
categories: 
  - "docker"
  - "provisioning"
  - "terraform"
---

# Storing Terraform's remote state in Minio

Whilst AWS's free S3 tier is almost certainly sufficient to store Terraform's remote state, it may be the case that you have a requirement to keep the data on-site, or alternatively if you're using Terraform in an air-gapped environment then you have no choice but to self-host.

Enter [Minio](https://www.minio.io/). If you've not used it before, the TLDR is that Minio provides an S3-compatible API in a single binary. This makes it perfect to store your Terraform state in.

Getting it running under Docker is also pretty simple using the official builds on the [Docker Hub](https://hub.docker.com/r/minio/minio). Below is my Ansible configuration:

```
- name: minio
  docker_container:
    name: minio
    hostname: s3.domain.com
    image: minio/minio:tag
    pull: true
    state: started
    command: "server /data"
    restart_policy: unless-stopped
    volumes:
      - "/srv/minio/config:/root/.minio"
      - "/srv/minio/data:/data"
    exposed_ports:
      - 9000
    env:
      MINIO_ACCESS_KEY: "myaccesskey"
      MINIO_SECRET_KEY: "mysupersecretkey"
```

Note that this config expects you to have a reverse proxy in front of Minio which handles the TLS termination. In my case, I use [Traefik](https://traefik.io) which also handles certificate issuance and renewals from LetsEncrypt.

With that done, log into your fresh Minio container using your access and secret keys, then create a new bucket called 'terraform' (or whatever floats your boat).

Next, we need to tell Terraform where to store its state file. In your working directory, create a `.tf` with the following details:

```
terraform {
  backend "s3" {
    endpoint = "https://s3.domain.com"
    key = "terraform.tfstate"
    region = "main"
    skip_requesting_account_id = true
    skip_credentials_validation = true
    skip_get_ec2_platforms = true
    skip_metadata_api_check = true
    skip_region_validation = true
  }
}
```

- `endpoint` sets the location of the S3 endpoint.
- `key` is the name of the remote statefile Terraform will use in the bucket.
- `region` can be set to whatever you like (usually this would take the name of an AWS region). We'll ignore this with `skip_region_validation`.
- `skip_*` variables are used to disable some of the sanity checking Terraform performs when talking to an actual AWS S3 API.

One issue to note here is that due to AWS's virtual-host style URLS, you'll need to have a valid DNS entry and certificate for `bucket.domain`, so in this case `terraform.s3.domain.com`. There is the [relatively recent addition](https://github.com/hashicorp/terraform/issues/13194) of a `force_path_style` variable, but at the time of writing the current version of Terraform (v0.11.11) doesn't support this. I expect it will be available in v0.12 and will result in a more DNS-friendly URL of `s3.domain.com/terraform`.

With the prep-work done, you now need to initialise your working directory:

```
export MINIO_ACCESS_KEY="myaccesskey"
export MINIO_SECRET_KEY="mysupersecretkey"
export BUCKET="terraform"
terraform init -backend-config="access_key=$MINIO_ACCESS_TOKEN" -backend-config="secret_key=$MINIO_SECRET_KEY" -backend-config="bucket=$BUCKET"
```

Provided all has gone well, you should see something along the lines of `Terraform has been successfully initialized!`. At this point we'll update the `.tf` file to include a `null` resource for demonstration purposes:

```
resource "null_resource" "test" {
}
```

If you then run a `terraform plan` followed by `terraform apply`, this won't actually configure any infrastructure but you should now see your remote state file in your Minio bucket!

**Note:** as an alternative to passing the S3 credentials at the time of initialising, you can provide them as environment variables - this makes the command easier to repeat and it paves the way for using your Minio-provided storage with [Atlantis](https://www.runatlantis.io/) (more on that another time).

```
export AWS_ACCESS_KEY_ID=myaccesskey
export AWS_SECRET_ACCESS_KEY=mysupersecretkey
terraform init
```
