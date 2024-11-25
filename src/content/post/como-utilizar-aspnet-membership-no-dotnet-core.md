---
publishDate: 2024-11-25T00:00:00Z
author: Guilherme Melo
title: Como Utilizar o Asp.net Identity Core no .NET 8 com as tabelas do ASP.NET Membership do .NET 4.7 legado 
excerpt: Conheça como fizemos essa integração entre duas tecnologias incompatíveis de autenticação de usuários combinando um cadastro legado com novas tecnologias
image: /postimages/membership-identity.png
category: artigos
tags:
  - dotnet
  - .net
  - csharp
  - asp.net
  - membership
  - sistemas legados
metadata:
  canonical: https://www.gldmelo.com.br/como-utilizar-aspnet-membership-no-dotnet-core
---

O ASP.NET Membership é um framework para gerenciamento de usuários lançado para o ASP.NET 2.0 em meados de 2005. Construído com base no Forms Authentication rapidamente se tornou o padrão para autenticação de aplicações web que utilizam o .Net Framework naquela década devido a sua simplicidade de configuração e uso.

Sua adoção se deu em uma época em que as as aplicações desktop começavam a abrir caminho para uma migração gradual para sistemas web mais complexos como conhecemos hoje. Dentro da stack Microsoft nessa mesma época se utilizava o ASP.NET Web Forms, tecnologia que serviu como ponte nessa mudança de paradigma desktop para web, mas isso podemos conversar em outro artigo =)

## Mudança de requisitos e necessidade de seguir em frente

A primeira vez que tivemos a intenção de migrar um projeto legado do .NET Framework o .NET ainda se chamava .NET Core e estava na versão 5.0. Atualmente o .NET Core é chamado apenas de .NET e esta é a nomenclatura que irei utilizar no restante do artigo.
O primeiro obstáculo que esbarramos foi a autenticação. Não haveria como utilizarmos o ASP.NET Identity Core nos novos projetos pois toda base de usuários foi criada com o Membership há mais de 10 anos.

Uma das limitações do Membership é que ele foi concebido em uma época que a principal forma de autenticação de usuários se dava através de um formulário com usuário e senha. Outras formas de autenticação foram surgindo, bem como a necessidade de autenticar-se sem uma senha como observamos em sites que permitem logar com uma conta Google ou Facebook, ou até mesmo utilizar a autenticação de dois fatores.

Além das limitações soma-se ao fato que o Membership foi construido de forma acoplada ao .NET Framework, não sendo possível utilizá-lo em versões do .NET baseadas no Core.

Embora seja possível integrar logins como Google em aplicações legadas com Membership através de pacotes Nuget específicos para esta finalidade, o próprio .NET Framework hoje já é um legado e não receberá novas atualizações de funcionalidade por parte da Microsoft. A tendência é que essas alternativas fiquem paradas no tempo, sem suporte ou apenas parem de funcionar quando algum protocolo desses for atualizado por qualquer motivo, sem aviso prévio.

### Possibilidades inicialmente avaliadas para solução do problema

A primeira opção provavelmente é a mais utilizada. Não é feito nenhum tipo de migração e a empresa segue utilizando o sistema legado. Novos sistemas são integrados através de interfaces na frente desse sistema de modo a criar uma camada de gambiarra para que tudo funcione e apenas alguns saibam como aquilo de fato funciona.

Outra opção é converter as tabelas do Membership em tabelas equivalentes do Identity. 
Esse tipo de script de migração se encontra pronto na internet, basta procurar.
Ao meu ver até é uma boa solução quando temos um projeto que pode ser migrado ou reescrito sem maiores complicações no .NET.
Desta forma temos uma migração completa, sem intermediários.

Agora que vimos dois casos extremos apresento uma terceira opção que é um meio termo, o pacote nuget MembershipIdentity que visa utilizar nativamente as classes do ASP.NET Identity dentro de uma aplicação .NET 8, 
porém com uma implementação personalizada do acesso a dados que possibilita utilizar as tabelas do Membership sem duplicação de dados.

Esta opção acabou sendo adotada, pois a maior parte dos sistemas deste cliente são baseados em .NET Framework 
que é incompatível com o .NET, e naquele momento era inviável iniciar um processo de migração completa desses sistemas 
devido a grande quantidade de ajustes necessários para esta adequação.

### Como funciona o Asp.net Membership no .net Framework (versões 2.0 a 4.8)

Para o nosso estudo de integração é necessário conhecer e ter acesso às tabelas do Membership:  
* aspnet_users - tabela geral de usuários (anônimos e cadastrados) 
* aspnet_membership - tabela com informações adicionais para usuários cadastrados
* aspnet_roles - tabela que armazena os nomes das roles de usuário, como Admin, Operador, entre outros
* aspnet_usersinroles - tabela associativa para role e usuário

As configurações e uso do Membership no .NET Framework não são objeto deste artigo, portanto não irei detalhar aqui todo o processo. Essas informações já estão bem documentado na internet e também no Microsoft Learn cujo link deixo aqui para estudos mais aprofundados:
https://learn.microsoft.com/en-us/aspnet/web-forms/overview/moving-to-aspnet-20/membership

### A construção do nuget MembershipIdentity

Podes conferir o código-fonte completo deste pacote no meu github:
https://github.com/gldmelo/membership-identity

Uma das características interessantes do Identity é a flexibilidade na configuração da entidade usuário.
No nosso exemplo criamos uma classe MembershipUser contendo as colunas da tabela aspnet_membership.
A classe MembershipRole representa um role de usuário e a classe MembershipPasswordHasher implementa a 
validação de senha usando um hash com salt e criptografia SHA256. 

Na versão 1.2 foi acrescentado o suporte para senhas não criptografadas com um hash e salt pois aplicações 
"desta época" muitas vezes não tinham essa preocupação com proteção das senhas, e esta é a configuração padrão do 
Membership também.

Ao todo dividi a solução em dois projetos, como tem sido praxe em diversos pacotes .NET.
Um pacote principal *MembershipIdentityProvider* e outro *MembershipIdentityProvider.SqlServer*. 

Com o pacote principal, cada banco de dados onde haja necessidade de se implementar tal artifício de integração 
pode implementar as interfaces e as consultas do banco de dados onde estiver suas tabelas do Membership.

Como o próprio nome diz, o segundo pacote já traz consigo uma implementação destas consultas para o Sql Server.
A implementação principal se dá na classe *SqlServerMembershipUserStore* que implementa as seguintes interfaces 
do Identity *IUserStore<TUser>*, *IUserRoleStore<TUser>*, *IUserClaimStore<TUser>*, 
e *IUserPasswordStore<TUser>* com essas consultas para o banco de dados.

### Então, como funciona a autenticação do usuário com o ASP.NET Identity Core?

Considerando um esquema de classes padrão, como no template de novo projeto web do Visual Studio, temos uma classe AccountController.
Nesta classe haverá um método Login como este abaixo:

```csharp
[HttpPost]
[AllowAnonymous]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Login(LoginViewModel model, string returnUrl = null)
{
    ViewData["ReturnUrl"] = returnUrl;
    if (ModelState.IsValid)
    {
        var result = await _signInManager.PasswordSignInAsync(model.UserLogin, model.Password, model.RememberMe, lockoutOnFailure: false);
        if (result.Succeeded)
        {
            //_logger.LogInformation("User logged in.");
            return RedirectToLocal(returnUrl);
        }
        //... restante do código foi omitido        
    }
```

A linha de interesse é a chamada do método *_signInManager.PasswordSignInAsync()*. Como configuramos em nosso 
projeto o pacote *MembershipIdentityProvider.SqlServer* esta chamada executará o método *FindByNameAsync*
dentro do nosso pacote para popular um objeto do tipo MembershipUser com os dados da tabela do Membership, 
e depois o método *PasswordSignInAsync* que utiliza o *PasswordHasher* para validação da senha digitada 
e criptografada com o mesmo hash e salt da senha armazenada no banco de dados.

Note que essas chamadas todas são em cima de métodos e classes do próprio Identity Core, e a nossa alteração é apenas 
uma personalização de alguns métodos que alteram qual consulta deve ser executada contra o banco de dados e qual é o 
objeto contendo esses dados que deve ser retornado.

Note que é o *SignInManager* original do Identity Core que é passado para as Controllers através de Injeção 
de Dependência, portanto a solução é totalmente desacoplada e independente, podendo ser substituída.

### Conclusão

Neste artigo vimos como foi possível criar um meio termo para autenticação no .NET 8 utilizando o ASP.NET Identity Core 
utilizando as tabelas do ASP.NET Membership.

O código completo em detalhes pode ser encontrado no meu [Github](https://github.com/gldmelo/membership-identity), 
ou utilizado diretamente em seus projetos. Basta referenciar os pacotes abaixo para a implementação em Sql Server.

*install-package MembershipIdentityProvider*

*install-package MembershipIdentityProvider.SqlServer*

Atualmente utilizo esta solução em projetos administrativos que precisam autenticar usuários que foram criados em
sistemas legados utilizando o Membership. Então se algum dia esse banco de dados for atualizado para utilizar as tabelas
nativas do Identity Core, basta remover essas referências e utilizar normalmente o Identity sem maiores alterações nos 
projetos .NET.
