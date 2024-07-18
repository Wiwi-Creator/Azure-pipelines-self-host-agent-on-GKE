# Azure pipelines Use Self-host Image

希望Azure Pipelines可以使用我們預設的環境(Dockerfile)

因為Azure Pipelines Default的Agent是 Microsoft-host Agent(該Agent在每次執行時都會重新清環境)

所以執行CICD時 出現 "Command Not Found"(EX: Dagster , dbt..etc) 

目前有兩種方法
 - 1.Set up a service service-connection for Azure Pipelines to GCP
   - Create a service account
   - Grant permission
   - Generate a service account key
   - Azure pipelines Service-connection setting 

 - 2.Build Self-host agent for Azure pipelines

## Set up a service account for Azure Pipelines
- Create a service account for Azure Pipelines:
``` bash
AZURE_PIPELINES_SERVICE_ACCOUNT=$(gcloud iam service-accounts create azure-pipelines --format "value(email)")
```

- Grant the (roles/editor') role to the azure-pipelines service account so that Azure Pipelines can use:

``` bash
gcloud projects add-iam-policy-binding datapool-1806 \
  --member serviceAccount:$azure-devops-account@datapool-1806.iam.gserviceaccount.com \
  --role='roles/editor'
```

- Generate a service account key
``` bash
gcloud iam service-accounts keys create azure-pipelines-key.json \
  --iam-account=$AZURE_PIPELINES_SERVICE_ACCOUNT

cat azure-pipelines-key.json | base64 -w 0;echo

rm azure-pipelines-key.json
```

- Azure pipelines Service-connection setting

在 Azure Pipelines 中，為容器註冊表建立新的服務連線：

1.在 Azure DevOps選單中，選擇「專案設定」，然後選擇「管道」  > “服務連線”。

2.按一下建立服務連線。

3.從清單中選擇Docker 註冊表，然後按一下下一步。

4.在對話方塊中，輸入以下欄位的值：
註冊表類型:其他
- Docker Registry：，替換為您的生產項目的名稱。https://gcr.io/PROD_PROJECT_IDPROD_PROJECT_IDhttps://gcr.io/azure-pipelines-test-project-12345
- Docker ID：_json_key
- 密碼：貼上 的內容azure-pipelines-publisher-oneline.json。
- Service connection name：gcr-tutorial
- 按一下“儲存”以建立連線。

使用 Azure Pipelines 來設定CICD。對於推送到 Git 儲存庫的每個提交，Azure Pipelines 都會建置程式碼並將建置專案打包到 Docker 容器中。然後將容器發佈到 Container Registry。


----------------------------------------------------------------------------------

## Build Self-host agent for Azure pipelines
- Create agent-pool
   
   UI -> Project Setting -> Agebt Pools > New Agent Pool

- Dockerfile -> azp-agent-linux/dockerfile

- Build Image (*注意* 將dockerfile內容修改為自己希望的環境)

``` bash
docker build --tag "azp-agent:linux" --file "./azp-agent-linux.dockerfile" .
```
- Docker push to Google Artifact Registry

``` bash
docker push "${HOSTNAME}/${PROJECT_ID}/${REPOSITORY}/${IMAGE}:${TAG}"
```

- Run self-host agent in a Docker

``` bash
docker run -e AZP_URL=${AZP_URL} -e AZP_TOKEN="${AZP_TOKEN} -e AZP_POOL=${AZP_POOL} -e AZP_AGENT_NAME=${AZP_AGENT_NAME} --name "azp-agent-linux" azp-agent:linux
```

*註* : AZP_TOKEN 最長久僅一年 , 修改agent_info中的 Token即可

- Azure Pipelines Yaml

``` yaml
trigger:
- master

pool:
  name: <pool_name>
  vmImage: <agent_name>

steps:
- script: echo Hello, world!
  displayName: 'Run a one-line script'
```

## Self-host agent deploy on kubernetes

既然有了 Image 當然也可以部署在 Kubernetes上。
以提升這個Agent他的Scalability , Flexibility 
或是因應效能提升其資源調度。

- Pod or deployment.yaml
- Make your variables in kubernetes secret
  
## Refence 

[Azure-self-host-agent](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml,browser)

[Azure-self-host-agent-on-Docker](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops)

[Azure-self-host-agent-on-Kubernetes](https://medium.com/@muppedaanvesh/azure-devops-self-hosted-agents-on-kubernetes-part-1-aa91e7912f79)