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

-----====-----
HOW TO RUN:
1. Mover se a la carpeta cloudformation
2. Ejecutar deploy_all.sh, esto creara toda la infraestructura(Cluster EKS, Bastion, VPC, IAM)
3. Agregar usuario X a EKSAccessGroup, para q tng accesso al cluster: aws2 iam add-user-to-group --user-name dev-zoluxiones-dsilva --group-name Iam-Stack-eks-group-EksAccessGroup
4. Habilitar accesso a la herramienta kubectl
5. Probar que se tiene accesso y esta habilitada la herramienta: kubectl get node, debe estar vacio

---
x. Modificar el script de 05_Nodes.yml y modificar la linea 8 y agregar el role ARN
6. Registrar los nodos al EKS: 
kubectl apply -f 05_Nodes.yaml
7. Revisar los nodos estan corriendo: kubectl get nodes --watch
8. Agregar el usuario al mapUsers del cluster: kubectl edit -n kube-system configmap/aws-auth
anadir lo comentado en 05_Nodes.yaml
9. Agregamos el usuario nuevo para q tng privilegios como el q creo el cluster:
aws2 eks update-kubeconfig --name Cluster-Test-eks --profile dev-zoluxiones-dsilva

---- opcional bastion
# TODO: 

---- Application
1. Necesitamos vincular kubernetes RBAC Role y el ALB Ingress
kubectl apply -f ./06_RBAC_Role.yaml
2. Necesitamos actualizar el nombre del cluster, editar archivo 06_RBAC_Role.yaml, linea: --cluster-name=...
y colocar el nombre del cluster EKS q hemos creado
3. Ejecutamos nuestra aplicacion, en orden:
 - namespace
 - deployment
 - service
 - ingress
4. Chekar el ALB Ingreess, demora mucho...:kubectl get ingress/2048-ingress -n 2048-game


Nota: Si instalaste la version 2 de AWS CLI, entonces modifica el archivo ~/.kube/config, y modifica la siguiente instruccion:
  command: aws -> command: aws2
























---===---

Nat Gateway deberia ser spot o Ondemand y darle salida a internet cuando se requiera y luego bajarlo