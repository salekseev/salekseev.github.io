---
layout: post
title: "Use nested maps in Terraform variables"
date: 2018-07-11 14:04:00 -0500
comments: true
categories: [terraform, aws]
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

  default = {
    "instance-01" = {
      instance_type = "t2.micro"
    }

    "instance-02" = {
      instance_type = "t2.nano"
    }
  }
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
