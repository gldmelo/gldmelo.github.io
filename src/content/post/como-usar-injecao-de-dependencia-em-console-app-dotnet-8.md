---
publishDate: 2024-10-27T00:00:00Z
author: Guilherme Melo
title: Como usar Injeção de Dependência em aplicações Console com o .NET 8
excerpt: Comece a utilizar injeção de dependência em seus projetos .NET hoje mesmo!
image: /postimages/injecao-dependencia.png
category: artigos
tags:
  - dotnet
  - .net
  - csharp
  - design patterns
  - injeção de dependência
  - dependency injection
  - ioc
metadata:
  canonical: https://www.gldmelo.com.br/como-usar-injecao-de-dependencia-em-console-app-dotnet-8
---

A Injeção de Dependência, também chamada de Dependency Injection ou simplesmente DI é uma técnica que visa reduzir o acoplamento entre classes, tornando o código mais modular, e facilitando a manutenção e testes individuais dos componentes.
Esta técnica faz parte de um padrão de projetos chamada Inversão de Controle ou (Inversion of Control ou apenas IoC), onde a responsabilidade de criar e gerenciar essas dependências é feita por um componente externo ao invés da própria classe.

## Como configurar a Injeção de Dependência na linguagem C# com .NET 8

Nas versões mais atuais utilizamos o nuget ou editamos diretamente o arquivo .csproj de nosso projeto para referenciar o pacote que vai nos ajudar a fazer isso do jeito .NET

Nuget:
```csharp
Microsoft.Extensions.DependencyInjection
```

no arquivo .csproj:
```csharp
<PackageReference Include="Microsoft.Extensions.DependencyInjection"
 Version="8.0.1" />
```

## Implementação da Injeção de Dependência

Vamos criar algumas classes para explicar o conceito. Considere as classes e interfaces a seguir:

Classe User que representa um usuário hipotético contendo apenas o nome e a idade.

```csharp
public class User(string name, int age)
{
	public string Name { get; set; } = name;
	public int Age { get; set; } = age;
}
```

Uma interface que define quais métodos um Repositório de usuários deve implementar. A classe UserRepository deve implementar essa interface trazendo o código das operações nela descritas.
Utilizei um array para simplificar o exemplo mas imagine que é um acesso a banco de dados =)

```csharp
public interface IUserRepository
{
	List<User> GetUsers();
}

public class UserRepository : IUserRepository
{
	public List<User> GetUsers()
	{
		return
		[
			new("Fulano", 30),
			new("Beltrano", 25)
		];
	}
}
```

Depois disso temos a interface IUserService e UserService, de forma análoga vão trazer métodos que uma classe de serviços para usuários deve implementar. Esse tipo de classe deve orquestrar operações em cima de usuários atendendo os métodos da interface. No nosso exemplo ela funciona apenas como uma classe de acesso ao repositório escondendo ele da aplicação principal (façade).

Não está contemplado nesse exemplo, mas considere que neste nível é um local onde é possível fazer alguma verificação se o usuário que está tentando acessar a lista de usuários possui permissão para fazer isso.

```csharp
public interface IUserService
{
	List<User> GetUsers();
}

public class UserService(IUserRepository userRepository) : IUserService
{
	public List<User> GetUsers()
	{
		return userRepository.GetUsers();
	}
}
```

Agora por fim vamos juntar as partes do nosso .NET Console Application e configurar a Injeção de Dependência!


Na classe Program.cs do Console Application coloque o código a seguir. O que isso faz? Estamos configurando um ServiceCollection. Ele é responsável por guardar os mapeamentos das interfaces para suas respectivas implementações concretas, sendo o único ponto do sistema que deve conhecer essas classes. Ao final usamos o método BuildServiceProvider() que vai nos retornar o Service Provider, que é o objeto que vamos utilizar sempre que quisermos um objeto que segue uma interface sem conhecer qual é a classe de fato!

```csharp
var serviceCollection = new ServiceCollection()
	.AddScoped<IUserService, UserService>()
	.AddScoped<IUserRepository, UserRepository>();

var serviceProvider = serviceCollection.BuildServiceProvider();
```

Com o objeto serviceProvider em mãos agora é só executar o código abaixo para chamar a nossa lista de usuários e concluir este exercício.

```csharp
var userService = serviceProvider.GetRequiredService<IUserService>();

List<User> users = userService.GetUsers();
users.ForEach(user => 
  Console.WriteLine($"Nome: { user.Name } - Age: { user.Age }"));
```

Ao executar o console app pela linha de comando através do comando dotnet run (ou através da sua IDE de preferência) os nomes e idades dos dois usuários do exemplo são apresentados no terminal, como vemos no quadro abaixo.

```csharp
C:\cs-designpatterns\DependencyInjection.Console>dotnet run
Nome: Fulano - Age: 30
Nome: Beltrano - Age: 25
```

## Conclusão e insights

Note que a obtenção de uma instância de UserService se dá a partir do IUserService. Dentro da própria Program.cs talvez não seja óbvio ver o benefício desta configuração, mas se olharmos o código da UserService percebemos que dentro do construtor desta classe há uma referencia para a interface IUserRepository e não diretamente a classe UserRepository.

Desta forma a classe UserService passa a enxergar a interface IUserRepository como uma caixa preta, cuja implementação pode ser modificada sem prejuizo ao funcionamento de UserService. O contrário também deve ser verdadeiro, podemos mudar as regras de negócio codificadas na classe UserService e isso deve ser compatível com qualquer implementação de UserRepository, devendo-se apenas respeitar o comportamento descrito pelas interfaces.

