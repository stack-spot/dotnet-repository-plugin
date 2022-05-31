## **Visão Geral**
O **dotnet-repository-plugin** adiciona em uma stack a capacidade de provisionar o uso do Amazon DynamoDB seja recuperando, salvando ou apagando entidades.

#### **Pré-requisitos**
Para utilizar esse plugin é necessário ter uma stack dotnet criada pelo `CLI` do `StackSpot` que você pode baixar [**aqui**](https://stackspot.com/).

Ter instalado:
- .NET 5 ou 6 
- O template `dotnet-api-template` ou o `dotnet-worker-template` deverá estar aplicado para você conseguir utilizar este plugin.

#### **Inputs**

* RegionEndpoint - Endpoint regional que será utilizado para requisitar o DynamoDB - Campo Obrigatório.

Você pode sobrescrever a configuração padrão adicionando a seção `DynamoDB` em seu `appsettings.json`. Os valores aceitáveis você pode encontrar [aqui](https://docs.aws.amazon.com/pt_br/pt_br/AWSEC2/latest/WindowsGuide/using-regions-availability-zones.html#concepts-available-regions).

```json
  "DynamoDB": {
      "RegionEndpoint": "sa-east-1"
  }
```

Adicione ao seu `IServiceCollection` via `services.AddDynamoDB()` no `Startup` da aplicação ou `Program` tendo como parêmetro de entrada `IConfiguration` e `IWebHostEnvironment`. 

```csharp
services.AddDynamoDB(Configuration);
```

#### **Operações**

* A classe que será utilizada para o mapeamento da tabela deverá ter a anotação `DynamoDBTable`, é aqui que definimos o nome da tabela. 
* A chave de partição e a chave de classificação(se presente) devem ter, respectivamente, as anotações no nível de propriedade `DynamoDBRangeKey` e `DynamoDBRangeKey`.
* Propriedades regulares devem ser marcadas com a anotação `DynamoDBProperty`, que aceita um parâmetro opcional para especificar um nome para a coluna da tabela distinto do nome da propriedade.

```csharp
[DynamoDBTable("Test")]
public class Test
{
    //Chave de Partição 
    [DynamoDBHashKey]
    public string Id { get; set; }

    //Chave de Classificação
    [DynamoDBRangeKey]
    public string IdTest { get; set; }    

    [DynamoDBProperty]
    public string Name { get; set; }

    [DynamoDBProperty]
    public string LastName { get; set; }

    [DynamoDBProperty("Emails")]
    public List<string> TestEmails { get; set; }    
}
```

*  SaveAsync - Utilizado para criar ou atualizar a entidade na tabela(Se a entidade existir na tabela, ele atualizada, do contrário ela é criada).

```csharp
public class TestRepository : ITestRepository
    { 
        private readonly IDynamoDB _dynamoDB;

        public TestRepository(IDynamoDB dynamoDB)
        {
            _dynamoDB = dynamoDB;
        }

        public async Task<Test> AddAsync(Test entity)
        {
            await _dynamoDB.SaveAsync(entity);

            return entity;
        }
    }
```

*  QueryAsync - Utilizado para recuperar uma ou mais entidades. Ele aceita apenas um parâmetro `QueryConfig`, que é utilizado como filtro.

```csharp
public class TestRepository : ITestRepository
{ 
    private readonly IDynamoDB _dynamoDB;

    public TestRepository(IDynamoDB dynamoDB)
    {
        _dynamoDB = dynamoDB;
    }

    public async Task<Test> GetByIdAsync(string id, string idTest)
    {
        var queryFilter = new QueryFilter();
        queryFilter.AddCondition("Id", QueryOperator.Equal, id);
        queryFilter.AddCondition("IdTest", QueryOperator.Equal, idTest);

        var queryResult = await _dynamoDB.QueryAsync<Test>(new QueryConfig(queryFilter));

        return queryResult.FirstOrDefault();
    }
}
```

*  LoadAsync - Utilizado para recuperar uma entidade. Existem duas opções, uma utilizando a própria entidade a ser recuperada e outra utilizando a chave de partição e a chave de classificação(caso a tabela tenha).

```csharp
public class TestRepository : ITestRepository
{ 
    private readonly IDynamoDB _dynamoDB;

    public TestRepository(IDynamoDB dynamoDB)
    {
        _dynamoDB = dynamoDB;
    }

    public async Task<Test> LoadAsync(Test test)
    {
      await _dynamoDB.LoadAsync(test);
    }

    public async Task<Test> LoadAsync(string id, string idTest)
    {
      return await _dynamoDB.LoadAsync<Test>(id,idTest);
    }           
}
```

*  DeleteAsync - Utilizado para excluir fisicamente uma entidade. Existem duas opções, uma utilizando a própria entidade a ser excluída e outra utilizando a chave de partição e a chave de classificação(necessário caso a tabela tenha).

```csharp
public class TestRepository : ITestRepository
{ 
    private readonly IDynamoDB _dynamoDB;

    public TestRepository(IDynamoDB dynamoDB)
    {
        _dynamoDB = dynamoDB;
    }

    public async Task RemoveAsync(Test test)
    {
      await _dynamoDB.DeleteAsync(test);
    }

    public async Task RemoveAsync(string id, string idTest)
    {
      await _dynamoDB.DeleteAsync<Test>(id,idTest);
    }           
}
```

#### 4. Ambiente local

* Esta etapa não é obrigatória.
* Recomendamos, para o desenvolvimento local, a criação de um contâiner com a imagem do [Localstack](https://github.com/localstack/localstack). 
* Para o funcionamento local você deve preencher a variável de ambiente `LOCALSTACK_CUSTOM_SERVICE_URL` com o valor da url do serviço. O valor padrão do localstack é http://localhost:4566.
* Abaixo um exemplo de arquivo `docker-compose` com a criação do contâiner: 

```yaml
version: '2.1'

services:
  localstack:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      - SERVICES=dynamodb
      - AWS_DEFAULT_OUTPUT=json
      - DEFAULT_REGION=sa-east-1
```

Após a criação do contâiner, crie uma tabela para realizar os testes com o componente. Recomendamos que você tenha instalado em sua estação o [AWS CLI](https://aws.amazon.com/pt/cli/). Abaixo um exemplo de comando para criação de uma tabela:

```bash
aws --endpoint-url=http://localhost:4566 --region=sa-east-1 dynamodb create-table --table-name [NOME DA TABELA] --attribute-definitions AttributeName=[NOME DO ATRIBUTO],AttributeType=[TIPO DO ATRIBUTO] --key-schema AttributeName=[NOME DO ATRIBUTO],KeyType=[TIPO DA KEY] --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```
