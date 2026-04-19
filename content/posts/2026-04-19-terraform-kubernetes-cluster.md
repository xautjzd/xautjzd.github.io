---
title: 基于 Terraform 构建企业内部 kubernetes 集群
date: "2026-04-19T10:00:00+08:00"
tags: ["infrastructure", "devops", "kubernetes", "terraform"]
---

## 1. 为什么要构建企业内部 Kubernetes 集群？

随着企业业务规模的不断扩张，微服务架构逐渐成为主流，应用的数量、版本迭代的频率以及部署的复杂度都在急剧上升。在这一背景下，建立一套稳定、高效、可扩展的企业内部 Kubernetes 集群，成为支撑企业数字化转型的核心基础设施之一。

### 1.1 构建统一的资源调度平台

企业内部往往同时运行着多条业务线、多个团队的应用服务。Kubernetes 提供了统一的容器编排与资源调度能力，能够将计算、存储、网络资源进行统一抽象和调度，避免各团队烟囱式的资源使用模式，显著提升整体资源利用率。

### 1.2 支撑 CI/CD 流水线落地

Kubernetes 是现代 CI/CD 流水线的天然载体。通过与 GitLab CI、Jenkins、ArgoCD、Tekton 等工具集成，企业可以实现：

- **持续集成（CI）**：代码提交后自动触发构建、测试，产出镜像并推送到私有镜像仓库。
- **持续交付（CD）**：基于 GitOps 理念，通过声明式配置驱动应用在集群中的自动发布与回滚。
- **多环境隔离**：通过 Namespace、NetworkPolicy、RBAC 等机制，在同一集群内划分开发、测试、预发、生产等多套环境，降低环境管理成本。

### 1.3 提升业务稳定性与弹性

Kubernetes 内置的自愈能力（Pod 自动重启、节点故障迁移）、水平自动伸缩（HPA/VPA/KEDA）以及滚动更新策略，能够大幅提升业务服务的可用性和弹性，满足企业对 SLA 的严格要求。

### 1.4 降低运维复杂度

统一的控制平面使运维团队可以通过标准化的 `kubectl` 命令或声明式 YAML 配置管理所有工作负载，结合 Helm Chart 进行应用打包与版本管理，有效降低了跨团队、跨项目的运维协作成本。

## 2. 构建内部 K8s 集群核心流程

### 2.1 整体流程概览

```
基础资源准备 → 操作系统初始化 → 集群安装部署 → 网络插件配置 → 存储插件配置 → 集群验证 → 监控告警部署
```

### 2.2 基础资源准备

构建 K8s 集群首先需要准备充足的计算、存储和网络资源。

**计算资源方案对比：**

| 方案 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| 裸金属服务器 | 性能最优、成本可控 | 运维复杂、交付周期长 | 大规模、对性能极致要求 |
| 云服务器（VM） | 弹性伸缩、快速交付 | 存在虚拟化开销 | 中小规模、快速上线 |
| 混合云 | 灵活兼顾 | 网络打通复杂 | 已有 IDC 并需弹性扩展 |

**主流云厂商选型：**

- **AWS**：EC2 + VPC + EBS，生态最完善，全球覆盖最广
- **阿里云**：ECS + VPC + 云盘，国内合规友好
- **Azure**：VM + VNet，微软生态集成强
- **GCP**：GCE + VPC，网络性能优秀

### 2.3 基础资源准备方案

企业通常有两种主流方式来完成基础资源的创建和初始化：

**方案一：脚本方式（Shell/Ansible）**

通过 Shell 脚本或 Ansible Playbook 调用云厂商 CLI/SDK 创建资源，再通过 SSH 批量初始化操作系统配置。这种方式上手快，但缺乏状态管理，在资源规模扩大后维护成本急剧上升。

**方案二：Terraform（推荐）**

通过声明式 HCL 语言描述所需的基础设施资源，Terraform 负责将期望状态与实际状态对齐。具备完整的状态管理、依赖解析和变更预览能力，是生产环境的首选方案。

## 3. 为什么采用 Terraform？

### 3.1 Terraform 核心优势

**① 声明式基础设施即代码（IaC）**

Terraform 采用 HashiCorp Configuration Language（HCL），用声明式语法描述基础设施的目标状态，而非一步步的操作指令。工程师只需关心"我要什么资源"，而非"如何创建资源"，大幅降低了认知负担。

**② 统一的多云支持**

Terraform 通过 Provider 机制支持 600+ 云服务商和 SaaS 平台（AWS、阿里云、Azure、GCP、Kubernetes、Helm 等），企业无需为每个云厂商学习不同的 SDK 或 CLI，实现真正的多云统一管理。

**③ 状态管理与变更预览**

Terraform 通过 `.tfstate` 文件追踪已部署资源的实际状态。每次变更前执行 `terraform plan` 可以清晰预览变更内容，避免误操作。结合远端 State Backend（S3 + DynamoDB），团队可以安全地协作管理同一套基础设施。

**④ 模块化与复用**

Terraform Module 机制支持将常用的资源组合（如 VPC 模块、EKS 节点组模块）封装为可复用的模块，在不同项目、不同环境间复用，保持一致性。

**⑤ 幂等性与自愈能力**

多次执行 `terraform apply` 不会产生副作用，Terraform 会自动识别已存在的资源并跳过创建，确保基础设施始终与代码描述一致。

### 3.2 与其他方案的对比

| 维度 | Terraform | Shell 脚本 | Ansible | CloudFormation（AWS 专属） |
|------|-----------|------------|---------|---------------------------|
| 多云支持 | ✅ 原生支持 | ⚠️ 需自行适配 | ⚠️ 部分支持 | ❌ 仅 AWS |
| 状态管理 | ✅ 内置 | ❌ 无 | ❌ 无 | ✅ 内置 |
| 变更预览 | ✅ plan | ❌ 无 | ⚠️ 有限 | ✅ ChangeSet |
| 学习曲线 | 中等 | 低 | 低 | 中等 |
| 模块复用 | ✅ 完善 | ❌ 弱 | ✅ Role | ⚠️ 一般 |
| 社区生态 | ✅ 非常活跃 | — | ✅ 活跃 | ⚠️ 局限 AWS |
| 幂等性 | ✅ 强 | ❌ 弱 | ✅ 较强 | ✅ 较强 |

### 3.3 Terraform 的局限性

- **状态文件风险**：`.tfstate` 包含敏感信息，需妥善保管，并启用加密和访问控制。
- **学习成本**：HCL 语法和 Provider 文档有一定学习曲线。
- **漂移检测**：若有人手动修改了云资源，Terraform 需要通过 `terraform refresh` 或 `terraform plan` 才能感知漂移。

尽管存在上述局限，Terraform 在企业级基础设施管理场景中的综合能力仍然是目前最优的选择。

## 4. 基于 Terraform 构建 Kubernetes 集群方案

### 4.1 主流方案概览

基于 Terraform 构建 K8s 集群主要有以下几种思路：

**① 托管 K8s 服务（EKS / AKS / GKE）**

直接通过 Terraform 的云厂商 Provider 创建托管集群，控制平面由云厂商负责运维。优点是运维简单、升级便捷；缺点是定制化空间有限，且存在厂商锁定风险。适合对运维能力要求不高的中小团队。

**② Kubespray（Ansible + Terraform）**

Kubespray 是 Kubernetes 官方维护的基于 Ansible 的部署工具。结合 Terraform 完成基础设施准备后，通过 Ansible Playbook 部署集群。支持高度定制化，但部署流程较复杂，调试成本高，升级路径也较为繁琐。

**③ RKE2（Rancher Kubernetes Engine 2）**

RKE2 是 Rancher 推出的生产级 Kubernetes 发行版，专注于安全性与合规性（通过 CIS Benchmark），内置 containerd、CoreDNS、Ingress 等核心组件。结合 Terraform 可以通过 `remote-exec` 或 `cloud-init` 方式在节点上自动化完成 RKE2 的安装和集群初始化，整体流程简洁、可靠。

### 4.2 方案对比

| 维度 | 托管 K8s（EKS 等） | Kubespray | RKE2 |
|------|-------------------|-----------|-------|
| 控制平面自管 | ❌ | ✅ | ✅ |
| 部署复杂度 | 低 | 高 | 中 |
| 定制化能力 | 低 | 高 | 高 |
| 安全合规（CIS） | 需额外配置 | 需额外配置 | ✅ 内置 |
| 升级便捷性 | 高 | 低 | 中 |
| 厂商锁定 | 有 | 无 | 无 |
| 社区活跃度 | — | 高 | 高 |
| 生产可靠性 | 高 | 中 | 高 |

### 4.3 生产级别推荐：RKE2

对于需要完整自主掌控控制平面、追求安全合规、同时希望保持部署流程相对简洁的企业，**RKE2 是生产环境的首选方案**，原因如下：

- **开箱即用的安全加固**：默认通过 CIS Kubernetes Benchmark Level 1/2 审计，满足企业合规要求。
- **内置关键组件**：containerd、CNI 占位符、CoreDNS、Ingress-nginx、metrics-server 等均已内置，无需额外安装。
- **简洁的安装方式**：通过单一安装脚本 + 配置文件即可完成 Server/Agent 节点的初始化，易于与 Terraform `remote-exec` 或 `cloud-init` 集成。
- **活跃的商业支持**：背靠 SUSE/Rancher，有完善的商业支持和长期维护承诺。

## 5. Kubernetes 集群网络选型

CNI（Container Network Interface）插件是 K8s 集群网络的核心组件，负责 Pod 间通信、网络策略执行等关键功能。目前企业级最主流的两款 CNI 插件是 **Calico** 和 **Cilium**。

### 5.1 Calico 与 Cilium 的对比分析

**Calico**

Calico 是目前生产环境中使用最广泛的 CNI 插件之一，由 Tigera 公司主导开发。

优势：
- 成熟稳定，社区案例丰富，生产验证充分
- 支持 BGP 原生路由模式，网络性能优秀，无需 Overlay 封装
- NetworkPolicy 实现完善，支持 Kubernetes 标准及扩展的 GlobalNetworkPolicy
- 与 AWS VPC、阿里云 VPC 等主流云环境集成良好
- 运维调试工具链完善（`calicoctl`、Felix 日志等）
- 资源消耗相对较低，对节点内核版本要求宽松

劣势：
- 高级可观测性能力（流量追踪、L7 策略）依赖商业版 Calico Enterprise
- eBPF 支持为可选模式，默认仍基于 iptables，在超大规模集群中 iptables 规则数量庞大

**Cilium**

Cilium 是基于 eBPF 技术构建的新一代 CNI 插件，由 Isovalent 主导开发。

优势：
- 原生基于 eBPF，绕过 iptables，在高并发场景下网络性能更优
- 内置 Hubble 可观测性平台，提供丰富的 L3/L4/L7 流量可视化
- 原生支持 Service Mesh 能力（Cilium Service Mesh），可替代部分 Sidecar 代理
- 对 NetworkPolicy 的支持更加丰富（DNS-based、FQDN policy 等）

劣势：
- 对 Linux 内核版本要求较高（推荐 5.10+），在一些老旧 OS 环境中兼容性存在风险
- 生产落地案例相对 Calico 少，运维人员学习曲线较陡
- 架构相对复杂，故障排查难度更高

### 5.2 选用 Calico 的原因

在本次架构中，选择 **Calico** 作为集群 CNI 插件，主要基于以下考量：

1. **生产成熟度高**：Calico 在业界有大量超过 1000 节点规模集群的生产验证案例，稳定性有充分保证。
2. **AWS VPC 集成友好**：Calico 的 VPC Native Routing 模式可以直接利用 AWS VPC 路由表进行 Pod 路由，无需 Overlay 封装，网络性能接近原生，且便于与 AWS 安全组联动。
3. **运维友好**：团队现有运维人员对 Calico 的 iptables 模式更为熟悉，故障定位路径清晰，降低了生产运维风险。
4. **50 节点规模适配**：在 50 节点规模下，iptables 规则数量尚在可控范围内，Calico 的性能表现完全满足需求，无需引入 Cilium eBPF 的额外复杂度。
5. **开源免费**：Calico 开源版本功能已能满足企业 NetworkPolicy 的核心需求，无需额外采购商业授权。

## 6. 基于 Terraform + RKE2 + Calico 构建 K8s 集群

### 6.1 架构设计概览

关键技术选型：

- **云平台**：AWS
- **节点规模**：50 个工作节点 + 3 个控制平面节点（高可用）
- **网络方案**：AWS VPC（自定义）
- **集群发行版**：RKE2 v1.30+
- **CNI 插件**：Calico（替换 RKE2 内置 Canal）
- **操作系统**：Ubuntu 24.04 LTS

**集群节点规划：**

| 节点类型 | 数量 | 实例规格 | 用途 |
|----------|------|----------|------|
| Control Plane | 3 | m5.xlarge（4C 16G） | API Server、etcd、Scheduler、Controller Manager |
| Worker Node | 50 | m5.2xlarge（8C 32G） | 业务工作负载 |
| Bastion Host | 1 | t3.micro | 运维跳板机 |

### 6.2 目录结构

```
terraform-rke2-aws/
├── main.tf                  # 根模块入口
├── variables.tf             # 变量定义
├── outputs.tf               # 输出定义
├── terraform.tfvars         # 变量赋值
├── versions.tf              # Provider 版本锁定
├── modules/
│   ├── vpc/                 # VPC 及网络资源
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── security_groups/     # 安全组规则
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── iam/                 # IAM 角色与实例 Profile
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── control_plane/       # 控制平面节点
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── worker_nodes/        # 工作节点（ASG）
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── scripts/
│   ├── rke2-server-init.sh  # 首个 Server 节点初始化脚本
│   ├── rke2-server-join.sh  # 追加 Server 节点加入脚本
│   └── rke2-agent.sh        # Worker 节点加入脚本
└── calico/
    └── custom-resources.yaml # Calico 安装配置
```

### 6.3 Step 1：配置 Terraform Provider 与 Backend

**versions.tf**

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }

  backend "s3" {
    bucket         = "your-company-terraform-state"
    key            = "rke2-cluster/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "rke2-k8s-cluster"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

### 6.4 Step 2：构建 VPC 网络基础设施

**modules/vpc/main.tf**

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr          # 10.0.0.0/16
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.cluster_name}-vpc"
  }
}

# 互联网网关
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.cluster_name}-igw"
  }
}

# 公有子网（Bastion / NLB）
resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "${var.cluster_name}-public-${count.index + 1}"
    "kubernetes.io/role/elb" = "1"
  }
}

# 私有子网（Control Plane / Worker Nodes）
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.cluster_name}-private-${count.index + 1}"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

# NAT 网关（每个 AZ 一个，生产高可用要求）
resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.cluster_name}-nat-${count.index + 1}"
  }
}

# 私有子网路由表
resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.cluster_name}-private-rt-${count.index + 1}"
  }
}

resource "aws_route_table_association" "private" {
  count          = length(var.availability_zones)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

### 6.5 Step 3：配置安全组

**modules/security_groups/main.tf**

```hcl
# Control Plane 安全组
resource "aws_security_group" "control_plane" {
  name        = "${var.cluster_name}-control-plane-sg"
  description = "RKE2 Control Plane Security Group"
  vpc_id      = var.vpc_id

  # RKE2 API Server
  ingress {
    from_port   = 6443
    to_port     = 6443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "Kubernetes API Server"
  }

  # RKE2 Supervisor API
  ingress {
    from_port   = 9345
    to_port     = 9345
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "RKE2 Supervisor API"
  }

  # etcd
  ingress {
    from_port       = 2379
    to_port         = 2380
    protocol        = "tcp"
    self            = true
    description     = "etcd client and peer"
  }

  # Calico BGP
  ingress {
    from_port   = 179
    to_port     = 179
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "Calico BGP"
  }

  # Calico VXLAN
  ingress {
    from_port   = 4789
    to_port     = 4789
    protocol    = "udp"
    cidr_blocks = [var.vpc_cidr]
    description = "Calico VXLAN"
  }

  # Kubelet
  ingress {
    from_port   = 10250
    to_port     = 10250
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "Kubelet"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.cluster_name}-control-plane-sg"
  }
}

# Worker Node 安全组
resource "aws_security_group" "worker" {
  name        = "${var.cluster_name}-worker-sg"
  description = "RKE2 Worker Node Security Group"
  vpc_id      = var.vpc_id

  # NodePort 范围
  ingress {
    from_port   = 30000
    to_port     = 32767
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "NodePort Services"
  }

  # Kubelet
  ingress {
    from_port   = 10250
    to_port     = 10250
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "Kubelet"
  }

  # Calico BGP & VXLAN
  ingress {
    from_port   = 179
    to_port     = 179
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  ingress {
    from_port   = 4789
    to_port     = 4789
    protocol    = "udp"
    cidr_blocks = [var.vpc_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.cluster_name}-worker-sg"
  }
}
```

### 6.6 Step 4：IAM 角色配置

**modules/iam/main.tf**

```hcl
# Control Plane IAM Role
resource "aws_iam_role" "control_plane" {
  name = "${var.cluster_name}-control-plane-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "control_plane" {
  name = "${var.cluster_name}-control-plane-policy"
  role = aws_iam_role.control_plane.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:DescribeInstances",
          "ec2:DescribeRegions",
          "ec2:DescribeRouteTables",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeSubnets",
          "ec2:DescribeVolumes",
          "ec2:DescribeVpcs",
          "ec2:CreateSecurityGroup",
          "ec2:CreateTags",
          "ec2:CreateVolume",
          "ec2:ModifyInstanceAttribute",
          "ec2:AttachVolume",
          "ec2:DetachVolume",
          "elasticloadbalancing:*",
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:GetRepositoryPolicy",
          "ecr:DescribeRepositories",
          "ecr:ListImages",
          "ecr:BatchGetImage",
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_instance_profile" "control_plane" {
  name = "${var.cluster_name}-control-plane-profile"
  role = aws_iam_role.control_plane.name
}

# Worker Node IAM Role（权限收窄）
resource "aws_iam_role" "worker" {
  name = "${var.cluster_name}-worker-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "worker" {
  name = "${var.cluster_name}-worker-policy"
  role = aws_iam_role.worker.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:DescribeInstances",
          "ec2:DescribeRegions",
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:GetRepositoryPolicy",
          "ecr:DescribeRepositories",
          "ecr:ListImages",
          "ecr:BatchGetImage"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_instance_profile" "worker" {
  name = "${var.cluster_name}-worker-profile"
  role = aws_iam_role.worker.name
}
```

### 6.7 Step 5：部署控制平面节点

**modules/control_plane/main.tf**

```hcl
# 生成集群 Token
resource "random_password" "rke2_token" {
  length  = 64
  special = false
}

# 将 Token 存入 SSM Parameter Store
resource "aws_ssm_parameter" "rke2_token" {
  name  = "/${var.cluster_name}/rke2-token"
  type  = "SecureString"
  value = random_password.rke2_token.result
}

# 首个 Server 节点（Init 节点）
resource "aws_instance" "control_plane_init" {
  ami                    = var.ami_id
  instance_type          = "m5.xlarge"
  subnet_id              = var.private_subnet_ids[0]
  vpc_security_group_ids = [var.control_plane_sg_id]
  iam_instance_profile   = var.control_plane_instance_profile
  key_name               = var.key_pair_name

  root_block_device {
    volume_size = 100
    volume_type = "gp3"
    encrypted   = true
  }

  user_data = base64encode(templatefile("${path.module}/../../scripts/rke2-server-init.sh", {
    rke2_version     = var.rke2_version
    cluster_name     = var.cluster_name
    rke2_token       = random_password.rke2_token.result
    aws_region       = var.aws_region
    pod_cidr         = var.pod_cidr
    service_cidr     = var.service_cidr
  }))

  tags = {
    Name = "${var.cluster_name}-control-plane-init"
    "kubernetes.io/cluster/${var.cluster_name}" = "owned"
    "rke2-node-role" = "server"
  }
}

# NLB（API Server 高可用入口）
resource "aws_lb" "api_server" {
  name               = "${var.cluster_name}-api-nlb"
  internal           = true
  load_balancer_type = "network"
  subnets            = var.private_subnet_ids
}

resource "aws_lb_target_group" "api_server" {
  name     = "${var.cluster_name}-api-tg"
  port     = 6443
  protocol = "TCP"
  vpc_id   = var.vpc_id

  health_check {
    protocol            = "HTTPS"
    path                = "/healthz"
    port                = "6443"
    healthy_threshold   = 3
    unhealthy_threshold = 3
  }
}

resource "aws_lb_listener" "api_server" {
  load_balancer_arn = aws_lb.api_server.arn
  port              = 6443
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api_server.arn
  }
}

# 追加 2 个 Server 节点
resource "aws_instance" "control_plane_join" {
  count                  = 2
  ami                    = var.ami_id
  instance_type          = "m5.xlarge"
  subnet_id              = var.private_subnet_ids[count.index + 1]
  vpc_security_group_ids = [var.control_plane_sg_id]
  iam_instance_profile   = var.control_plane_instance_profile
  key_name               = var.key_pair_name

  root_block_device {
    volume_size = 100
    volume_type = "gp3"
    encrypted   = true
  }

  user_data = base64encode(templatefile("${path.module}/../../scripts/rke2-server-join.sh", {
    rke2_version  = var.rke2_version
    rke2_token    = random_password.rke2_token.result
    server_url    = "https://${aws_lb.api_server.dns_name}:9345"
  }))

  depends_on = [aws_instance.control_plane_init]

  tags = {
    Name = "${var.cluster_name}-control-plane-${count.index + 2}"
    "kubernetes.io/cluster/${var.cluster_name}" = "owned"
    "rke2-node-role" = "server"
  }
}
```

**scripts/rke2-server-init.sh**

```bash
#!/bin/bash
set -eux

# 系统初始化
swapoff -a
sed -i '/swap/d' /etc/fstab

# 内核参数
cat >> /etc/sysctl.d/99-k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# 安装 RKE2
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION="${rke2_version}" sh -

# 配置 RKE2 Server
mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/config.yaml <<EOF
token: "${rke2_token}"
tls-san:
  - "${server_lb_dns}"
cluster-cidr: "${pod_cidr}"
service-cidr: "${service_cidr}"
cni: none
disable:
  - rke2-canal
cloud-provider-name: aws
node-label:
  - "node.kubernetes.io/role=control-plane"
EOF

# 启动 RKE2 Server
systemctl enable rke2-server.service
systemctl start rke2-server.service

# 等待 kubeconfig 生成
until [ -f /etc/rancher/rke2/rke2.yaml ]; do sleep 5; done

# 配置 kubectl
mkdir -p /root/.kube
cp /etc/rancher/rke2/rke2.yaml /root/.kube/config
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
ln -sf /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl

echo "RKE2 Server Init Completed"
```

### 6.8 Step 6：部署工作节点（Auto Scaling Group）

**modules/worker_nodes/main.tf**

```hcl
# 启动模板
resource "aws_launch_template" "worker" {
  name_prefix   = "${var.cluster_name}-worker-"
  image_id      = var.ami_id
  instance_type = "m5.2xlarge"
  key_name      = var.key_pair_name

  vpc_security_group_ids = [var.worker_sg_id]

  iam_instance_profile {
    name = var.worker_instance_profile
  }

  block_device_mappings {
    device_name = "/dev/sda1"
    ebs {
      volume_size           = 200
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  user_data = base64encode(templatefile("${path.module}/../../scripts/rke2-agent.sh", {
    rke2_version = var.rke2_version
    rke2_token   = var.rke2_token
    server_url   = var.server_url
  }))

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "${var.cluster_name}-worker"
      "kubernetes.io/cluster/${var.cluster_name}" = "owned"
      "rke2-node-role" = "agent"
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "worker" {
  name                = "${var.cluster_name}-worker-asg"
  desired_capacity    = 50
  min_size            = 10
  max_size            = 60
  vpc_zone_identifier = var.private_subnet_ids

  launch_template {
    id      = aws_launch_template.worker.id
    version = "$Latest"
  }

  health_check_type         = "EC2"
  health_check_grace_period = 300

  tag {
    key                 = "Name"
    value               = "${var.cluster_name}-worker"
    propagate_at_launch = true
  }

  tag {
    key                 = "kubernetes.io/cluster/${var.cluster_name}"
    value               = "owned"
    propagate_at_launch = true
  }
}
```

**scripts/rke2-agent.sh**

```bash
#!/bin/bash
set -eux

swapoff -a
sed -i '/swap/d' /etc/fstab

cat >> /etc/sysctl.d/99-k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_VERSION="${rke2_version}" sh -

mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/config.yaml <<EOF
server: "${server_url}"
token: "${rke2_token}"
cloud-provider-name: aws
node-label:
  - "node.kubernetes.io/role=worker"
EOF

systemctl enable rke2-agent.service
systemctl start rke2-agent.service

echo "RKE2 Agent Join Completed"
```

### 6.9 Step 7：安装 Calico CNI

**calico/custom-resources.yaml**

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
  nodeAddressAutodetectionV4:
    interface: eth0
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

**安装步骤（在 Control Plane 节点执行）：**

```bash
# 1. 安装 Tigera Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# 2. 应用自定义配置
kubectl create -f calico/custom-resources.yaml

# 3. 等待 Calico 组件就绪
kubectl wait --for=condition=Ready pods -n calico-system --all --timeout=300s

# 4. 验证节点状态
kubectl get nodes
```

### 6.10 Step 8：执行 Terraform 部署

```bash
# 初始化
terraform init

# 预览变更
terraform plan -var-file="terraform.tfvars"

# 执行部署
terraform apply -var-file="terraform.tfvars" -auto-approve

# 查看输出
terraform output
```

### 6.11 集群验证

```bash
# 检查节点状态
kubectl get nodes -o wide

# 检查核心组件
kubectl get pods -n kube-system
kubectl get pods -n calico-system

# 检查集群信息
kubectl cluster-info

# 部署测试 Deployment
kubectl create deployment nginx-test --image=nginx --replicas=3
kubectl get pods -o wide
kubectl delete deployment nginx-test
```

## 7. 如何做 K8s 集群的监控

### 7.1 监控体系概览

完整的 K8s 集群监控体系应覆盖以下四个层次：

```
基础设施层（节点 CPU/内存/磁盘/网络）
    ↓
K8s 组件层（API Server、etcd、Scheduler、Controller Manager）
    ↓
工作负载层（Pod、Deployment、DaemonSet 指标）
    ↓
应用业务层（服务 SLI/SLO、错误率、延迟）
```

### 7.2 核心监控组件栈

**推荐方案：kube-prometheus-stack（Prometheus + Grafana + Alertmanager）**

`kube-prometheus-stack` 是目前最主流的 K8s 监控解决方案，通过 Helm 一键部署，集成了：

- **Prometheus Operator**：以 CRD 方式管理 Prometheus 实例、告警规则、抓取配置
- **Prometheus**：时序指标采集与存储
- **Grafana**：可视化大盘
- **Alertmanager**：告警路由与通知（支持钉钉/Slack/PagerDuty 等）
- **kube-state-metrics**：K8s 对象状态指标（Pod 状态、Deployment 副本数等）
- **node-exporter**：宿主机层面指标（CPU、内存、磁盘、网络）

```bash
# 通过 Helm 安装 kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=gp3 \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi \
  --set grafana.adminPassword="your-secure-password" \
  --set alertmanager.enabled=true
```

### 7.3 关键监控指标

**节点层指标**

| 指标名 | 说明 | 告警阈值建议 |
|--------|------|-------------|
| `node_cpu_seconds_total` | 节点 CPU 使用率 | > 85% 持续 5min |
| `node_memory_MemAvailable_bytes` | 节点可用内存 | < 10% 持续 5min |
| `node_filesystem_avail_bytes` | 磁盘剩余空间 | < 15% |
| `node_disk_io_time_seconds_total` | 磁盘 IO 压力 | ioutil > 90% |

**K8s 组件层指标**

| 指标名 | 说明 | 告警阈值建议 |
|--------|------|-------------|
| `apiserver_request_duration_seconds` | API Server 延迟 | P99 > 1s |
| `apiserver_request_total` | API Server 请求错误率 | 5xx 错误率 > 1% |
| `etcd_disk_wal_fsync_duration_seconds` | etcd WAL 写入延迟 | P99 > 10ms |
| `etcd_server_has_leader` | etcd Leader 是否存在 | = 0 立即告警 |

**工作负载层指标**

| 指标名 | 说明 | 告警阈值建议 |
|--------|------|-------------|
| `kube_pod_status_phase` | Pod 非 Running 状态 | Pending > 10min |
| `kube_deployment_status_replicas_unavailable` | Deployment 不可用副本数 | > 0 持续 5min |
| `container_memory_working_set_bytes` | 容器内存使用量 | 接近 limit 的 90% |
| `container_oom_events_total` | 容器 OOM 次数 | > 0 立即告警 |

### 7.4 日志监控方案

指标监控解决"是什么问题"，日志则帮助解决"为什么出问题"。推荐采用 **Loki + Promtail** 方案与 Grafana 集成，实现指标与日志的统一视图：

```bash
helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi
```

### 7.5 链路追踪（可选）

对于微服务架构，建议引入分布式追踪能力，推荐 **OpenTelemetry + Jaeger / Tempo** 方案：

- **OpenTelemetry Collector**：统一采集应用 Trace、Metric、Log
- **Jaeger / Grafana Tempo**：Trace 存储与查询
- **Grafana**：通过 Exemplar 实现 Metric → Trace 的联动跳转

### 7.6 告警通知配置示例

**Alertmanager 钉钉通知配置（alertmanager-config.yaml）**

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: cluster-alerts
  namespace: monitoring
spec:
  route:
    receiver: dingtalk-webhook
    groupBy: ['alertname', 'namespace']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
  receivers:
    - name: dingtalk-webhook
      webhookConfigs:
        - url: 'http://dingtalk-webhook-adapter:8060/dingtalk/webhook/send'
          sendResolved: true
```

### 7.7 推荐 Grafana Dashboard

| Dashboard ID | 用途 |
|-------------|------|
| 15757 | Kubernetes / Views / Global（集群全局视图） |
| 13332 | Kubernetes Cluster Monitoring |
| 1860  | Node Exporter Full（节点详细监控） |
| 7249  | Kubernetes Cluster（工作负载视图） |
| 3070  | Kubernetes Persistent Volumes |

通过 Grafana 的 Import 功能直接导入以上 Dashboard，可以快速获得开箱即用的监控大盘，无需从零配置。

我之前所在的企业 PaaS 平台为了满足客户多样性需求自研了一套多云管理平台，提供了源码托管、CI/CD、资源管理等一整套体系，管控端部署在中心集群，一个客户一套边缘集群。
