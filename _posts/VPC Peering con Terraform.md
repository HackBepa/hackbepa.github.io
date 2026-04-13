
---
* Tags: #aws #terraform #vpc #peering #networking
---
# Overview

**Objetivo**: Conectar dos VPCs privadas en AWS mediante VPC Peering desplegado con Terraform, y verificar conectividad entre las instancias EC2 usando EC2 Instance Connect Endpoints.

**Entorno**: AWS (eu-central-1) + Terraform + EC2 Instance Connect

---

## Conceptos previos

### VPC Peering
Conexión de red directa entre dos VPCs que permite que el tráfico fluya entre ellas usando IPs privadas.

> [!INFO]
> El peering **no es transitivo**. Si A hace peering con B, y B hace peering con C, A **no puede** hablar con C a través de B. Habría que crear un peering directo A-C.

> [!WARNING]
> No puede haber solapamiento de CIDRs entre las VPCs que quieres conectar. Si ambas usan `10.0.0.0/16` el peering no es posible.
> 
> **Calculadora de Subnets:**  https://visualsubnetcalc.com/

### Subnets: pública vs privada

| Tipo        | Característica                                                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Pública** | Tiene una **Internet Gateway** asociada. Las instancias pueden tener IP pública y salir/entrar directamente a internet.                |
| **Privada** | Sin Internet Gateway. El tráfico de salida a internet pasa por un **NAT Gateway** (si existe). No recibe tráfico entrante de internet. |

### Internet Gateway vs NAT Gateway

| | Internet Gateway | NAT Gateway |
|---|---|---|
| **Dirección** | Bidireccional (entrada y salida) | Solo salida (outbound) |
| **Uso** | Subnets públicas | Subnets privadas que necesitan salir a internet |
| **IP pública** | La instancia necesita IP pública | La instancia usa IP privada, el NAT tiene la pública |

### EC2 Instance Connect Endpoint (EICE)
Permite conectarse por SSH a instancias en subnets **privadas** sin necesidad de un bastion host ni IP pública. AWS crea un endpoint en la subnet que actúa como túnel SSH.

---

## Arquitectura del lab

![[Lab - VPC Peering con Terraform.png]]

Ambas subnets son **privadas** (sin Internet Gateway). El acceso SSH se realiza a través de los EICE.

---

## Recursos Terraform desplegados

### 1. VPCs

Dos VPCs con CIDRs que no se solapan:

```hcl
resource "aws_vpc" "vpc_a" {
  cidr_block = "10.0.0.0/17"
}

resource "aws_vpc" "vpc_b" {
  cidr_block = "10.0.128.0/17"
}
```

### 2. Subnets

Una subnet privada por VPC, en la misma AZ (`eu-central-1a`):

```hcl
resource "aws_subnet" "subnet_a" {
  vpc_id     = aws_vpc.vpc_a.id
  cidr_block = "10.0.0.0/17"
}

resource "aws_subnet" "subnet_b" {
  vpc_id     = aws_vpc.vpc_b.id
  cidr_block = "10.0.128.0/17"
}
```

### 3. Security Groups

Cada EC2 permite:
- **ICMP entrante** desde el CIDR de la VPC opuesta (para hacer ping entre instancias)
- **SSH entrante** desde cualquier IP (el EICE lo necesita)
- **Todo el tráfico saliente**

Los EICE tienen su propio SG que solo permite **SSH saliente** dentro de su VPC.

### 4. EC2 Instances

Dos instancias `t3.micro` con Amazon Linux 2023, una en cada subnet:

```hcl
resource "aws_instance" "ec2_a" {
  ami           = "ami-023adbbb2c440f837"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.subnet_a.id
}
```

### 5. VPC Peering

```hcl
resource "aws_vpc_peering_connection" "peering" {
  vpc_id      = aws_vpc.vpc_a.id
  peer_vpc_id = aws_vpc.vpc_b.id
  auto_accept = true   # Solo funciona si ambas VPCs están en la misma cuenta
}
```

### 6. Route Tables

Para que el tráfico fluya por el peering, cada VPC necesita una ruta que apunte al CIDR de la otra:

```hcl
# VPC A → conoce la red de VPC B por el peering
resource "aws_route_table" "rt_a" {
  vpc_id = aws_vpc.vpc_a.id

  route {
    cidr_block                = "10.0.128.0/17"
    vpc_peering_connection_id = aws_vpc_peering_connection.peering.id
  }
}

# VPC B → conoce la red de VPC A por el peering
resource "aws_route_table" "rt_b" {
  vpc_id = aws_vpc.vpc_b.id

  route {
    cidr_block                = "10.0.0.0/17"
    vpc_peering_connection_id = aws_vpc_peering_connection.peering.id
  }
}
```

> [!WARNING]
> Crear el peering **no es suficiente**. Sin las route tables el tráfico no sabe cómo llegar a la otra VPC.

### 7. EC2 Instance Connect Endpoints

```hcl
resource "aws_ec2_instance_connect_endpoint" "eice_a" {
  subnet_id          = aws_subnet.subnet_a_axel.id
  security_group_ids = [aws_security_group.sg_eice_a.id]
}
```

---

## Despliegue

```bash
cd lab-peering

terraform init
terraform validate
terraform plan
terraform apply
```

---

## Verificación

### Conectarse a las instancias

Desde la consola AWS → EC2 → seleccionar instancia → Connect → EC2 Instance Connect.

### Probar conectividad entre VPCs

Desde `ec2-a`, hacer ping a la IP privada de `ec2-b`:

```bash
ping 10.0.128.x
```

Si hay respuesta, el peering y las route tables están bien configurados.

---

## Destruir el lab

```bash
terraform destroy
```

---

## Conclusiones

Este lab me ha gustado mucho ya que he entendido lo que es un peering en AWS y he ampliado horizontes con Terraform.

**¿Qué hemos practicado?**
- Crear VPCs con subnets privadas
- Configurar VPC Peering entre dos redes sin solapamiento de CIDRs
- Entender que el peering requiere route tables en **ambas** VPCs
- Acceder a instancias privadas sin bastion usando EICE
- Desplegar toda la infraestructura como código con Terraform

**Puntos clave a recordar:**
- El peering no es transitivo
- Sin route tables el peering no funciona aunque esté activo
- `auto_accept = true` solo funciona si las dos VPCs son de la misma cuenta AWS
