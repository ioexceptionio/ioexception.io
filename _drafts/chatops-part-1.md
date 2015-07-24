---
layout: post
title: "ChatOps - Comunicação, Deploys rápidos, times satisfeitos."
author: fernando
comments: true
tags: [ Hubot, deploy, heaven, github ]
css_classes: [ permalink ]
---

## ChatOps - o que é?
Vamos começar por aqui. 

ChatOps é um termo muito creditado ao pessoal do Github. Se formos resumir, podemos dizer que é "_conversation-driven development_". Usando um _bot_ com _plugins_ e _scripts_, os times podem automatizar tarefas e colaborar, melhorando o trabalho de todos, jogando fora os procedimentos repetitivos e economizando tempo.

## Deployments práticos 
Como pode ver, podemos usar _bots_ para automatizar muitas coisas, como realizar _backups_, notificar colaboradores sobre algum evento (alguma modificação em _issues_/_tasks_, por exemplo), preparação de novos ambientes, _code deployments_, etc. Hoje focarei mais no último: _**code deployments**_.

## _Code deployments_ - Frequência e duração 
A frequência do _deploy_ do(s) produto(s) que você trabalha diariamente, provavelmente, tem relação com o tempo gasto para cada _deployment_ que é feito nos ambientes de qualidade/_staging_ e produção. 

Esse assunto é velho, mas será que seu time está tirando o maior proveito possível deste processo? Não temos como saber, mas se os seus _deploys_ não estão muito frequentes, talvez seja um sinal que é possível melhorar.

## Contextualização
Esse negócio de ChatOps, surgiu na minha vida profissional em um time que o adotou quando estava começando a trabalhar com alguns elementos que até "pouco" tempo atrás eram novidade: _feature branches_, _microservices_, etc. Meu desafio aqui, é relatar a experiência e os benefícios identificados em uma equipe em que tais práticas foram adotadas, já o seu é perceber se há ou não benefícios para seu time adotar algo aqui dito.

Claro que tudo é questão de perspectiva. Alguns times adotaram, outros não quiseram ou não tiveram necessidade. O fato é que hoje _feature branches_ e _microservices_ são assuntos massificados e muitas vezes precisamos tornar certos procedimentos mais eficientes, mas a história aqui é outra. Vamos em frente. 

Até aqui... Lhe contextualizei o suficiente?

O motivo de adotarmos tais praticas posso contar no futuro. A escolha é do leitor.

Então voltemos a focar apenas no nosso processo de _code deployment_.

## A vida do time antes do ChatOps
O processo de _deploy_ durava até 30 minutos. O time utilizava apenas o Bamboo, da Atlassian, para integração contínua. 

O elemento mais adequado para ser classificado como "legado" do nosso sistema é o servidor web, escrito em Java, que publica uma API REST para ser consumida pelo front-end e pelas aplicações móveis (iOS e Android).

Este servidor Java, era o qual mais realizávamos _deployments_ diariamente. Tanto pelo time de qualidade, quanto pelo time de operações, em produção. 

O processo de _deploy_ era muito lento, principalmente para o time de qualidade.

Começava pelo _build_, que era composto por: 
- Rodar testes unitários;
- Gerar o artefato no servidor de _CI_ (Bamboo);
- Rodar análise de qualidade do Sonarqube;
- Baixar o artefato gerado (.war);
- Enviar para o servidor e rodar um script para  realizar a atualização.

Agora, imagine esses passos para cada _feature branch_ a ser testado pela qualidade? Pois é! Muito tempo perdido!

Além disso, o time de qualidade possuía alguns ambientes para testar, e eles precisavam saber de forma rápida qual versão/_feature branch_/_revision_ estava rodando em cada um destes ambientes. Como resolver?

## Bots são divertidos
Nesse ponto, decidimos testar a ideia e encontramos o _Hubot_ e outras ferramentas divertidas. Decidimos também descomplicar tudo que fosse possível, mantendo apenas o que fosse necessário para não retroceder na qualidade do software.

É nesse ponto que começa a diversão!

Nosso objetivo era realizar automaticamente todos aqueles procedimentos que precisavam ser feitos de forma semi-manual, focando no _code deployment_.

O que quero dizer com focar no _code deployment_?

O Bamboo realiza varias tasks para garantir a qualidade do software. Algumas dessas tasks são meio lentas, e se precisássemos esperar por elas diversas vezes por dia, resultava em muito tempo perdido.

Com isso, **pelo menos por enquanto**, precisamos manter o Bamboo para continuar executando os procedimentos que asseguram o rastreio da qualidade do software, mas podemos utilizar outras ferramentas em paralelo para realizar o deploy. O Bamboo continua gerando artefatos, rodando Sonarqube, etc e, em paralelo, fazemos nossos deploys. :)

Precisamos de um **IM** qualquer para utilizar com o Hubot (IRC, Flowdock, Slack, HipChat, etc). Escolhemos o **Slack**.

Até aqui:
* Matemos o Bamboo para rodar todas as tarefas que asseguram a qualidade do software;
* Escolhemos o Hubot para ser nosso bot.
* Escolhemos o Slack como IM.

Agora precisamos entender um pouco a API de Deployments do Github, pois tudo gira em torno dela.

## Descobrindo a API de Deployments do Github

O Github lançou há um tempo a API de Deployments. 
Resumindo, ela serve meio como um CRUD de deployments.

Você pode criar um Deployment e relacionar ele a uma _revision_/_tag_/_branch_ do seu projeto/repositório.

O Github vai guardar isso e você pode registrar **_webhooks_** para que outra ferramenta faça alguma coisa após a criação deste _deployment_, como por exemplo realizar o deployment propriamente dito.

Esse deployment registrado no Github pode ter seu status atualizado (**_pending_**/**_started_**/**_completed_**).

Abaixo tem um diagrama de sequencia retirado de uma [página do Github][github-deployments-api-page] que fala sobre essa API de Deployments. Dê uma olhada:

<pre>
+---------+             +--------+            +-----------+        +-------------+
| Tooling |             | GitHub |            | 3rd Party |        | Your Server |
+---------+             +--------+            +-----------+        +-------------+
     |                      |                       |                     |
     |  Create Deployment   |                       |                     |
     |--------------------->|                       |                     |
     |                      |                       |                     |
     |  Deployment Created  |                       |                     |
     |<---------------------|                       |                     |
     |                      |                       |                     |
     |                      |   Deployment Event    |                     |
     |                      |---------------------->|                     |
     |                      |                       |     SSH+Deploys     |
     |                      |                       |-------------------->|
     |                      |                       |                     |
     |                      |   Deployment Status   |                     |
     |                      |<----------------------|                     |
     |                      |                       |                     |
     |                      |                       |   Deploy Completed  |
     |                      |                       |<--------------------|
     |                      |                       |                     |
     |                      |   Deployment Status   |                     |
     |                      |<----------------------|                     |
     |                      |                       |                     |
</pre>

## Colando as partes

Tá. Agora vamos juntar as partes.

Tem um script prontinho para registrar os _deployments_ lá na API do Github. O [hubot-deploy][hubot-deploy], do [@atmos][atmos]. Já está prontinho para usar. É muito bom. 

Para escutar os _webhooks_ e consumir a API de Deployments do Github, utilizamos o projeto que também é de autoria do [@atmos][atmos], o [heaven][heaven]. 

O heaven possui integração com diversas ferramentas que realizam deploy. Sozinho ele basicamente consome a API do Github e identifica qual repositório precisa clonar e qual _revision_ precisa fazer checkout. O resto do trabalho ele deixa com alguma ferramenta a sua escolha. Essa ferramenta pode ser uma das disponíveis ou facilmente você pode implementar uma simples classe Ruby e fazer do seu jeito.

Ele possui integração com **fabric**, **capistrano**, **AWS OPSWorks** e algumas outras já prontas. 

Nós escolhemos utilizar o [fabric][fabric], que é um framework simples feito em python para realizar deployments em múltiplas máquinas.

Lá é que rodamos o `mvn clean package` para gerar o `.war` (Java, não é?) e enviamos para o servidor, que está na nuvem (na [AWS][aws], no caso), realizamos as configurações e tudo que for necessário para deixar a aplicação atualizada e funcionando.

[github-deployments-api-page]: https://developer.github.com/v3/repos/deployments/
[hubot-deploy]: https://github.com/atmos/hubot-deploy
[heaven]: https://github.com/atmos/heaven
[atmos]: https://github.com/atmos
[fabric]: http://www.fabfile.org
[aws]: http://www.fabfile.org





















