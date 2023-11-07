# Script passo a passo dos Checkpoints IaC



### Checkpoint 05 - 2º Semestre

### Passo-a-passo

01. Fazer um **Fork** do repositório [app-notifier](https://github.com/kledsonhugo/notifier) 

02. Clone em uma pasta o repositório 

```
git clone https://github.com/NaderSouza/app-notifier.git
```

03. Crie a **Branch** develop

```
git checkout -b develop 
```

04. Entrar na **AWS** e dar start o **LAB** e pegar as credencias e colocar no **GitHub Secrets** e autorizar o workflows no **Actions**

![secrets](/images/secret.png)


5. Ajustar o código do Terraform para uma estrutura em módulos, contendo dois módulos (rede,data e compute).


6. Crias a pasta modules com as pastas **rede**, **dados** e **compute**


7. Criar os arquivos de acordo com a imagem abaixo: 

![files](/images/files.png)



8. Criar na AWS o **S3** e **DynamoDB**  coloque um nome que seja fácil para reconhecer


9. Criar agora na pasta **terraform** o **provider.tf** com esses códigos

<br>

```
# PROVIDER
terraform {

  required_version = "~> 1.5.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.13"
    }
  }

  backend "s3" {
    bucket         = "terraform-state-lock-nadin"
    key            = "terraform.tfstate"
    dynamodb_table = "terraform-state-lock-nadin"
    region         = "us-east-1"
  }

}
```

10. Criar agora na pasta **terraform** o **notifier.tf** com esses códigos:

```
# MODULES ORCHESTRATOR

module "rede" {
    source               = "./modules/rede"
    vpc_cidr             = "10.0.0.0/16"
    vpc_az1              = "${var.vpc_az1}"
    vpc_az2              = "${var.vpc_az2}"
    vpc_sn_pub_az1_cidr  = "${var.vpc_sn_pub_az1_cidr}"
    vpc_sn_pub_az2_cidr  = "${var.vpc_sn_pub_az2_cidr}"
    vpc_sn_priv_az1_cidr = "${var.vpc_sn_priv_az1_cidr}"
    vpc_sn_priv_az2_cidr = "${var.vpc_sn_priv_az2_cidr}"
}

module "dados" {
    source               = "./modules/dados"
    rds_identifier       = "${var.rds_identifier}"
    rds_engine_version   = "${var.rds_engine_version}"
    rds_sn_group_name    = "${var.rds_sn_group_name}"
    rds_param_group_name = "${var.rds_param_group_name}"
    rds_dbname           = "${var.rds_dbname}"
    rds_dbuser           = "${var.rds_dbuser}"
    rds_dbpassword       = "${var.rds_dbpassword}"
    vpc_sn_priv_az1_id   = "${module.rede.vpc_sn_priv_az1_id}"
    vpc_sn_priv_az2_id   = "${module.rede.vpc_sn_priv_az2_id}"
    vpc_sg_priv_id       = "${module.rede.vpc_sg_priv_id}"
}

module "compute" {
    source                   = "./modules/compute"
    ec2_lt_name              = "${var.ec2_lt_name}"
    ec2_lt_ami               = "${var.ec2_lt_ami}"
    ec2_lt_instance_type     = "${var.ec2_lt_instance_type}"
    ec2_lt_ssh_key_name      = "${var.ec2_lt_ssh_key_name}"
    ec2_lb_name              = "${var.ec2_lb_name}"
    ec2_lb_tg_name           = "${var.ec2_lb_tg_name}"
    ec2_asg_name             = "${var.ec2_asg_name}"
    ec2_asg_desired_capacity = "${var.ec2_asg_desired_capacity}"
    ec2_asg_min_size         = "${var.ec2_asg_min_size}"
    ec2_asg_max_size         = "${var.ec2_asg_max_size}"
    vpc_cidr                 = "${var.vpc_cidr}"
    vpc_id                   = "${module.rede.vpc_id}"
    vpc_sn_pub_az1_id        = "${module.rede.vpc_sn_pub_az1_id}"
    vpc_sn_pub_az2_id        = "${module.rede.vpc_sn_pub_az2_id}"
    vpc_sg_pub_id            = "${module.rede.vpc_sg_pub_id}"
    rds_endpoint             = "${module.dados.rds_endpoint}"
    rds_dbuser               = "${var.rds_dbuser}"
    rds_dbpassword           = "${var.rds_dbpassword}"
    rds_dbname               = "${var.rds_dbname}"
}

```


> **Note**: Não esqueça de trocar a AMI do **vars.tf** para a mesma do **notifier.tf**



11. Criar o pipeline de **Apply** - Crie uma pasta **.github/workflows** e dentro dela crie o arquivo **pipe.yaml**



12. Coloque o código do **Pipeline Apply** dessa forma:

```
name: IaC Checkpoint

on:
  push:
    branches:
    - develop

jobs:

  job-Terraform_Apply:
    runs-on: ubuntu-latest
 
    steps:

    - name: Step 01 - Terraform Install
      env :
        TERRAFORM_VERSION: "1.5.6"
      run : |
        tf_version=$TERRAFORM_VERSION
        wget https://releases.hashicorp.com/terraform/"$tf_version"/terraform_"$tf_version"_linux_amd64.zip
        unzip terraform_"$tf_version"_linux_amd64.zip
        sudo mv terraform /usr/local/bin/
    - name: Step 02 - Terraform Version
      run : terraform --version

    - name: Step 03 - CheckOut GitHub Repo
      uses: actions/checkout@v1

    - name: Step 04 - Set AWS Account
      uses: aws-actions/configure-aws-credentials@v1-node16
      with:
        aws-access-key-id    : ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-session-token    : ${{ secrets.AWS_SESSION_TOKEN }}
        aws-region           : us-east-1

    - name: Step 05 - Terraform Init
      run : terraform -chdir=./terraform init -input=false

    - name: Step 06 - Terraform Validate
      run : terraform -chdir=./terraform validate

    - name: Step 07 - Terraform Plan
      run : terraform -chdir=./terraform plan -input=false -out tfplan
      # run : terraform -chdir=./terraform plan -input=false -destroy -out tfplan

    - name: Step 08 - Terraform Apply
      run : terraform -chdir=./terraform apply -auto-approve -input=false tfplan

    - name: Step 09 - Terraform Show
      run : terraform -chdir=./terraform show
```
<br>

13. Fazer o comandos para subir o repositório

<br>

```
git add .
```
```
git commit -m "cp-2"
```
```
git push origin develop
```
> **Note**: Se não aparecer o **Actions**, suba mais um **commit** - exemplo: dar algum espaçamento em algum arquivo e salvar 


14. Veja se no **Github Actions** subiu corretamente o pipeline 
<br>
<br>
15. Alterar o código do pipeline para o **Checkov** - dentro dela o arquivo altere o nome do **pipe.yaml** para algum outro para saber diferenciar e coloque esse código

```
name: Checkov Pipeline

on:
  push:
    branches:
    - develop

jobs:

  tf-run:
    name: Terraform Run
    runs-on: ubuntu-latest
 
    steps:

    - name: Step 01 - Terraform Setup
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.6

    - name: Step 02 - Terraform Version
      run : terraform --version

    - name: Step 03 - CheckOut GitHub Repo
      uses: actions/checkout@v3

    - name: Step 04 - Set AWS Account
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id    : ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-session-token    : ${{ secrets.AWS_SESSION_TOKEN }}
        aws-region           : us-east-1

    - name: Step 05 - Terraform Init
      run : terraform -chdir=./terraform init -input=false

    - name: Step 06 - Terraform Unit Tests
      run : terraform -chdir=./terraform validate

    - name: Step 07 - Terraform Plan with Contract Tests
      run : terraform -chdir=./terraform plan -input=false -out tfplan
      # run : terraform -chdir=./terraform plan -input=false -destroy -out tfplan

    - name: Step 08 - Terraform Security Scannning Setup
      run: pip3 install checkov

    - name: Step 09 - Terraform Security Scannning Run
      run: checkov --directory ./terraform

    - name: Step 10 - Terraform Apply
      run : terraform -chdir=./terraform apply -auto-approve -input=false tfplan

    - name: Step 11 - Terraform Show
      run : terraform -chdir=./terraform show 
```

16. Fazer o comandos para subir o repositório

```
git add .
```
```
git commit -m "cp-2"
```
```
git push origin develop
```

17. Veja se no **Github Actions** subiu corretamente o pipeline 
