# Amazon EKS: Gerenciando aplicações conteinerizadas com Kubernetes
##### https://cursos.alura.com.br/course/amazon-eks-kubernetes

## Pré-requisitos
* Kubernetes
* AWS CLI

## Primeiros passos - VPC
Antes de criar um cluster k8s através da Amazon EKS (Elastic Kubernetes Service), primeiramente, deve-se criar no **IAM** uma **Role** para termos autorização de **criar e gerenciar um cluster k8s**.

Após isso, como a própria AWS sugere, é preciso criar uma **VPC (Virtual Private Cloud)** para que o cluster utilize uma rede privada e segura.

Neste curso, para simplificar e manter o foco apenas no EKS, a VPC é criada através do **Amazon CloudFormation**, que é basicamente um serviço de criação de VPC via arquivo json/yaml. Para o curso, a VPC é criada através do arquivo yaml:

```https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-12-10/amazon-eks-vpc-sample.yaml```

## Criando o cluster

Para se criar o cluster através do CLI, é utilizado o comando:

```
aws eks create-cluster --name <cluster_name> \
    --role-arn <role_arn_com_permissao_criacao_gerenciamento_cluster_k8s_criada_anteriormente> \
    --resources-vpc-config subnetIds=<vpc_subnet_id1>,<vpc_subnet_id2>...,securityGroupIds=<vpc_security_group_id>
```

Após isso, para ver se o cluster foi criado, liste o cluster com o comando:

`aws eks list-clusters`

Então, verifique o status do cluster:

`aws eks describe-cluster --name <cluster_name>`

Após o cluster ter sido criado, para associar o `kubectl` ao cluster criado:

`aws eks update-kubeconfig --name <cluster_name>`

Para validar se o `kubectl` está associado corretamente ao cluster criado, verificar se o contexto existe e está associado com:

`kubectl config get-contexts`

## Criando os nodes

Após o cluster ter sido criado, é preciso criar os nodes workers para então ser possível fazer o deploy das pods. Para tal, é necessário então adicionar as instâncias EC2 no cluster criado anteriormente.

Para agilizar o processo, será utilizado novamente o CloudFormation para que as instâncias sejam criadas de forma mais ágil.

No CloudFormation então, criar as instâncias necessárias através do seguinte arquivo de configuração:

`https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-12-10/amazon-eks-nodegroup.yaml`
 
Após selecionar a stack a ser criada através do arquivo acima, se atentar aos passos abaixo:

#### EKS Cluster 
**ClusterName**: aqui deverá ser preenchido com o nome do cluster criado na seção anterior.

**ClusterControlPlaneSecurityGroup**: utilizar o Security Group criado no passo de criação da VPC (estará algo como `VPC-****-ControlPlaneSecurityGroup***`).

#### Worker Node Configuration
**NodeGroupName**: qualquer nome desejado para esse grupo de nodes.

**NodeAutoScalings**: selecionar a quantidade de instâncias conforme desejado.

**NodeInstanceType**: pode ser utilizado até mesmo a t3.micro (free tier).

**NodeImageId**: AMI (ou ID de imagem da máquina) a ser criado, no User Guide do EKS há uma lista de AMIs otimizadas para o EKS conforme as regiões.

**KeyName**: EC2 Key Pair para acessar via SSH

#### Worker Network Configuration

**VpcId**: Selecionar a VPC criada anteriormente.

**Subnets**: Selecionar as subnets criadas juntamente com a VPC acima.

---

Seguindo todos os passos acima, ao finalizar o processo, basicamente ocorrerá a criação de 3 instâncias do EC2 e todas as configurações de Auto Scale.

Para verificar a criação de ambos, primeiramente acessar o menu EC2 e verificar se as instâncias foram criadas conforme a configuração utilizada.
Depois disso, acessar no menu lateral de Auto Scaling o Launch Configurations. Nesta seção estará criada a configuração de auto scaling utilizada para as instâncias criadas no passo anterior, sendo possível ver os perfis das máquinas e a AMI utilizada.

Acessando no menu lateral o Auto Scaling Groups, estará as configurações de quantidade de instâncias deste grupo. Aqui, poderá ser adicionado ou retirada instâncias conforme o desejado.

## Adicionando os nodes ao cluster

Apesar de ter sido criados os nodes, eles não foram adicionados ao cluster do kubernetes, somente ao cluster da Amazon. Basta verificar com o comando:

`kubectl get nodes`

Para adicionar os nodes ao cluster do kubernetes, primeiro baixar o arquivo de configuração:

`curl -O https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-08-30/aws-auth-cm.yaml`

Neste arquivo de configuração, alterar o item rolearn com o arn gerado nos Outputs dos nodes criados pelo CloudFormation.
Após isso, aplique o arquivo no cluster kubernetes e em seguida, observe os nodes serem associados ao cluster com os seguintes comandos:

`kubectl apply -f aws-auth-cm.yaml`

`kubectl get nodes --watch`

## Deploy da aplicação no cluster

Com toda a estrutura criada, basta apenas agora realizar o deploy de qualquer aplicação desejada. Neste caso, será realizado o deploy da aplicação [zup-client-services]().

Para fazer o deploy da aplicação, primeiramente entregar o stateful-set, o persistent-volume-claim e o cluster-ip do banco de dados PostgreSQL utilizado pela aplicação: 

`kubectl apply -f https://raw.githubusercontent.com/ariielm/zup-client-services/master/kubernetes/database-stateful-set.yaml`

Após isso, poderá ser entregue a aplicação java com suas três réplicas através do deployment e o seu load-balancer:

`kubectl apply -f https://raw.githubusercontent.com/ariielm/zup-client-services/master/kubernetes/deployment.yaml`

Com toda a aplicação entregue, pode-se acessar no AWS Console o load-balancer criado, e através do DNS provisionado pelo load-balancer, acessar a aplicação com o path `/actuator/health`.

Para testar se tudo está funcionando corretamente, utilizar o [postman do projeto](https://github.com/ariielm/zup-client-services/tree/master/postman), e com isso será possível testar todas as funcionalidades da API.
 
## Escalonando cluster

Acesssar no console da AWS o menu Auto Scaling Groups no EC2. Selecionar o cluster criado e editá-lo, com quantas instâncias desejar.
Após isso, ao acessar as instâncias, elas já terão sido alteradas conforme feito anteriormente.

Quando as instâncias tiverem sido criadas, pode-se validar através do kubernetes com o comando:

`kubectl get nodes`

E caso necessite aumentar a quantidade de replicas do zup-client-services, basta executar:

`kubectl scale deploy zup-client-services-deployment --replicas=5`

## Deploy do Dashboard

Para acessar o dashboard do kubernetes precisa-se primeiro fazer o deploy do mesmo, pois ele não vem habilitado inicialmente na criação do cluster.

Seguir os passos através da documentação da Amazon EKS [Tutorial: Implantar o painel do Kubernetes (interface do usuário da web)](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/dashboard-tutorial.html).

Ao seguir todos os passos já será possível acessar o dashboard e alterar a quantidade de pods para um deployment e todas as métricas do kubernetes.

## Arrumando a casa

Caso o cluster não vá ser utilizado, é necessário excluir toda a arquitetura criada.

1. Primeiramente deve-se acessar o CloudFormation e deletar o cluster de nodes criado. Isso irá deletar todas as instâncias EC2 que continham todas as pods.
2. Também é necessário deletar o cluster eks. Para isso, acessar o menu de EKS e deletar o cluster que havia sido criado.
3. Para finalizar, acessar o CloudFormation e deletar a VPC que havia sido criada no primeiro passo.

Com isso agora, toda a infraestrutura criada já foi deletada.




