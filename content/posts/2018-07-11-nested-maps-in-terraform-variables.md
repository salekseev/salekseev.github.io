---
categories:
    - Development
    - Terraform
    - AWS
date: 2018-07-11T13:42:37-04:00
slug: nested-maps-in-terraform-variables
tags:
    - terraform
    - hcl
    - json
title: Using nested maps in Terraform input variables
---

Here's my working terraform file:

```terraform
variable "create" {
  description = "Whether to the resources"
  default     = true
}

variable "config" {
  type        = "map"
  description = "Instances to create (map of maps keyed on 'name')"
}

data "template_file" "config_instance" {
  count    = "${length(var.config)}"
  template = "${element(keys(var.config), count.index)}"
}

data "template_file" "config_instance_type" {
  count    = "${length(var.config)}"
  template = "${lookup(var.config[element(keys(var.config), count.index)], "instance_type")}"
}

data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-bionic-18.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}

resource "aws_instance" "instance" {
  count = "${var.create ? length(var.config) : 0}"

  ami           = "${data.aws_ami.ubuntu.id}"
  instance_type = "${element(data.template_file.config_instance_type.*.rendered, count.index)}"

  tags {
    Name = "${element(data.template_file.config_instance.*.rendered, count.index)}"
  }
}

output "instance_id" {
  value = "${length(aws_instance.instance.*.id) > 0 ? element(concat(aws_instance.instance.*.id, list("")), 0) : ""}"
}
```

Given the following variable file:

```terraform
create = true

config = {
  "instance-01" = {
    instance_type = "t2.micro"
  }

  "instance-02" = {
    instance_type = "t2.nano"
  }
}
```

Terraform would produce the following plan:

```
$ terraform plan -var-file=test.tfvars
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.template_file.config_instance_type[0]: Refreshing state...
data.template_file.config_instance[1]: Refreshing state...
data.template_file.config_instance[0]: Refreshing state...
data.template_file.config_instance_type[1]: Refreshing state...
data.aws_ami.ubuntu: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + aws_instance.instance[0]
      id:                           <computed>
      ami:                          "ami-5cc39523"
      associate_public_ip_address:  <computed>
      availability_zone:            <computed>
      ebs_block_device.#:           <computed>
      ephemeral_block_device.#:     <computed>
      get_password_data:            "false"
      instance_state:               <computed>
      instance_type:                "t2.micro"
      ipv6_address_count:           <computed>
      ipv6_addresses.#:             <computed>
      key_name:                     <computed>
      network_interface.#:          <computed>
      network_interface_id:         <computed>
      password_data:                <computed>
      placement_group:              <computed>
      primary_network_interface_id: <computed>
      private_dns:                  <computed>
      private_ip:                   <computed>
      public_dns:                   <computed>
      public_ip:                    <computed>
      root_block_device.#:          <computed>
      security_groups.#:            <computed>
      source_dest_check:            "true"
      subnet_id:                    <computed>
      tags.%:                       "1"
      tags.Name:                    "instance-01"
      tenancy:                      <computed>
      volume_tags.%:                <computed>
      vpc_security_group_ids.#:     <computed>

  + aws_instance.instance[1]
      id:                           <computed>
      ami:                          "ami-5cc39523"
      associate_public_ip_address:  <computed>
      availability_zone:            <computed>
      ebs_block_device.#:           <computed>
      ephemeral_block_device.#:     <computed>
      get_password_data:            "false"
      instance_state:               <computed>
      instance_type:                "t2.nano"
      ipv6_address_count:           <computed>
      ipv6_addresses.#:             <computed>
      key_name:                     <computed>
      network_interface.#:          <computed>
      network_interface_id:         <computed>
      password_data:                <computed>
      placement_group:              <computed>
      primary_network_interface_id: <computed>
      private_dns:                  <computed>
      private_ip:                   <computed>
      public_dns:                   <computed>
      public_ip:                    <computed>
      root_block_device.#:          <computed>
      security_groups.#:            <computed>
      source_dest_check:            "true"
      subnet_id:                    <computed>
      tags.%:                       "1"
      tags.Name:                    "instance-02"
      tenancy:                      <computed>
      volume_tags.%:                <computed>
      vpc_security_group_ids.#:     <computed>


Plan: 2 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

Due to weirdness in HCL library if you wanted to pass input variables using JSON, you would need to wrap the nested map inside a list like so:

```json
{
  "create": true,
  "config": {
    "instance-03": [
      {
        "instance_type": "t2.micro"
      }
    ],
    "instance-04": [
      {
        "instance_type": "t2.nano"
      }
    ]
  }
}
```

Terraform plan produced:

```
$ terraform plan -var-file=test.tfvars.json
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.template_file.config_instance[1]: Refreshing state...
data.template_file.config_instance_type[1]: Refreshing state...
data.template_file.config_instance_type[0]: Refreshing state...
data.template_file.config_instance[0]: Refreshing state...
data.aws_ami.ubuntu: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + aws_instance.instance[0]
      id:                           <computed>
      ami:                          "ami-5cc39523"
      associate_public_ip_address:  <computed>
      availability_zone:            <computed>
      ebs_block_device.#:           <computed>
      ephemeral_block_device.#:     <computed>
      get_password_data:            "false"
      instance_state:               <computed>
      instance_type:                "t2.micro"
      ipv6_address_count:           <computed>
      ipv6_addresses.#:             <computed>
      key_name:                     <computed>
      network_interface.#:          <computed>
      network_interface_id:         <computed>
      password_data:                <computed>
      placement_group:              <computed>
      primary_network_interface_id: <computed>
      private_dns:                  <computed>
      private_ip:                   <computed>
      public_dns:                   <computed>
      public_ip:                    <computed>
      root_block_device.#:          <computed>
      security_groups.#:            <computed>
      source_dest_check:            "true"
      subnet_id:                    <computed>
      tags.%:                       "1"
      tags.Name:                    "instance-03"
      tenancy:                      <computed>
      volume_tags.%:                <computed>
      vpc_security_group_ids.#:     <computed>

  + aws_instance.instance[1]
      id:                           <computed>
      ami:                          "ami-5cc39523"
      associate_public_ip_address:  <computed>
      availability_zone:            <computed>
      ebs_block_device.#:           <computed>
      ephemeral_block_device.#:     <computed>
      get_password_data:            "false"
      instance_state:               <computed>
      instance_type:                "t2.nano"
      ipv6_address_count:           <computed>
      ipv6_addresses.#:             <computed>
      key_name:                     <computed>
      network_interface.#:          <computed>
      network_interface_id:         <computed>
      password_data:                <computed>
      placement_group:              <computed>
      primary_network_interface_id: <computed>
      private_dns:                  <computed>
      private_ip:                   <computed>
      public_dns:                   <computed>
      public_ip:                    <computed>
      root_block_device.#:          <computed>
      security_groups.#:            <computed>
      source_dest_check:            "true"
      subnet_id:                    <computed>
      tags.%:                       "1"
      tags.Name:                    "instance-04"
      tenancy:                      <computed>
      volume_tags.%:                <computed>
      vpc_security_group_ids.#:     <computed>


Plan: 2 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

Please note that if default map is provided for the `config` variable, it would result in concatenation of those maps!
