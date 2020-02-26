EKS:
Subnets:
 - Al menos 2 AZ, public o private
 - Si es privado: kubernetes no puede crear salida a internet incluyendo a balanceador en los pods
 - Si es publico: Todos los nodos seran publicos.

Security group: basados en la recomendacion de AWS: https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html
Protocol Port 		Source 			Destination 		Note
TCP 	10250 		EKS Control Plane 	EKS Worker 		Required Port for Communication from Control Plane to Worker
TCP 	1025-65535 	EKS Control Plane 	EKS Worker 		Recommended Port for Communication from Control Plane to Worker
TCP 	0-65535 	EKS Control Plane 	EKS Worker 		To allow proxy functionality on privileged ports or to run the CNCF conformance tests
TCP 	443 		EKS Worker 		EKS Control Plane 	Allow Worker Nodes to call Control Plane API
All 	All 		EKS Worker 		EKS Worker 		Communication between Nodes internally
All 	All 		EKS Worker 		0.0.0.0/0 		Allow Workder to access Internet


Tageo:
Key 					Value 	Resource 	Note
kubernetes.io/cluster/<cluster-name> 	shared 	VPC 		Tag VPC that's contain EKS
kubernetes.io/cluster/<cluster-name> 	shared 	All Subnets 	Tag All EKS Subnets
kubernetes.io/role/internal-elb 	1 	Private Subnets Tag EKS Private Subnets for Internal LB
kubernetes.io/role/elb 			1 	Public Subnets 	Tag EKS Public Subnets that could be used for External LB otherwise it will auto choose lexicographical order by subnet ID
kubernetes.io/cluster<cluster-name> 	owned 	All EC2 	Tag All EC2 that's belong to EKS Worker Nodes


Seguridad: Otra caracteristica q se debe tomar en cuenta si se debe ejecutar comandos a travez de kubectl para modificare el cluster de EKS, esto no se debe poder. Si se requiere esta caracteristica, debemos colocar una instancia 'bastion' que nos permita desde esta instancia ejecutar comandos al cluster,
IMPORTANTE: no se puede activar EKS privado por cloudformation, solo x el comando aws eks

aws eks --region <region> update-cluster-config --name <cluster> --resources-vpc-config endpointPublicAccess=<true/false>,endpointPrivateAccess=<true/false>

Kubernetes Role Based Access Control:
En el caso de conectarnos al cluster de EKS y se requiera permisos finos para cada usuario necesitamos RBAC el cual homologara los roles definidos en aws con el cluster d eks

Bastion Host es una instancia EC2 afuera del cluster de EKS 

Worker node AMI: se proveera una AMI optimizada para eks por AWS: https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html, esta wa esta sobre una AWS AMI Linux 2, osea YUMero, adicionalmente tiene: Docker, el agente kubelet, AWS IAM Authenticator, chekar tmb la version de kubernetes dado q con eso se tiene la Imagen EKS

---===---
Herramientas instaladas:
1. AWS CLI 2: Call AWS API Services, https://aws.amazon.com/cli/
2. kubectl: Call Kubernetes API, https://kubernetes.io/docs/tasks/tools/install-kubectl/
3. aws-iam-authenticator: A tool to authenticate to Kubernetes using AWS IAM credentials, https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html
4. ssh-keygen
5. ssh

---===---
https://akachain.readthedocs.io/en/latest/_images/slide3.png
https://user-images.githubusercontent.com/33742463/37858995-e8131e08-2f47-11e8-8dea-37aeea762ea0.png

---===---
CLOUDFORMATION

Plantillas Cloudformations:
01_IAM.yml: Responsable de aprovisionar recursos relacionados a IAM; roles, politicas, grupos, roles para instancias bastion, EKS instance profile.
VPC: Responsable de aprovisionar recursos como: simple VPC, y toda la red, subnets publicas y privadas, InternetGateway, RouteTable
Bastion: Instancia para conectar al cluster de EKS
EKS Cluster: Cluster EKS, Nodos, launch template.
EKS Nodes: nodes config to EKS
RBAC Role: Ec2 - ALB Ingress controller
ALB Ingress Controller: Ruteo a los nodos.































---===---

Nat Gateway deberia ser spot o Ondemand y darle salida a internet cuando se requiera y luego bajarlo