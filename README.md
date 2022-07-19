## **Visão Geral**
O plugin **`dotnet-repository-plugin`** adiciona em uma Stack a capacidade de provisionar o uso do **Amazon DynamoDB** seja recuperando, salvando ou apagando entidades.

#### **Pré-requisitos**
Para utilizar este plugin é preciso ter instalado na sua máquina os itens abaixo:  

- Uma Stack **DotNET** criada pelo [**STK CLI**](https://stackspot.com/);  
- .NET 5 ou 6 
- O template `dotnet-api-template` ou o `dotnet-worker-template` já instalados.

#### **Inputs configurados automaticamente**  
- **`RegionEndpoint`**  
É o endpoint regional que será utilizado para requisitar o **DynamoDB**. 

É possível sobrescrever a configuração padrão adicionando a seção `DynamoDB` no seu **`appsettings.json`**. Confira abaixo:  

```json
  "DynamoDB": {
      "RegionEndpoint": "sa-east-1"
  }
```

> Os valores aceitáveis são encontrados [aqui](https://docs.aws.amazon.com/pt_br/pt_br/AWSEC2/latest/WindowsGuide/using-regions-availability-zones.html#concepts-available-regions).


- A configuração abaixo será feita no `IServiceCollection`, através do `services.AddDynamoDB()`, no `Startup` da aplicação ou `Program`. Ela terá **`IConfiguration`** e **`IWebHostEnvironment`** como parâmetros de entrada. 

```csharp
services.AddDynamoDB(Configuration);
```

#### **Exemplos de aplicação do plugin**  

- A classe que será utilizada a seguir para o mapeamento da tabela deverá ter a anotação `DynamoDBTable` que definirá o nome da tabela;   
- A chave de partição e a chave de classificação (se houver) devem ter, respectivamente, as anotações no nível de propriedade **`DynamoDBRangeKey`** e **`DynamoDBRangeKey`**;  
- As propriedades regulares deverão ser marcadas com a anotação **`DynamoDBProperty`**, que aceita um parâmetro opcional para especificar um nome para a coluna da tabela distinto do nome da propriedade.

Confira abaixo alguns exemplos de aplicação do plugin:  

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

- **`SaveAsync`**  
É utilizado para criar ou atualizar a entidade na tabela. Se a entidade estiver na tabela, ela é atualizada. Se não, a tabela é criada. Confira a execução abaixo:  

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

- **`QueryAsync`**
É utilizado para recuperar uma ou mais entidades. Ele aceita apenas um parâmetro: **`QueryConfig`**, que é utilizado como filtro. Confira a execução abaixo:  

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

- **`LoadAsync`**  
Utilizado para recuperar uma entidade. Para isso existem duas opções:

1. Utilizando a própria entidade a ser recuperada;
2. Utilizando a chave de partição e a chave de classificação (caso a tabela tenha).

Confira a execução abaixo:  

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

- **`DeleteAsync`**
É utilizado para excluir fisicamente uma entidade. Para isso existem duas opções:

1. Itilizando a própria entidade a ser excluída;
2. Utilizando a chave de partição e a chave de classificação (caso a tabela tenha).  

Confira a execução abaixo:  

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

#### **Execução em Ambiente local**  

Para que o plugin funcione localmente, siga os passos abaixo:

1. Crie um contêiner com a imagem do imagem do [LocalStack](https://github.com/localstack/localstack);  
2. Preencha a variável de ambiente **`LOCALSTACK_CUSTOM_SERVICE_URL`** com o valor da URL do serviço. O valor padrão do LocalStack é **http://localhost:4566**.

Confira baixo um exemplo de arquivo `docker-compose` depois da criação do contêiner:  

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

Depois que o contêiner foi criado, crie uma tabela para fazer os testes com o componente. 

É recomendado que você tenha instalado em sua estação o [**AWS CLI**](https://aws.amazon.com/pt/cli/). 

Confira abaixo um exemplo de comando para criar uma tabela:  

```bash
aws --endpoint-url=http://localhost:4566 --region=sa-east-1 dynamodb create-table --table-name [NOME DA TABELA] --attribute-definitions AttributeName=[NOME DO ATRIBUTO],AttributeType=[TIPO DO ATRIBUTO] --key-schema AttributeName=[NOME DO ATRIBUTO],KeyType=[TIPO DA KEY] --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```