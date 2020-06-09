# Amazon EKS: Gerenciando aplicações conteinerizadas com Kubernetes
##### https://cursos.alura.com.br/course/amazon-eks-kubernetes

## Pré-requisitos
* Kubernetes
* AWS CLI
* eksctl

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



