# task13
1) VPC <br>

![image](https://github.com/user-attachments/assets/138afeff-1abc-4fff-bfdf-f733d86ad46f) <br>

MAIN.TF:
```
  
//Авл. зона
data "aws_availability_zones" "available" {}


//впс
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "${var.env}-vpc"
  }
}


//интернер
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "${var.env}-igw"
  }
}


// публ сабнет
resource "aws_subnet" "public_subnets" {
  count                   = length(var.publ_sub_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = element(var.publ_sub_cidrs, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = {
    Name = "${var.env}-public-${count.index + 1}"
  }
}


//раутинг
resource "aws_route_table" "public_subnets" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = {
    Name = "${var.env}-route-public-subnets"
  }
}

resource "aws_route_table_association" "public_routes" {
  count          = length(aws_subnet.public_subnets[*].id)
  route_table_id = aws_route_table.public_subnets.id
  subnet_id      = element(aws_subnet.public_subnets[*].id, count.index)
}



//НАТ интернет и эластичн. ип
resource "aws_eip" "nat" {
  count   = length(var.prv_sub_cidrs)
  domain = "vpc"
  tags = {
    Name = "${var.env}-nat-gw-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "nat" {
  count         = length(var.prv_sub_cidrs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = element(aws_subnet.public_subnets[*].id, count.index)
  tags = {
    Name = "${var.env}-nat-gw-${count.index + 1}"
  }
}


//прив. сабнет
resource "aws_subnet" "private_subnets" {
  count             = length(var.prv_sub_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = element(var.prv_sub_cidrs, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = {
    Name = "${var.env}-private-${count.index + 1}"
  }
}


//раутинг
resource "aws_route_table" "private_subnets" {
  count  = length(var.prv_sub_cidrs)
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.nat[count.index].id
  }
  tags = {
    Name = "${var.env}-route-private-subnet-${count.index + 1}"
  }
}

resource "aws_route_table_association" "private_routes" {
  count          = length(aws_subnet.private_subnets[*].id)
  route_table_id = aws_route_table.private_subnets[count.index].id
  subnet_id      = element(aws_subnet.private_subnets[*].id, count.index)
}
```
______________________________________________________________________

OUTPUTS.TF
```
  output "vpc_id" {
  value = aws_vpc.main.id
}

output "vpc_cidr" {
  value = aws_vpc.main.cidr_block
}

output "publ_sub_ids" {
  value = aws_subnet.public_subnets[*].id
}

output "prv_sub_ids" {
  value = aws_subnet.private_subnets[*].id
}
```
_______________________________________________________________________

VARIABLES.TF
```
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "env" {
  default = "my"
}

variable "publ_sub_cidrs" {
  default = [
    "10.0.1.0/24",
    "10.0.2.0/24"
  ]
}

variable "prv_sub_cidrs" {
  default = [
    "10.0.11.0/24",
    "10.0.22.0/24"
  ]
}
```
_______________________________________________________________________
_______________________________________________________________________

2) EC2 <br>

![image](https://github.com/user-attachments/assets/dac4a2ba-e921-4bc4-8984-fbac47bc9877) <br>

MAIN.TF:

```
  resource "aws_instance" "publ" {
  ami = var.publ_ami
  instance_type = var.publ_type
  vpc_security_group_ids = [var.sg_id]
  subnet_id = var.pub_sub
  associate_public_ip_address = true

  tags = {
    Name = "Publick"
  }
}

resource "aws_instance" "prvt" {
  ami = var.prvt_ami
  instance_type = var.prvt_type
  vpc_security_group_ids = [var.sg_id]
  subnet_id = var.prvt_sub
  associate_public_ip_address = false

  tags = {
    Name = "Private"
  }
}
```
____________________________________________________________________________

VARIABLES.TF:
```
variable "publ_ami" {
  default = "ami-0731becbf832f281e"
}


variable "prvt_ami" {
  default = "ami-0731becbf832f281e"
}


variable "publ_type" {
  default = "t2.micro"
}

variable "prvt_type" {
  default = "t2.micro"
}

variable "sg_id" {
  description = "ID of the SG"
  type        = string
}

variable "pub_sub" {
  description = "ID of PUB"
  type = string
}

variable "prvt_sub" {
  description = "ID of prvt"
  type = string
}
```
_____________________________________________________________________________________
_____________________________________________________________________________________

3) SECURITY GROUP <br>

![image](https://github.com/user-attachments/assets/b3199c9d-a146-44bc-b0bc-fc6a234fe897) <br>

MAIN.TF:
```
resource "aws_security_group" "taska1" {
  name = var.name
  vpc_id      = var.vpc_id
  dynamic "ingress" {
    for_each = [ "80", "443" ]
    content {
      from_port = ingress.value
      to_port   = ingress.value
      protocol = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
______________________________________________________________________________

OUTPUTS.TF:
```
output "sg_id" {
  value       = aws_security_group.taska1.id
}
```
____________________________________________________________________________  

VARIABLES.TF:
```
variable "name" {
    default = "for_taska"
}

variable "vpc_id" {
  description = "ID of the VPC"
  type        = string
}
```
___________________________________________________________________________________
___________________________________________________________________________________

4) RDS <br>

![image](https://github.com/user-attachments/assets/c07f2d48-978b-424b-82c7-599e34b7c8c3) <br>

MAIN.TF 
```
  resource "aws_db_subnet_group" "db_subnet_group" {
  name       = "rds-subnet-group"
  subnet_ids = var.db_subnet_ids

  tags = {
    Name = "RDS subnet group"
  }
}

resource "aws_db_instance" "rds" {
  allocated_storage      = 20
  engine                 = "postgres"
  engine_version         = "15"
  instance_class         = "db.t3.micro"
  username               = "terraform"
  password               = "123456789"
  multi_az               = true
  db_subnet_group_name   = aws_db_subnet_group.db_subnet_group.name
  vpc_security_group_ids = var.vpc_security_group_ids
  skip_final_snapshot    = true

  tags = {
    Name = "Basa"
  }


}


```
___________________________________________________________________________

VARIABLES.TF
```
variable "db_subnet_ids" {
  type = list(string)
}

variable "vpc_security_group_ids" {
  type = list(string)
}

```
___________________________________________________________________________
___________________________________________________________________________

5) MAIN.TF
```
      provider "aws" {
    access_key = "???????????????????????????"
    secret_key = "??????????????????????"
    region = "us-east-1"
}

module "vpc-1" {
  source = "../vpc"
}

module "sg-1" {
  source = "../sg"
  vpc_id  = module.vpc-1.vpc_id
}

module "inst-1" {
  source = "../instance"
  sg_id = module.sg-1.sg_id
  pub_sub = module.vpc-1.publ_sub_ids[0]
  prvt_sub = module.vpc-1.prv_sub_ids[0]
}

module "rds-1" {
  source = "../rds"
  vpc_security_group_ids = [module.sg-1.sg_id]
  db_subnet_ids = module.vpc-1.prv_sub_ids
}

```
_________________________________________________________________________
_________________________________________________________________________

ИТОГ: <br>
VPC: <br>
![image](https://github.com/user-attachments/assets/10225d57-8328-47f6-98c8-009dd5e3a550) <br>

![image](https://github.com/user-attachments/assets/24877f21-a3ae-4dbf-ad22-282c25495dde) <br>

![image](https://github.com/user-attachments/assets/25db7a77-a012-4df0-a3a6-b0a60a67b75b) <br>
![image](https://github.com/user-attachments/assets/9d9d1d75-76d8-4765-af52-de436438db85)
![image](https://github.com/user-attachments/assets/af77789d-ae55-4d26-ab2a-8d1f513ceef4)

![image](https://github.com/user-attachments/assets/19d0a816-19e3-4269-a6c2-a0ed79ec1177) <br>
![image](https://github.com/user-attachments/assets/079d3154-fd2c-4458-b1eb-c2931d5e11a6) <br>


EC2: <br>
![image](https://github.com/user-attachments/assets/87a6c292-619c-40df-b705-145c70b5ecac) <br>

SECURITY GROUP: <br>
![image](https://github.com/user-attachments/assets/ab358edb-0889-4111-bae0-a5af1cb05a6a) <br>


RDS: <br>
![image](https://github.com/user-attachments/assets/12919962-98af-47a2-b54f-8859b5838081) <br>

![image](https://github.com/user-attachments/assets/ac305566-a124-4c4b-8027-7cdc47b50360)

