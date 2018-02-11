# Plug-in AWS para Graylog


[![Build Status](https://travis-ci.org/Graylog2/graylog-plugin-aws.svg)](https://travis-ci.org/Graylog2/graylog-plugin-aws)
[![Github Downloads](https://img.shields.io/github/downloads/Graylog2/graylog-plugin-aws/total.svg)](https://github.com/Graylog2/graylog-plugin-aws/releases)
[![GitHub Release](https://img.shields.io/github/release/Graylog2/graylog-plugin-aws.svg)](https://github.com/Graylog2/graylog-plugin-aws/releases)

Este plugin fornece os seguintes módulos Graylog:

* plugin de entrada de Logs para [AWS Flow ](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/flow-logs.html)  de conexão de interface de rede
* plugin de e entrada  logs para [AWS Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)
* plugin de entrada  logs para [AWS CloudTrail](http://aws.amazon.com/cloudtrail/) 

Compatibilidade da versão do Graylog
-----------------------------

| Plugin Version | Graylog Version |
| -------------- | --------------- |
| 2.4.x          | 2.4.x           |
| 2.3.x          | 2.3.x           |
| 1.3.2          | 2.2.2           |
| 1.2.1          | 2.1.3           |
| 0.6.0          | 2.0.x           |

## Instalação

Instale o graylog ou use uma imagem iam do graylog

Adicione uma role somente com as permissões  para  acesso aos recursos  necessários para execução dos procedimentos na instãncia ec2   após a execução dos procedimentos mantenha  somente os acessos necessários , os recursos usados no procedimento são :

 Cloudwatch
AmazonDynamodb
 AmazonKinesis

* Obs: Não vou explicar  o perigo de  criar role com  acessos indevidos, apenas tenha cuidado  para  não liberar acesso total aos recursos e esquecer de remove-los ,  na documentação original não tem procedimento de liberação de acesso mas o acesso ao  stream do Kinesis após a criação do stream   é somente leitura e o plugin  somente cria uma tabela no dynamodb  e depois somente grava as informações de integração  do kinesis nela


> Desde Graylog Versão 2.4.0 este plugin já está incluído no pacote de instalação do servidor Graylog como plugin padrão.

[Baixe o plugin](https://github.com/Graylog2/graylog-plugin-aws/releases)
e coloque o arquivo `.jar`   no seu diretório de plugins do Graylog. O diretório de plugin é a pasta relativa `plugins/`  de seu diretório do  `graylog-server`  por  padrão e pode ser configurado no seu arquivo `graylog.conf`.

Reinicie `graylog-server` e  e pronto.

## Configuração geral

Depois de instalar o plugin, você terá uma nova seção de configuração do cluster em  “System -> Configurations” na sua interface  Web do seu Graylog. Certifique-se de completar a configuração antes de usar qualquer um dos módulos que este plugin fornece. Você verá muitos avisos no arquivo de log do `graylog-server` se você não conseguir fazê-lo.


### Inserindo logs de recursos AWS

A configuração deste plugin possui um parâmetro que controla se as logs de recurso AWS estão sendo inseridos  ou não. Isso basicamente significa que o plugin tentará encontrar determinados campos como um endereço IP de origem e enriquecer a mensagem de log com mais informações sobre o recurso AWS (como um  EC2, uma instância ELB, um banco de dados RDS, ...) automaticamente.

Isso seria algo assim:

[![](https://s3.amazonaws.com/graylog2public/aws_translation.jpg)](https://s3.amazonaws.com/graylog2public/aws_translation.jpg)

Aqui estão as permissões IAM necessárias caso você decida usar esse recurso:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1469415911000",
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:DescribeLoadBalancerAttributes",
                "elasticloadbalancing:DescribeLoadBalancers"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Sid": "Stmt1469415936000",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeNetworkInterfaceAttribute",
                "ec2:DescribeNetworkInterfaces"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

## configuração do FlowLogs

Os exemplos de integração e análise do Flow Logs estão descritos [neste post no blog www.itculate.io](https://www.itculate.io/documentation/knowledge-base/kb-0109/).

### Passo 1: Habilitar Flow Logs

Há duas maneiras de ativar o Flow Logs para uma interface de rede AWS:

Pela  interface de rede específica no seu console EC2,“Network Interfaces” no link de navegação principal

[![](https://s3.amazonaws.com/graylog2public/flowlogs_1.jpg)](https://s3.amazonaws.com/graylog2public/flowlogs_1.jpg)

… ou para todas as interfaces de rede em sua VPC usando o console VPC:

[![](https://s3.amazonaws.com/graylog2public/flowlogs_3.jpg)](https://s3.amazonaws.com/graylog2public/flowlogs_3.jpg)

Após alguns minutos (geralmente 15 minutos, mas pode demorar até uma hora), a AWS começará a escrever o Flow Logs e você pode visualizá-los no seu console CloudWatch:

[![](https://s3.amazonaws.com/graylog2public/flowlogs_2.jpg)](https://s3.amazonaws.com/graylog2public/flowlogs_2.jpg)

Agora vamos continuar instruindo AWS para escrever o FlowLogs para um Stream [Kinesis](https://aws.amazon.com/kinesis/) .

### Passos 2: configurar o Stream do Kinesis

Criar um Stream [Kinesis](https://aws.amazon.com/kinesis/) usando as ferramentas AWS CLI:

    aws kinesis create-stream --stream-name "flowlogs" --shard-count 1

Agora receba os detalhes do Stream:

    aws kinesis describe-stream --stream-name "flowlogs"

**Copie o StreamARN da saída.** Nós precisaremos dele mais tarde.

Em seguida, crie um arquivo chamado _trust_policy.json_ com o seguinte conteúdo:

```
{
  "Statement": {
    "Effect": "Allow",
    "Principal": { "Service": "logs.us-east-1.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }
}
```

**Certifique-se de alterar o Service de us-east-1 para a Região em que você está executando.**

Agora, crie uma nova função IAM com as permissões no arquivo que acabamos de criar:

    aws iam create-role --role-name CWLtoKinesisRole --assume-role-policy-document file://trust_policy.json

**Copie o ARN da role que você acabou de criar.** Você vai precisar disso no próximo passo.

Crie um novo arquivo chamado _permissions.json_ e defina ambos ARNs para os ARNs copiados acima:

```
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "kinesis:PutRecord",
      "Resource": "[SUA ARN DO STREAM DO KINESIS AQUI]"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "[SUA ARN DO IAM AQUI]"
    }
  ]
}
```

Agora atribua esse role:

    aws iam put-role-policy --role-name CWLtoKinesisRole --policy-name Permissions-Policy-For-CWL --policy-document file://permissions.json

O último passo é criar uma subscription  que irá escrever o FlowLogs para o Kinesis:

```
aws logs put-subscription-filter \
    --filter-name "MatchAllValidFilter" \
    --filter-pattern "OK" \
    --log-group-name "my-flowlogs" \
    --destination-arn "[SEU ARN STREAM DO KINESIS AQUI]" \
    --role-arn "[SUA ARN DO IAM AQUI]"
```

Agora você deve ver FlowLogs sendo escrito em seu Stream Kinesis.

### Etapa 4: iniciar  Entrada(Input)

Agora entre na interface  Web do Graylog e comece uma nova *Entrada AWS FlowLogs*. Ele irá pedir-lhe alguns parâmetros simples, como o “nome” do Kinesis Stream ao qual você está escrevendo o FlowLogs.

Você deve ver algo assim no arquivo de log do `graylog-server` depois de iniciar a entrada:

```
2017-06-03T15:22:43.376Z INFO  [InputStateListener] Input [AWS FlowLogs Input/5932d443bb4feb3768b2fe6f] is now STARTING
2017-06-03T15:22:43.404Z INFO  [FlowLogReader] Starting AWS FlowLog reader.
2017-06-03T15:22:43.404Z INFO  [FlowLogTransport] Starting FlowLogs Kinesis reader thread.
2017-06-03T15:22:43.410Z INFO  [InputStateListener] Input [AWS FlowLogs Input/5932d443bb4feb3768b2fe6f] is now RUNNING
2017-06-03T15:22:43.509Z INFO  [LeaseCoordinator] With failover time 10000 ms and epsilon 25 ms, LeaseCoordinator will renew leases every 3308 ms, takeleases every 20050 ms, process maximum of 2147483647 leases and steal 1 lease(s) at a time.
2017-06-03T15:22:43.510Z INFO  [Worker] Initialization attempt 1
2017-06-03T15:22:43.511Z INFO  [Worker] Initializing LeaseCoordinator
2017-06-03T15:22:44.060Z INFO  [KinesisClientLibLeaseCoordinator] Created new lease table for coordinator with initial read capacity of 10 and write capacity of 10.
2017-06-03T15:22:54.251Z INFO  [Worker] Syncing Kinesis shard info
2017-06-03T15:22:55.077Z INFO  [Worker] Starting LeaseCoordinator
2017-06-03T15:22:55.279Z INFO  [LeaseTaker] Worker graylog-server-master saw 1 total leases, 1 available leases, 1 workers. Target is 1 leases, I have 0 leases, I will take 1 leases
2017-06-03T15:22:55.375Z INFO  [LeaseTaker] Worker graylog-server-master successfully took 1 leases: shardId-000000000000
2017-06-03T15:23:05.178Z INFO  [Worker] Initialization complete. Starting worker loop.
2017-06-03T15:23:05.203Z INFO  [Worker] Created new shardConsumer for : ShardInfo [shardId=shardId-000000000000, concurrencyToken=9f6910f6-4725-3464e7e54251, parentShardIds=[], checkpoint={SequenceNumber: LATEST,SubsequenceNumber: 0}]
2017-06-03T15:23:05.204Z INFO  [BlockOnParentShardTask] No need to block on parents [] of shard shardId-000000000000
2017-06-03T15:23:06.300Z INFO  [KinesisDataFetcher] Initializing shard shardId-000000000000 with LATEST
2017-06-03T15:23:06.719Z INFO  [FlowLogReader] Initializing Kinesis worker.
2017-06-03T15:23:44.277Z INFO  [Worker] Current stream shard assignments: shardId-000000000000
2017-06-03T15:23:44.277Z INFO  [Worker] Sleeping ...
```

**Levará alguns minutos até os primeiros registros entrarem.**

**Importante: A AWS entrega o FlowLogs com alguns minutos de atraso e nem sempre de uma forma ordenada. Tenha isso em mente ao pesquisar sobre mensagens em um frame de tempo recente.**

##  configuração do CloudTrail

### Etapa 1: ativando o CloudTrail para uma região AWS

Comece habilitando CloudTrail para uma região AWS:

![Configuring CloudTrail](https://raw.githubusercontent.com/Graylog2/graylog-plugin-aws/master/images/plugin-aws-input-1.png)

* **Create a new S3 bucket:** Yes
* **S3 bucket:** Escolha qualquer coisa aqui, você não precisa disso para a configuração do Graylog mais tarde
* **Log file prefix:** Opcional, não é necessário para a configuração Graylog
* **Include global services:** Yes (você pode querer mudar isso ao usar CloudTrail em várias regiões AWS)
  * **SNS notification for every log file delivery:** Yes
  * **SNS topic:** Escolha algo como * cloudtrail-log-write * aqui. Lembre-se do nome.

### Step 2: Configure SQS para notificações de gravação CloudTrail

Navegue até o serviço AWS SQS (na mesma região que o CloudTrail ativado) e pressione **Create New Queue**.

![Creating a SQS queue](https://raw.githubusercontent.com/Graylog2/graylog-plugin-aws/master/images/plugin-aws-input-2.png)

Você pode deixar todas as configurações em seus valores padrão por enquanto, mas anote o **Queue Name** porque você precisará para a configuração Graylog mais tarde. Nosso valor padrão recomendado é *cloudtrail-notifications*.

CloudTrail irá escrever notificações sobre os arquivos de log que escreveu para o S3 para esta fila e o Graylog precisa dessas informações. Vamos assinar a fila SQS no tópico CloudTrail SNS que você criou na primeira etapa agora:

![Subscribing SQS queue to SNS topic](https://raw.githubusercontent.com/Graylog2/graylog-plugin-aws/master/images/plugin-aws-input-3.png)

Clique com o botão direito do mouse na nova fila que você acabou de criar e selecione *Subscribe Queue to SNS Topic*. Selecione o tópico SNS que você configurou no primeiro passo ao configurar CloudTrail. **Acerte se inscreva e você terá completado  a configuração AWS.**

### Passo 3: Instale e configure o plugin Graylog CloudTrail

Copie o arquivo `.jar` que você recebeu no seu diretório do plugin Graylog que está configurado no arquivo de configuração` graylog.conf` usando a variável `plugin_dir`.

Reinicie `graylog-server` e você deve ver o novo tipo de entrada * AWS CloudTrail Input * em * System -> Inputs -> Launch new input *. A configuração de entrada necessária deve ser auto-explicativa.

**Importante:** O usuário IAM que você configurou em “System -> Configurations” tem permissões para ler os logs CloudTrail do S3 e escrever notificações do SQS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1411854479000",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::cloudtrail-logfiles/*"
      ]
    }
  ]
}
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1411834838000",
      "Effect": "Allow",
      "Action": [
        "sqs:DeleteMessage",
        "sqs:ReceiveMessage"
      ],
      "Resource": [
        "arn:aws:sqs:eu-west-1:450000000000:cloudtrail-write"
      ]
    }
  ]
}
```

(Certifique-se de substituir os valores * resource * por ARNs reais do seu ambiente)

**Mais roles IAM necessários:** A maneira como nos comunicamos com a Kinesis nos obriga a armazenar alguns metadados no AWS DynamoDB e também estamos escrevendo algumas métricas para a AWS CloudWatch. Para que isso funcione, você deve anexar as seguintes políticas AWS IAM padrão ao seu usuário da AWS API:

* CloudWatchFullAccess
* AmazonDynamoDBFullAccess
* AmazonKinesisReadOnlyAccess

**Observe que estas são permissões padrão muito abertas. ** Recomendamos usá-las para uma configuração de teste, mas também as reduzirem para permitir apenas o acesso (leitura + gravação) à tabela DynamoDB que criamos automaticamente (você a verá na lista de tabelas) e também para chamar `cloudwatch: PutMetricData`. Como obter as ARNs e como criar políticas personalizadas ficaria fora do escopo deste guia.

## Uso

Você deve ver as mensagens do CloudTrail entrar depois de iniciar a entrada. (Observe que pode demorar alguns minutos com base em quantos sistemas freqüentes acessam seu recurso AWS) ** Você pode até parar o Graylog e alcançará todas as mensagens do CloudTrail que foram escritas desde que foi interrompido quando ele foi iniciado novamente. * *

**Agora faça uma pesquisa no Graylog. Selecione “Search in all messages” and search for:** `source:"aws-cloudtrail"`

## Construir

Este projeto está usando Maven 3 e requer Java 8 ou superior.

Você pode criar um plugin (JAR) com `mvn package`.

Os pacotes DEB e RPM podem ser compilados com `mvn jdeb:jdeb` e` mvn rpm:rpm` respectivamente.

## Versão do Plugin

Estamos usando o  maven  para gerar uma versão do plugin:

```
$ mvn release:prepare
[...]
$ mvn release:perform
```

Isso configura os números de versão, cria uma tag e empurra para o GitHub. A Travis CI irá construir os artefatos de lançamento e fazer o upload para o GitHub automaticamente.
