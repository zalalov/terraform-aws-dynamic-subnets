# terraform-aws-dynamic-subnets [![Build Status](https://travis-ci.org/cloudposse/terraform-aws-dynamic-subnets.svg)](https://travis-ci.org/cloudposse/terraform-aws-dynamic-subnets)

Terraform module for public and private [`subnets`](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html) provisioning in existing AWS [`VPC`](https://aws.amazon.com/vpc)*[]:

The module creates one private and one public subnet in each Availability Zone specified in `var.availability_zones`.
The public subnets are routed to the Internet Gateway (using the provided Internet Gateway ID).
NAT Gateways are created in each AZ, and the private subnets are routed to the corresponding NAT Gateways (this could be disabled by `nat_gateway_enabled` flag).



## Usage

Note: this module is intended for use with existing VPC and existing Internet Gateway.
You should use [terraform-aws-vpc](https://github.com/cloudposse/terraform-aws-vpc) module if you plan to create a new VPC.

```hcl
module "subnets" {
  source = "git::https://github.com/cloudposse/terraform-aws-dynamic-subnets.git?ref=master"

  availability_zones         = "${var.availability_zones}"
  namespace                  = "${var.namespace}"
  name                       = "${var.name}"
  stage                      = "${var.stage}"
  region                     = "${var.region}"
  vpc_id                     = "${var.vpc_id}"
  igw_id                     = "${var.igw_id}"
  cidr_block                 = "${var.cidr_block}"
  vpc_default_route_table_id = "${var.vpc_default_route_table_id}"
  public_network_acl_id      = "${var.public_network_acl_id}"
  private_network_acl_id     = "${var.private_network_acl_id}"
}
```


## Variables

|  Name                        |  Default       |  Description                                                                                                                         | Required |
|:-----------------------------|:--------------:|:-------------------------------------------------------------------------------------------------------------------------------------|:--------:|
| namespace                    | ``             | Namespace (e.g. `cp` or `cloudposse`)                                                                                                | Yes      |
| stage                        | ``             | Stage (e.g. `prod`, `dev`, `staging`)                                                                                                | Yes      |
| name                         | ``             | Name  (e.g. `bastion` or `db`)                                                                                                       | Yes      |
| tags                         | ``             | Additional tags (e.g. `Key, Value`)                                                                                                  | No       |
| region                       | ``             | AWS Region where module should operate (e.g. `us-east-1`)                                                                            | Yes      |
| vpc_id                       | ``             | The VPC ID where subnets will be created (e.g. `vpc-aceb2723`)                                                                       | Yes      |
| cidr_block                   | ``             | The base CIDR block which will be divided into subnet CIDR blocks (e.g. `10.0.0.0/16`)                                               | Yes      |
| igw_id                       | ``             | The Internet Gateway ID public route table will point to (e.g. `igw-9c26a123`)                                                       | Yes      |
| vpc_default_route_table_id   | ``             | The default route table for public subnets. Provides access to the Internet. If not set here, will be created. (e.g. `rtb-f4f0ce12`) | No       |
| availability_zones           | []             | The list of Availability Zones where subnets will be created (e.g. `["us-eas-1a", "us-eas-1b"]`)                                     | Yes      |
| public_network_acl_id        | ``             | Network ACL ID that will be added to public subnets.  If empty, a new ACL will be created                                            | No       |
| private_network_acl_id       | ``             | Network ACL ID that will be added to private subnets.  If empty, a new ACL will be created                                           | No       |
| nat_gateway_enabled          | `true`         | Flag to enable/disable NAT gateways                                                                                                  | No       |


## TL;DR

`terraform-aws-dynamic-subnets` creates a set of subnets based on `${var.cidr_block}` input
and the number of Availability Zones in the region.

For subnet set calculation `terraform-aws-dynamic-subnets` uses TF
[cidrsubnet](https://www.terraform.io/docs/configuration/interpolation.html#cidrsubnet-iprange-newbits-netnum-)
interpolation.

### Calculation logic

```hcl
${
  cidrsubnet(
  signum(length(var.cidr_block)) == 1 ?
  var.cidr_block : data.aws_vpc.default.cidr_block,
  ceil(log(length(data.aws_availability_zones.available.names) * 2, 2)),
  count.index)
}
```


1. Use `${var.cidr_block}` input (if specified) or
   use the VPC CIDR block `data.aws_vpc.default.cidr_block` (e.g. `10.0.0.0/16`)
2. Get number of available AZs in the region (e.g. `length(data.aws_availability_zones.available.names)`)
3. Calculate `newbits`. `newbits` number specifies how many subnets will
   the CIDR block (input or VPC) be divided into. `newbits` is the number of `binary digits`.

    Example:

    `newbits = 1` - 2 subnets are available (`1 binary digit` allows to count up to `2`)

    `newbits = 2` - 4 subnets are available (`2 binary digits` allows to count up to `4`)

    `newbits = 3` - 8 subnets are available (`3 binary digits` allows to count up to `8`)


    etc.


    1. We have `6` AZs in a `us-east-1` region (see step 2).
    2. We need to create `1 public` subnet and `1 private` subnet in each AZ,
       thus we need to create `12 subnets` in total (`6` AZs * (`1 public` + `1 private`)).
    3. We need `4 binary digits` for that ( 2<sup>4</sup> = 16 ).
       In order to calculate the number of `binary digits` we use `logarithm`
       function. We use `base 2` logarithm because decimal numbers
       can be calculated as `powers` of binary number.
       See [Wiki](https://en.wikipedia.org/wiki/Binary_number#Decimal)
       for more details

       Example:

       For `12 subnets` we need `3.58` `binary digits` (log<sub>2</sub>12)

       For `16 subnets` we need `4` `binary digits` (log<sub>2</sub>16)

       For `7 subnets` we need `2.81` `binary digits` (log<sub>2</sub>7)

       etc.

    4. We can't have `binary digits` as fractional values.
       We can't round it down because smaller `binary digits` number is insufficient.
       Thus we should round it up. See TF [ceil](https://www.terraform.io/docs/configuration/interpolation.html#ceil-float-).

       Example:

       For `12 subnets` we need `4` `binary digits` (ceil(log<sub>2</sub>12))

       For `16 subnets` we need `4` `binary digits` (ceil(log<sub>2</sub>16))

       For `7 subnets` we need `3` `binary digits` (ceil(log<sub>2</sub>7))

       etc.

    5. Assign private subnets according to AZ number (we're using `count.index`).
    6. Assign public subnets according to AZ number but with a shift.
       We calculate the shift number as `length(data.aws_availability_zones.available.names) + count.index`,
       which depends on the number of AZs in the region (see step 2)


## License

Apache 2 License. See [`LICENSE`](LICENSE) for full details.
