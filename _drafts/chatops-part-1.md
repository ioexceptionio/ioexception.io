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

ChatOps é um termo muito creditado ao pessoal do Github. Se formos resumir, podemos dizer que é "_conversation-driven development_". Usando um _bot_ com _plugins_ e _scripts_, os times podem automatizar tarefas e colaborar, jogando fora os procedimentos repetitivos e economizando tempo.

Para aplicar isso, os times usam bots para automatizar os procedimentos manuais e repetitivos. Alguns dos mais conhecidos são o [Hubot][hubot] e o [Lita][lita].

![hubot-image]({{ site.url }}/assets/images/2015/07/hubot.png)


## <a name="deployments-praticos"></a>Deployments práticos
Podemos usar _bots_ para automatizar muitas coisas, como realizar _backups_, notificar colaboradores sobre algum evento (alguma modificação em _issues_/_tasks_, por exemplo), preparação de novos ambientes, _code deployments_, etc. Hoje focarei mais no último: _**code deployments**_.

Sabe aquele procedimento chato que você teve que fazer com Shell Script? Copiava uma coisa pra uma máquina, rodava um script alí, mudava umas configurações e rodava a aplicação? É ele mesmo que vamos focar aqui. Vamos acabar com isso. Vamos mandar os bots fazerem tudo pra nós. Quando terminarem que nos avise.

<a name="imagem-exemplo-deployment"></a>
![bots-escravos]({{ site.url }}/assets/images/2015/07/chatops-1-jarvis-deploy2.png)

Gostou da ideia?

## _Code deployments_ - Frequência e duração 
A frequência do _deploy_ do(s) produto(s) que você trabalha diariamente, provavelmente, tem relação com o tempo gasto para cada _deployment_ que é feito nos ambientes de qualidade/_staging_ e produção. 

Esse assunto é velho, mas será que seu time está tirando o maior proveito possível deste processo? Não temos como saber, mas se os seus _deploys_ não estão muito frequentes, talvez seja um sinal que é possível melhorar.

## Contextualização
Esse negócio de ChatOps surgiu na minha vida profissional em um time que o adotou quando estava começando a trabalhar com alguns elementos que até "pouco" tempo atrás eram novidade: _feature branches_, _microservices_, etc. Meu desafio aqui, é relatar a experiência e os benefícios identificados em uma equipe em que tais práticas foram adotadas, já o seu é perceber se há ou não benefícios para seu time adotar algo aqui dito.

Claro que tudo é questão de perspectiva. Alguns times adotaram, outros não quiseram ou não tiveram necessidade. O fato é que hoje _feature branches_ e _microservices_ são assuntos massificados e muitas vezes precisamos tornar certos procedimentos mais eficientes, mas a história aqui é outra. Vamos em frente. 

Até aqui... Lhe contextualizei o suficiente?

O motivo de adotarmos tais praticas posso contar no futuro. A escolha é do leitor.

Então voltemos a focar apenas no nosso processo de _code deployment_.

## A vida do time antes do ChatOps
O processo de _deploy_ durava até 30 minutos. O time utilizava apenas o Bamboo, da Atlassian, para integração contínua. 

O elemento mais adequado para ser classificado como "legado" do nosso sistema é o servidor web, escrito em Java, que publica uma API REST para ser consumida pelo front-end e pelas aplicações móveis (iOS e Android).
Este servidor Java, era o qual mais realizávamos _deployments_ diariamente. Tanto pelo time de qualidade, quanto pelo time de operações, em produção. 
O processo de _deploy_ era muito lento, principalmente para o time de qualidade.

O time de _QA_ realizava procedimentos manuais como baixar o artefato, fazer acesso remoto (SSH), etc. mas tudo começava pelo _build_, que era composto por: 
- Rodar testes unitários;
- Gerar o artefato no servidor de _CI_ (Bamboo);
- Rodar análise de qualidade do Sonarqube;
- Baixar o artefato gerado (.war);
- Enviar para o servidor e rodar um script para  realizar a atualização.

Agora, imagine esses passos para cada _feature branch_ a ser testado pela qualidade? Pois é! Muito tempo perdido!
Além disso, o time de qualidade possuía alguns ambientes para testar, e eles precisavam saber de forma rápida qual versão/_feature branch_/_revision_ estava rodando em cada um destes ambientes. Como resolver?

Com ChatOps!

## Bots são divertidos
Nesse ponto, decidimos testar a ideia e encontramos o _**Hubot**_ e outras ferramentas divertidas. Decidimos também descomplicar tudo que fosse possível, mantendo apenas o que fosse necessário para não retroceder na qualidade do software.
O Hubot nos permite usar [_**CoffeeScript**_][coffeescript] para criar scripts e construir comandos que fazem o trabalho daqueles passos manuais, assim podemos descarta-los. 
Todo processo que antes era feito de forma manual ou que é dispendioso, podemos deixar por conta do Hubot.
Onde existem vários passos, podemos transformar em apenas um comando, como `hubot do stuff` e ele fará por você, como pode ver em uma das [imagens mostradas anteriormente](#imagem-exemplo-deployment).

É nesse ponto que começa a diversão!

Nosso objetivo era realizar automaticamente todos aqueles procedimentos que precisavam ser feitos de forma semi-manual, focando no _code deployment_.
O que quero dizer com focar no _code deployment_?
O Bamboo realiza varias tasks para garantir a qualidade do software. Algumas dessas tasks são meio lentas, e se precisássemos esperar por elas diversas vezes por dia, resultava em muito tempo perdido.

Com isso, **pelo menos por enquanto (quem sabe?)**, precisamos manter o Bamboo para continuar executando os procedimentos que asseguram o rastreio da qualidade do software, mas podemos utilizar outras ferramentas em paralelo para realizar o deploy. O Bamboo continua gerando artefatos, rodando Sonarqube, etc e, em paralelo, fazemos nossos deploys. :)

Precisamos de um aplicativo de **IM** (_Instant Messaging_) qualquer para utilizar com o Hubot (IRC, Flowdock, Slack, HipChat, etc). Escolhemos o **Slack**.

Até aqui:
* Matemos o Bamboo para rodar todas as tarefas que asseguram a qualidade do software;
* Escolhemos o Hubot para ser nosso bot.
* Escolhemos o Slack como IM.

Com ChatOps, o deployment pode ser iniciado por qualquer um e, ainda, de forma assíncrona. Desta forma, precisamos de um lugar para centralizar o controle de todos os deployments.

Por isso, precisamos entender um pouco a API de Deployments do Github, pois tudo gira em torno dela. É lá que vamos centralizar o controle de nossos deployments.

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

Tem um script do Hubot para registrar os _deployments_ lá na API do Github. O [hubot-deploy][hubot-deploy], do [@atmos][atmos]. Vamos usa-lo pra criar nosso deployments.

Porém, os deployments criados no Github precisam ser coordenados. É necessário também ter alguma ferramenta para receber os _webhooks_ de deployment do Github. No diagrama acima, essa ferramenta está descrita como `3rd Party`. 
Tem um projeto, também de autoria do [@atmos][atmos], para isso: o [heaven][heaven]. 

O Heaven possui integração com diversas ferramentas de deployment. Sozinho ele basicamente consome a API do Github e identifica qual repositório precisa clonar e qual _revision_ precisa fazer checkout. O resto do trabalho ele deixa com alguma ferramenta a *sua* escolha. Essa ferramenta pode ser uma das disponíveis ou facilmente você pode escrever uma simples classe Ruby e fazer do seu jeito.

Ele possui integração com **fabric**, **capistrano**, **AWS OPSWorks** e algumas outras já prontas. 

Nós escolhemos utilizar o [fabric][fabric], que é um framework simples feito em python para realizar deployments em múltiplas máquinas.

Lá é que rodamos o `mvn clean package` para gerar o `.war` (Java, não é?) e enviamos para o servidor, que está na nuvem (na [AWS][aws], no caso), realizamos as configurações e tudo que for necessário para deixar a aplicação atualizada e funcionando.

## Hmm... E aí, tem vantagem?

No fim, ficou assim:

```
+---------+             +--------+            +----------+         +-------------+
|  Hubot  |             | GitHub |            |  Heaven  |         | Your Server |
+---------+             +--------+            +----------+         +-------------+
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
```

Um simples comando e todo o trabalho de deployment é feito pra nós:
![bots-escravos]({{ site.url }}/assets/images/2015/07/chatops-1-jarvis-deploy2.png)

Como pode ver, o tempo para realizar o deployment e ter a aplicação funcionando, é de 3 minutos hoje em dia. Eu mencionei anteriormente que esse tempo era por volta de ~15 minutos.

Além disso, todos do time podem ver o que está acontecendo e quais ambientes estão sendo utilizados. Isso economiza tempo. Ninguém precisa parar ninguém para perguntar. (Lembra que antes o time precisava criar um controle para saber qual versão/branch estava em cada ambiente?!)

O hubot-deploy também disponibiliza outros comandos interessantes como o que lista os deployments realizados em um ambiente específico: `hubot deploys PROJECT in ENV`. Esse comando ajuda a saber quais _feature branches_ estão aplicados em cada um dos seus ambientes ou qual versão está em produção, por exemplo.

![hubot-deploys-env]({{ site.url }}/assets/images/2015/07/hubot-deploys-env.png)

Isso tudo deixou o time mais rápido, os deployments mais rápidos e, consequentemente, melhorou todo o processo do time em geral, mitigando o tempo perdido com tudo o que foi mencionado neste post.

Posteriormente, caso exista interesse, posso criar um tutorial envolvendo hubot, hubot-deploy, heaven e fabric.

Mas e aí? Será que isso traria alguma vantagem para o seu time? Isso é só uma das poucas coisas que podemos fazer com o Hubot.
;)


[github-deployments-api-page]: https://developer.github.com/v3/repos/deployments/
[hubot-deploy]: https://github.com/atmos/hubot-deploy
[heaven]: https://github.com/atmos/heaven
[atmos]: https://github.com/atmos
[fabric]: http://www.fabfile.org
[aws]: http://www.fabfile.org
[hubot]: https://hubot.github.com
[lita]: https://www.lita.io
[coffeescript]: http://coffeescript.org



















