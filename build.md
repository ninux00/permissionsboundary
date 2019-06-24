# Permissions boundaries workshop <small> Fase de Construção </small>

<large> **Atenção** </large>
<p style="font-size:16px;">
Durante o workshop, tenha em mente que você precisa adicionar o Account ID (troque <ACCOUNT_ID>), use corretamente o "path" e altere a região especificada caso necessário (caso esteja realizando este workshop através de um evento da AWS, simplesmente use a região já especificada "us-east-2"). A falta de qualquer um desses componentes pode resultar em problemas e erros como **"An error occurred (MalformedPolicyDocument) when calling the CreatePolicy operation: The policy failed legacy parsing"**.
</p>

!!! Dica
<p style="font-size:16px;">
As tarefas 1, 2 e 3 podem ser realizadas independentemente se você estiver trabalhando em grupo e quiser dividir as tarefas.
</p>


## Instruções de configuração do ambiente

Para configurar o ambiente, expanda e selecione a opção apropriada de acordo com as opções abaixo, realize as tarefas e então prossiga para **Tarefa 1**.

**Informação**  "Siga as instruções abaixo se você está em um *evento AWS* e  *AWS disponibilizou uma conta para você*" 

**Passo 1**:  Logue no console e execute o CloudFormation template.

**Console Login:** Seu time deve ter recebido um pedaço de papel com URL e as credenciais. Isso te permitirá fazer login na conta AWS.
    Depois de se autenticar na conta AWS, tenha certeza de selecionar a região us-east-2**(Ohio)** 

** Click no botão *Deploy to AWS*  abaixo para lançar a stack do CloudFormation. **
	
Region| Deploy
------|-----
US East 2 (Ohio) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=Perm-Bound-Adv&templateURL=https://s3-us-west-2.amazonaws.com/sa-security-specialist-workshops-us-west-2/identity-workshop/permissionboundary/identity-workshop-web-admins-advanced.yaml" target="_blank">![Deploy in us-east-2](./images/deploy-to-aws.png)</a>
	
1. Click em **Next** na seção da página **Select Template**.
2. Click em **Next** na seção da página **Specify Details** (o nome da stack será preenchida automaticamente - você pode alterar ou deixar como está)
3. Click em **Next** na seção da página **Options**.
4. Finalmente, permita que o template crie Roles do IAM preenchendo o check box abaixo de **Capabilities** e click em **Create**.
5. Isso te trará de volta para a página de console do CloudFormation. Você pode recarregar a página para ver o progresso da criação da stack. Antes de continuar tenha certeza que o status está mostrando **CREATE_COMPLETE**.

**Passo 2** : Conecte na IDE do AWS Cloud9
	
1. Navegue para o console do <a href="https://us-east-2.console.aws.amazon.com/cloud9/home" target="_blank">AWS Cloud9</a>.
2. Click em **Open IDE** na página `workshop-environment` em baixo de **Your environments**
3. Click no ícone de **engrenagem** no canto superior direito para abrir as preferencias do Cloud9. Role para baixo e click na seção **AWS SETTINGS** e click  no botão próximo de **AWS Managed Temporary Credentials** para desabilitá-lo e feche a página de preferencias.
4. Agora abra uma nova aba do navegador e vá para o console AWS, selecione o serviço IAM. Dentro do console do IAM crie um novo usuário (tenha certeza de habilitar a opção **Programmatic access** durante o processo), na seção **Set permisisons** selecione **Attach existing policies directly** e selecione a política **AdministratorAccess**, ao final do processo baixe o arquivo .csv contendo as credenciais.
5. Volte para o console do Cloud9. Na parte inferior da tela você deve ver um terminal do linux, digite o comando `aws configure` e aperte a tecla "enter". Cole o **AWS Access Key ID** obtido através do arquivo .csv gerado no passo anterior e aperte a tecla "enter", então cole o **AWS Secret Access Key** e aperte a tecla "enter", então na opção **Default region name** digite `us-east-2`. A seguir aperte a tecla "enter" até sair deste menu.
6. Agora você pode executar comandos via Cloud9 IDE utilizando as credenciais do usuário IAM criado anteriormente.
7. Vá para **Tarefa 1**.

---

###


## Tarefa 1 - <small>Criar a role para os webadmins</small>

Primeiramente você criará uma role para os webadmins (Inicialmente esta role irá ter o "trust" na sua própria conta AWS, porém na fase de **VERIFICAÇÃO** você irá configurar o "trust" para a conta de outro time):

* Para muitos dos passos abaixo você irá precisar do ID da sua conta AWS. Para saber o ID da sua conta AWS digite no terminal `aws sts get-caller-identity`. O ID da conta será o primeiro número listado depois de **Account**. Você pode executar este comando de um segundo terminal do Cloud9, desta forma você pode voltar nele sempre que precisar do ID da conta AWS.
* Utilize o JSON a seguir para criar um arquivo chamado **`trustpolicy1.json`** que será utilizado na "trust policy" (você pode utilizar o vim, nano, emacs ou outro editor de texto de sua preferência):
```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
    },
    "Action": "sts:AssumeRole"
  }
}
```
* Crie a Role do IAM que será utilizada pelos webadmins:
```
aws iam create-role --role-name webadmins --assume-role-policy-document file://trustpolicy1.json
```
* Anexe a policy AWSLambdaReadOnlyAccess gerenciada pela AWS à role: 
```
aws iam attach-role-policy --policy arn:aws:iam::aws:policy/AWSLambdaReadOnlyAccess --role-name webadmins
```


## Tarefa 2 - <small>Crie a política "permissions boundary" para os webadmins</small>

A seguir você criará a politica que será utilizada como "permissions boundary". A "permissions boundary" deve permitir somente as ações a seguir:  Create log groups, create log streams, put logs e listar os aqruivos dentro da pasta webadmins dentro do bucket que inicia com `"shared-logging-"` e termina com `"-data"`:

* Utilize o JSON a seguir para criar um arquivo chamado **`boundarydoc.json`** que será utilizado na politica "permissions boundary":
``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CreateLogGroup",
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:us-east-2:<ACCOUNT_ID>:*"
        },
        {
            "Sid": "CreateLogStreamandEvents",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:us-east-2:<ACCOUNT_ID>:log-group:/aws/lambda/*:*"
        },
        {
            "Sid": "AllowedS3GetObject",
            "Effect": "Allow",
            "Action": [
                "s3:List*"
            ],
            "Resource": "arn:aws:s3:::shared-logging-<ACCOUNT_ID>-us-east-2-data",
             "Condition": {
                "StringEquals": {
                    "s3:prefix": "webadmins"
             		}
        		}
        }		
    ]
}
```
* Crie a política:
```
aws iam create-policy --policy-name webadminspermissionsboundary --policy-document file://boundarydoc.json
```

## Task 3 - <small>Criar a política de permissões para os webadmins</small>

!!! Atenção 
	**IMPORTANTE!!!!** -  Os pontos de interrogação **`????`** na política abaixo devem ser alterados para algo que atuará como uma restrição na a criação dos recursos. Como resultado final teremos como requisito um "path" para as roles e políticas criadas pelos webadmins. Alterando os **`????`** é um dos desafios principais deste workshop e provavelmente a parte mais confusa. Fale com um facilitador se você tiver qualquer dificuldade com essa parte. 

A seguir você criará a política que será anexada a role utilizada pelos webadmins.

* Use o JSON a seguir para criar um arquivo chamado **`policydoc.json`** para a política de permissionamento dos webadmins:

``` json
{
    "Version": "2012-10-17",
    "Statement": [
		{
            "Sid": "CreateCustomerManagedPolicies",
            "Effect": "Allow",
            "Action": [
                "iam:CreatePolicy",
                "iam:DeletePolicy",
                "iam:CreatePolicyVersion",
                "iam:DeletePolicyVersion",
                "iam:SetDefaultPolicyVersion"
            ],
            "Resource": "arn:aws:iam::<ACCOUNT_ID>:policy/webadmins/????"
        },
        {
        	  "Sid": "RoleandPolicyActionswithnoPermissionBoundarySupport",
            "Effect": "Allow",
            "Action": [
            		"iam:UpdateRole",
                	"iam:DeleteRole"
            ],
            "Resource": [
                "arn:aws:iam::<ACCOUNT_ID>:role/webadmins/????"
            ]
        },
        {
            "Sid": "CreateRoles",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy"
            ],
            "Resource": [
                "arn:aws:iam::<ACCOUNT_ID>:role/webadmins/????"
            ],
            "Condition": {"StringEquals": 
                {"iam:PermissionsBoundary": "arn:aws:iam::<ACCOUNT_ID>:policy/webadminspermissionsboundary"}
            }
        },
        {
            "Sid": "LambdaFullAccess",
            "Effect": "Allow",
            "Action": "lambda:*",
            "Resource": "arn:aws:lambda:us-east-2:<ACCOUNT_ID>:function:*"
        },
        {
            "Sid": "PassRoletoLambda",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/webadmins/????",
            "Condition": {
                "StringLikeIfExists": {
                    "iam:PassedToService": "lambda.amazonaws.com"
                }
            }
        },
        {
            "Sid": "AdditionalPermissionsforLambda",
            "Effect": "Allow",
            "Action": 	["kms:ListAliases", "logs:Describe*", "logs:ListTagsLogGroup", "logs:FilterLogEvents", "logs:GetLogEvents"],
            "Resource": "*"
        },
        {
            "Sid": "DenyPermissionBoundaryandPolicyDeleteModify",
            "Effect": "Deny",
            "Action": [
                "iam:CreatePolicyVersion",
                "iam:DeletePolicy",
                "iam:DeletePolicyVersion",
                "iam:SetDefaultPolicyVersion"
            ],
            "Resource": [
                "arn:aws:iam::<ACCOUNT_ID>:policy/webadminspermissionsboundary",
                "arn:aws:iam::<ACCOUNT_ID>:policy/webadminspermissionpolicy"
            ]
        },
        {
            "Sid": "DenyRolePermissionBoundaryDelete",
            "Effect": "Deny",
            "Action": "iam:DeleteRolePermissionsBoundary",
            "Resource": "*"
        }
    ]
}
```
 
* Crie uma política chamada `webadminspermissionpolicy`:
```
aws iam create-policy --policy-name webadminspermissionpolicy --policy-document file://policydoc.json
```
* Anexe a política à role dos webadmins:
```
aws iam attach-role-policy --role-name webadmins --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/webadminspermissionpolicy
```
Quando você terminar a role **webadmins** deverá ter essas duas políticas anexadas: **webadminspermissionpolicy** & **AWSLambdaReadOnlyAccess**.


## Tarefa 4 - <small>Teste as permissões dos webadmins</small>
	
É hora de chacar o seu trabalho e garantir que as configurações/permissões dos webadmins estão corretas. Você precisa testar as seguintes ações como webadmins:	
1. Criar uma política do IAM (usando o perfil de webadmins)
2. Criar uma Role do IAM (e anexar a política criada como webadmins)
3. Criar uma função Lambda (e anexar a role criada como webadmins)

Se você não tiver certeza dos comandos para testar, você pode checar todos os detalhes na fase **[VERIFICAÇÃO](./verify.md)**. Quando você tiver validado que seu trabalho está correto, siga para o prossimo passo (**Tarefa 5**) que vai envolver a troca de informações com outro time para que o ele possa verificar o seu trabalho.

## Tarefa 5 - <small>Pegue as informações necessárias para a fase de **VERIFICAÇÃO**</small>

Agora é o momento de passar o trabalho para outro time que irá executar as taferas da fase de **VERIFICAÇÃO**. Se você está executando este workshop online por você mesmo então você pode pular esse passo e ir direto para a fase de **VERIFICAÇÃO**.

Se você recebeu um formulário para preencher, adicione os dados manualemente e troque com outro time. Se você seguiu todos os padrões de nomes apresentados durante os passos do workshop você só precisa adicionar seu **Account ID** e **Resource Restrictions**.

* Webadmins role ARN:	arn:aws:iam::`<YOUR_ACCOUNT_ID>`:role/**webadmins**
* Resource Restriction ("path" que deve ser utilizado para roles e políticas): /webadmins/**`Resource Restriction`**
* Permissions boundary name: **webadminspermissionsboundary**
* Permission policy name: **webadminspermissionpolicy**

Troque o formulário com outro time e então atualize a "trust policy" da role webadmins para que o outro time possa assumir a role através da conta dele (eles farão o mesmo para a conta do seu time)

* Use o JSON a seguir para criar um arquivo chamado `trustpolicy2.json` para a "trust policy". Altere `<YOUR_ACCOUNT_ID>` com o seu Account ID para que você continue conseguindo testar caso necessário (assumir a role webadmins a partir da sua conta) e o `<ACCOUNT_ID_FROM_OTHER_TEAM>` com o Account ID da conta do outro time:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::<YOUR_ACCOUNT_ID>:root",
          "arn:aws:iam::<ACCOUNT_ID_FROM_OTHER_TEAM>:root"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
* Atualize a "trust policy" para que ambos, você e o outro time possam assumir a role webadmins.
```
aws iam update-assume-role-policy --role-name webadmins --policy-document file://trustpolicy2.json
```

**[Click aqui para ir para a fase de VERIFICAÇÃO](./verify.md)**
