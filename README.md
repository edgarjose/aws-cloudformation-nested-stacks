# CloudFormation: laboratório de nested stacks

Uma stack principal coordena VPC, dois NAT Gateways, ALB público e um Auto Scaling Group de EC2 em sub-redes privadas.

Ao contrário de referências entre stacks independentes, as stacks deste exemplo pertencem à stack principal e recebem os seus parâmetros através dela.

## Estrutura

| Ficheiro | Conteúdo |
| --- | --- |
| `master.yaml` | Stack principal e ligações entre componentes |
| `vpc-subnets.yaml` | VPC, duas sub-redes públicas e duas privadas, security groups |
| `natgateway.yaml` | Saída das sub-redes privadas através de NAT |
| `alb.yaml` | Listener HTTP e target group |
| `asg.yaml` | Amazon Linux 2023, Apache, duas instâncias e target tracking de CPU |

O resultado é uma página de teste servida pelas instâncias através do ALB. Não existe RDS neste projeto.

## Pré-requisitos

AWS CLI v2, um bucket S3 privado para empacotar templates na região escolhida e permissões para os recursos do laboratório e parâmetros públicos SSM. Escolher uma região com pelo menos duas zonas disponíveis.

## Empacotar e implementar

Os `TemplateURL` apontam para ficheiros locais. É obrigatório empacotar antes de implementar: não executar deploy diretamente com o `master.yaml` original.

```bash
export AWS_DEFAULT_REGION=us-east-1
export ARTIFACT_BUCKET=SEU_BUCKET_PRIVADO
aws sts get-caller-identity
aws cloudformation package --template-file master.yaml --s3-bucket "$ARTIFACT_BUCKET" --output-template-file packaged.yaml
aws cloudformation deploy --template-file packaged.yaml --stack-name practical-nested-lab
```

O package publica os templates dependentes no teu bucket. A stack do ASG aguarda a de NAT para que a instalação do Apache tenha saída para a internet.

## Verificar

Na stack principal, abrir a nested stack `ALBStack` e consultar o output `ALBDNS`. Abrir `http://DNS_DO_ALB/` depois de confirmar os targets saudáveis. Verificar a página de teste, duas instâncias e as rotas privadas.

As AMIs vêm do parâmetro público SSM de Amazon Linux 2023, e podem mudar entre implementações. Um teste real na AWS continua necessário; validação de YAML não prova que o UserData ou o acesso à internet funcionaram.

## Custos e remoção

Dois NAT Gateways, ALB, EC2 e endereços IPv4 podem gerar custos. A configuração de SSH herdada é ampla: não associar esse grupo a hosts públicos sem a restringir.

Eliminar a stack principal, não cada nested stack manualmente:

```bash
aws cloudformation delete-stack --stack-name practical-nested-lab
aws cloudformation wait stack-delete-complete --stack-name practical-nested-lab
```

Confirmar que os recursos do laboratório foram removidos. Os artefactos enviados para o bucket de empacotamento não são eliminados pela stack; remover apenas os objetos dedicados ao lab, sem apagar ficheiros partilhados.

[Nested stacks na AWS](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html) · [Referências entre stacks](https://github.com/edgarjose/aws-cloudformation-cross-reference)
