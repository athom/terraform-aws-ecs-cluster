# terraform-aws-ecs-cluster

A Terraform module to create an Amazon Web Services (AWS) EC2 Container Service (ECS) cluster.

## Usage

This module creates a security group that gets associated with the launch configuration for the ECS cluster Auto Scaling group. By default, the security group contains no rules. In order for network traffic to flow egress or ingress (including communication with ECS itself), you must associate all of the appropriate `aws_security_group_rule` resources with the `container_instance_security_group_id` module output.

See below for an example.

```hcl
data "template_file" "container_instance_cloud_config" {
  template = "${file("cloud-config/container-instance.yml.tpl")}"

  vars {
    environment = "${var.environment}"
  }
}

module "container_service_cluster" {
  source = "github.com/azavea/terraform-aws-ecs-cluster?ref=3.0.0"

  vpc_id        = "vpc-20f74844"
  ami_id        = "ami-b2df2ca4"
  instance_type = "t2.micro"
  key_name      = "hector"
  cloud_config_content  = "${data.template_file.container_instance_cloud_config.rendered}"

  root_block_device_type = "gp2"
  root_block_device_size = "10"

  health_check_grace_period = "600"
  desired_capacity          = "1"
  min_size                  = "0"
  max_size                  = "1"

  enabled_metrics = [
    "GroupMinSize",
    "GroupMaxSize",
    "GroupDesiredCapacity",
    "GroupInServiceInstances",
    "GroupPendingInstances",
    "GroupStandbyInstances",
    "GroupTerminatingInstances",
    "GroupTotalInstances",
  ]

  subnet_ids = [...]

  project     = "Something"
  environment = "Staging"
}

resource "aws_security_group_rule" "container_instance_http_egress" {
  type        = "egress"
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]

  security_group_id = "${module.container_service_cluster.container_instance_security_group_id}"
}

resource "aws_security_group_rule" "container_instance_https_egress" {
  type        = "egress"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]

  security_group_id = "${module.container_service_cluster.container_instance_security_group_id}"
}
```

## Variables

- `vpc_id` - ID of VPC meant to house cluster
- `lookup_latest_ami` - lookup the latest Amazon-owned ECS AMI. If this variable is `true`, the latest ECS AMI will be used, even if `ami_id` is provided (default: `false`).
- `ami_id` - Cluster instance Amazon Machine Image (AMI) ID. If `lookup_latest_ami` is `true`, this variable will be silently ignored.
- `ami_owners` - List of accounts that own the AMI (default: `self, amazon, aws-marketplace`)
- `root_block_device_type` - Instance root block device type (default: `gp2`)
- `root_block_device_size` - Instance root block device size in gigabytes (default: `8`)
- `instance_type` - Instance type for cluster instances (default: `t2.micro`)
- `cpu_credit_specification` - Credit option for CPU usage. Can be ["standard"](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-standard-mode.html) or ["unlimited"](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-unlimited-mode.html). (default: `standard`).
- `detailed_monitoring` - If this variable is `true`, then [detailed monitoring](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-cloudwatch-new.html) will be enabled on the instance. (default: `false`)
- `key_name` - EC2 Key pair name
- `cloud_config_content` - user data supplied to launch configuration for cluster nodes
- `cloud_config_content_type` - the type of configuration being passed in as user data, see [EC2 user guide](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AmazonLinuxAMIBasics.html#CloudInit) for a list of possible types (default: `text/cloud-config`)
- `health_check_grace_period` - Time in seconds after container instance comes into service before checking health (default: `600`)
- `desired_capacity` - Number of EC2 instances that should be running in cluster (default: `1`)
- `min_size` - Minimum number of EC2 instances in cluster (default: `0`)
- `max_size` - Maximum number of EC2 instances in cluster (default: `1`)
- `enabled_metrics` - A list of metrics to gather for the cluster
- `subnet_ids` - A list of subnet IDs to launch cluster instances
- `cpu_reservation_target_value` - CPUReservation metric target value for [target tracking autoscaling](https://segment.com/blog/when-aws-autoscale-doesn-t/) (default: `90.0`)
- `memory_reservation_target_value` - MemoryReservation metric target value for target tracking autoscaling (default: `90.0`)
- `target_value_metric_statistic` - Statistic for target tracking autoscaling metric specification (default: `Average`)
- `project` - Name of project this cluster is for (default: `Unknown`)
- `environment` - Name of environment this cluster is targeting (default: `Unknown`)

## Outputs

- `id` - The container service cluster ID
- `name` - The container service cluster name
- `container_instance_security_group_id` - Security group ID of the EC2 container instances
- `container_instance_ecs_for_ec2_service_role_name` - Name of IAM role associated with EC2 container instances
- `ecs_service_role_name` - Name of IAM role for use with ECS services
- `ecs_service_role_arn` - ARN of IAM role for use with ECS services
- `container_instance_ecs_for_ec2_service_role_arn` - ARN of IAM role associated with EC2 container instances
- `container_instance_autoscaling_group_name` - Name of container instance Autoscaling Group
