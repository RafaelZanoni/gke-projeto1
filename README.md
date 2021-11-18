Para habilitar as APIs de container e sql admin:

gcloud services enable container.googleapis.com sqladmin.googleapis.com

Para setar uma zona de sua preferência:

gcloud config set compute/zone us-east1-b

Para definir o nome de seu projeto como uma variável de ambiente:

export PROJECT_ID=projeto-gcp-mygithub

Para clonar um repositório do GitHub:

git clone https://github.com/RafaelZanoni/gke-projeto1

Para abrir o diretório de onde foi clonado:

cd gke-projeto1/

Para definir o comando pwd como uma variável de ambiente (pwd mostra o diretório atual):

WORKING_DIR=$(pwd)

Vamos definir um nome para o Cluster, criá-lo com as seguintes configurações: 3 nós e com upgrade automático:

CLUSTER_NAME=persistent-disk
gcloud container clusters create $CLUSTER_NAME \
    --num-nodes=3 --enable-autoupgrade --no-enable-basic-auth \
    --no-issue-client-certificate --enable-ip-alias --metadata \
    disable-legacy-endpoints=true

Iremos aplicar o manifesto do kubernetes referente ao Persistent Volume Claim:

kubectl apply -f $WORKING_DIR/wordpress-volumeclaim.yaml
Esse comando visualiza o estado de provisionamento do PVC:

kubectl get persistentvolumeclaim


Iremos setar o nome da instância SQL e criá-la no Cloud SQL:

INSTANCE_NAME=mysql-wordpress-instance
gcloud sql instances create $INSTANCE_NAME


E em seguida, setar o nome da instância como uma variável de ambiente:


export INSTANCE_CONNECTION_NAME=$(gcloud sql instances describe $INSTANCE_NAME \
    --format='value(connectionName)')

Vamos criar uma database no SQL para que o Wordpress possa armazenar seus dados:

gcloud sql databases create wordpress --instance $INSTANCE_NAME


Criaremos um usuário e senha no banco de dados para que que o Wordpress consiga autenticá-la:

CLOUD_SQL_PASSWORD=ipnet2021
gcloud sql users create wordpress --host=% --instance $INSTANCE_NAME \
    --password $CLOUD_SQL_PASSWORD


Para que seja possível o Wordpress acessar o SQL via proxy do Cloud SQL, é necessário criar uma conta de serviço:

SA_NAME=cloudsql-proxy
gcloud iam service-accounts create $SA_NAME --display-name $SA_NAME


Esse comando fará com que a conta de serviço seja definida como uma variável de ambiente e em formato de e-mail:

SA_EMAIL=$(gcloud iam service-accounts list \
    --filter=displayName:$SA_NAME \
    --format='value(email)')


Vamos adicionar a role cloudsql.client na nossa conta de serviço:

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --role roles/cloudsql.client \
    --member serviceAccount:$SA_EMAIL


É necessário criar uma chave json para a conta de serviço:

gcloud iam service-accounts keys create $WORKING_DIR/key.json \
    --iam-account $SA_EMAIL


Criaremos um secret do Kubernetes para as credenciais do MySQL:

kubectl create secret generic cloudsql-db-credentials \
    --from-literal username=wordpress \
    --from-literal password=$CLOUD_SQL_PASSWORD


Agora vamos criar um secret para a conta de serviço:

kubectl create secret generic cloudsql-instance-credentials \
    --from-file $WORKING_DIR/key.json


Agora iremos aplicar o Wordpress, vamos substituir a variável de ambiente INSTANCE_CONNECTION_NAME:

cat $WORKING_DIR/wordpress_cloudsql.yaml.template | envsubst > \
    $WORKING_DIR/wordpress_cloudsql.yaml
E vamos aplicar o arquivo yaml que foi substituído:

kubectl create -f $WORKING_DIR/wordpress_cloudsql.yaml


Vamos analisar a criação do contêiner:

kubectl get pod -l app=wordpress --watch


Iremos criar o serviço de LoadBalancer utilizando este manifesto:

kubectl create -f $WORKING_DIR/wordpress-service.yaml


Vamos verificar sua criação e obter seu endereço de IP externo:

kubectl get svc -l app=wordpress --watch




