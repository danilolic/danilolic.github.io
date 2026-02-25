---
title: Microserviços vs Monolito sob a ótica das falácias
description: >-
  As 8 Falácias de sistemas distribuídos
author: danilo
date: 2025-12-18 16:20:00 -0300
categories: [Blogging, Not ready]
tags: [Ruby on Rails, Distributed Systems]
pin: true
media_subpath: '/posts/20260225'
---



![https://deniseyu.io/](https://deniseyu.io/art/sketchnotes/topic-based/8-fallacies.png)
imagem: [deniseyu](https://deniseyu.io/)

As 8 Falácias dos Sistemas Distribuídos foram formalizadas por engenheiros da Sun Microsystems em 1994. Elas representam suposições erradas que desenvolvedores frequentemente fazem ao projetar sistemas distribuídos. Dos anos 90 para cá tivemos uma popularização da arquitetura de microserviços enquanto que o monolito se tornou quase que um sinônimo de algo nagativo. Eu gostaria de abordar os tradeoffs dessa escolha ao longo desse artigo.

Antes de comparar as duas arquiteturas vamos entender melhor sobre cada falácia:

## Falácia 1 - A rede é confiável

**Falácia**: Assumir que a comunicação entre serviços nunca falha.

### Realidade
- Pacotes podem ser perdidos
- Conexões podem cair
- DNS pode falhar
- Load balancers podem reiniciar

Mesmo dentro da mesma VPC na Amazon Web Services isso acontece.

### Consequência prática

Se você não tratar falhas:

- Requests quebram
- Dados ficam inconsistentes
- Processos ficam parcialmente executados

### Boas práticas

- Retry com backoff exponencial: estratégia para fazer tentativas em intervalos espaçados de forma exponencial. Talvez você já tenha ouvido falar essa estratégia quando leu mais sobre o sidekiq. O Sideki possui mecanismo de retentativa e funciona com exponential backoff. Você pode ver a tabela de tentativas espalhadas no tempo aqui [Sidekiq exponential backoff](https://gist.github.com/marcotc/39b0d5e8100f0f4cd4d38eff9f09dcd5)
- Timeout bem definido: não deixe um requisição HTTP durar "para sempre". A `gem faraday`define uma espera de padrão 60 segundos, caso o servidor não responda nesse tempo, o faraday lançará uma exceção `Faraday::TimeoutError`.
- Circuit Breaker
- Idempotência: estratégia usada para evitar que eventos/requisições repetidas geram inconcistências. Por exemplo: um microserviço de pagamento recebe um evento duplicado para creditar saldo na conta do cliente, por exemplo uma entrada via Pix. Usando uma chave identificadora do evento, ou uma chave identificadora do pagamento Pix, podemos analisar se aquele pagamento já existe na base, caso exista você pode simplesmente ignorar o evento duplicado.

## Falácia 2 - Latência é zero

**Falácia**: Assumir que chamadas remotas são tão rápidas quanto chamadas locais.

### Realidade
- Uma chamada HTTP pode levar 5ms… ou 500ms
- Cross-region pode levar dezenas ou centenas de ms
- A latência pode variar (jitter)

### Impacto
- N chamadas em cascata multiplicam latência
- Sistemas síncronos demais ficam lentos

Exemplo:
- Se um endpoint chama 5 microserviços em sequência, a latência total pode explodir.

### Boas práticas

- Reduzir chamadas remotas: ao invés de fazer várias chamadas pegando pequenos pedaços, faça uma chamada só pegando tudo que precisar.
- Paralelizar quando possível
- Cache: Utilize cache em dados que mudam com pouco frequência e possuem grande volume de acessos. Caso precise diminuir custos com caching sugiro fortemente a leitura do [Rails Solid Cache](https://github.com/rails/solid_cache)
- Processamento assíncrono: Use o Sidekiq


## Falácia 3 - A largura de banda é infinita

Falácia: Assumir que podemos transferir qualquer volume de dados sem impacto.

### Realidade
- Transferência grande consome banda
- Pode gerar gargalo
- Pode aumentar custo (egress na AWS)

Exemplo:
- Serializar objetos gigantes em JSON
- Enviar payloads enormes em cada request

### Boas práticas
- Compressão
- Paginação
- Streaming
- Evitar over-fetching


## Falácia 4 - A rede é homogênea

Falácia: Supor que todos os nós usam o mesmo hardware, sistema operacional, versão de runtime, etc.

### Realidade
- Containers diferentes
- Versões diferentes de bibliotecas
- Sistemas legados convivendo com novos
- Ambientes híbridos (on-prem + cloud)

### Impacto
- Problemas de compatibilidade
- Diferenças de encoding
- Incompatibilidade de protocolos

Exemplo:

um caso clássico onde um projeto usando Ruby versão 3xx exige uma gem maior que 2.0.0 e o outro projeto com Ruby 2xx que exige a gem menor que 2.0.0 obrigatoriamente. Porém a atualização da gem também remove vários métodos depreciados, sendo impossível manter compatibilidade com ambos os projetos. Agora imagine que essa gem é na verdade uma dependência da gem principal que está no seu projeto Rails. Seus projetos teriam que criar duas versões da gem principal, uma que mantém a dependência menor que 2.0.0 e outra que mantém maior que 2.0.0. Você pode argumentar que seria só atualizar o Ruby para manter todos os projetos na versão 3xx, mas a realidade é que isso pode causar mais problemas de compatibilidade.

### Boas práticas
- Versionamento de API
- Contratos bem definidos (OpenAPI, Protobuf)
- Deploy gradual (canary)


## Falácia 5 - A rede é segura

Falácia: Achar que só porque está “dentro da rede interna” está protegido.

### Realidade
- Ataques internos existem
- Vazamentos de credenciais acontecem
- Movimento lateral é comum

Inclusive em ambientes corporativos grandes.

### Boas práticas
- Zero Trust
- TLS interno
- Autenticação entre serviços
- Rotação de segredos
