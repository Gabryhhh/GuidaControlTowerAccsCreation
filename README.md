# Provisioning di Account AWS con Control Tower tramite Terraform

Questo repository contiene script Terraform per la creazione e gestione di account AWS utilizzando AWS Control Tower Account Factory in modo programmatico. Utilizzando Terraform, è possibile creare, personalizzare e gestire account AWS in maniera scalabile e ripetibile.

## Prerequisiti

- AWS CLI installata e configurata
- jq installato (processore JSON)
- Terraform installato

## Dipendenze

Assicurati di avere le seguenti dipendenze installate:

### AWS CLI

Per installare AWS CLI, segui la [guida ufficiale per l'installazione di AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

### jq

jq è un processore JSON leggero e flessibile per la linea di comando. Per installare `jq`, esegui:

```sh
sudo yum install jq -y
```

stallare Terraform, segui la [guida ufficiale per l'installazione di Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli).

## Configurazione

Prima di eseguire gli script Terraform, configura le variabili d'ambiente e le impostazioni AWS.

1. **Definisci la regione AWS:**

    ```sh
    export REGION=<tua-regione-1>
    ```

2. **Imposta l'ID dell'account master:**

    ```sh
    export MASTER_ACCT=$(aws sts get-caller-identity --query 'Account' --output text)
    ```

3. **Definisci l'ARN dell'Admin:**

    ```sh
    export ADMIN_ARN="arn:aws:iam::${MASTER_ACCT}:role/service-role/AWSControlTowerStackSetRole"
    ```

4. **Crea un token casuale:**

    ```sh
    export RANDOM_TOKEN=$(echo $(( $RANDOM * 99999999999 )) | cut -c 1-13)
    ```

5. **Definisci l'ID del prodotto per Account Factory:**

    ```sh
    export PROD_ID=$(aws servicecatalog search-products --filters fullTextSearch='AWS Control Tower Account Factory' --region $REGION --query "ProductViewSummaries[*].ProductId" --output text)
    ```

6. **Identifica l'ID dell'artifatto di provisioning:**

    ```sh
    export PA_ID=$(aws servicecatalog describe-product --id $PROD_ID --region $REGION --query "ProvisioningArtifacts[-1].Id" --output text)
    ```

7. **Crea un file JSON con i parametri necessari:**

    Esempio di file `params.json`:

    ```json
    [
      {
        "Value": "aws@example.com",
        "Key": "SSOUserEmail"
      },
      {
        "Value": "Sample",
        "Key": "SSOUserFirstName"
      },
      {
        "Value": "User",
        "Key": "SSOUserLastName"
      },
      {
        "Value": "Development (ou-aaaa-cccccccc)",
        "Key": "ManagedOrganizationalUnit"
      },
      {
        "Value": "SampleAccount",
        "Key": "AccountName"
      },
      {
        "Value": "aws@example.com",
        "Key": "AccountEmail"
      }
    ]
    ```

8. **Esporta i valori dal file JSON:**

    ```sh
    export CATALOG_NAME='CatalogFor'$(jq -r .[4].Value params.json)
    export EMAIL_ID=$(jq -r .[0].Value params.json)
    ```

## Creazione degli Account

Utilizza il comando seguente per creare gli account in modo programmatico:

```sh
aws servicecatalog provision-product --product-id $PROD_ID --provisioning-artifact-id $PA_ID --provision-token $RANDOM_TOKEN --provisioned-product-name $CATALOG_NAME --provisioning-parameters file://params.json
```

### Monitoraggio dello Stato di Creazione dell'Account

Il processo di creazione dell'account potrebbe richiedere del tempo. Utilizza lo script seguente per monitorare lo stato della creazione ogni 30 secondi:

```sh
STATUS=$(aws servicecatalog scan-provisioned-products --region $REGION | jq -r '.ProvisionedProducts[] | select(.Name == env.CATALOG_NAME).Status')

while [[ $STATUS != "AVAILABLE" ]]
do
  echo "Stato di provisioning trovato: $STATUS .. Attesa di 30 secondi"
  sleep 30
  STATUS=$(aws servicecatalog scan-provisioned-products --region $REGION | jq -r '.ProvisionedProducts[] | select(.Name == env.CATALOG_NAME).Status')
done
```

### Query dell'Organizzazione per l'ID dell'Account

Una volta completata la creazione, esegui una query per ottenere l'ID dell'account associato all'email specificata nel file dei parametri:

```sh
NEW_ACCT=$(aws organizations list-accounts | jq -r '.Accounts[] | select(.Email == env.EMAIL_ID).Id')
echo "Il nuovo account è: $NEW_ACCT"
```

### Creazione di Stack Set per Ruoli di Automazione

Dopo aver creato l'account, è possibile utilizzare AWS CloudFormation per applicare personalizzazioni:

1. **Creazione di uno Stack Set:**

    ```sh
    aws cloudformation create-stack-set --stack-set-name CreateAdditionalRoles --parameters "[{\"ParameterKey\":\"AdminAccountId\",\"ParameterValue\":\"$MASTER_ACCT\"},{\"ParameterKey\":\"AdminRoleName\",\"ParameterValue\":\"SampleAdminRole\"},{\"ParameterKey\":\"ExecutionRoleName\",\"ParameterValue\":\"SampleExecutionRole\"},{\"ParameterKey\":\"sessionDurationInSecs\",\"ParameterValue\":\"14400\"}]" --template-url "https://automation-roles.s3.us-east-2.amazonaws.com/automation-exec-role.yaml" --administration-role-arn $ADMIN_ARN --execution-role-name AWSControlTowerExecution --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
    ```

2. **Creazione di istanze dello Stack nel nuovo account:**

    ```sh
    aws cloudformation create-stack-instances --stack-set-name CreateAdditionalRoles --regions $REGION --accounts $NEW_ACCT --operation-preferences FailureToleranceCount=0,MaxConcurrentCount=3
    ```

## Verifica

Aggiorna la pagina "Accounts" in AWS Control Tower per verificare che il nuovo account sia stato aggiunto correttamente. Verifica anche che l'account sia elencato in AWS Single Sign-On e che il ruolo di esecuzione dell'automazione sia presente nelle Trust Relationships.

## Conclusione

Utilizzando questo script Terraform, è possibile automatizzare la creazione e la gestione degli account AWS tramite AWS Control Tower, migliorando la scalabilità e la conformità della tua infrastruttura AWS.
