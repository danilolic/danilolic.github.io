---
title: Representação monetária em Ruby
description: >-
  O Problema da imprecisão
author: danilo
date: 2025-12-18 16:20:00 -0300
categories: [Blogging, Tutorial]
tags: [Ruby, Rails]
pin: true
math: true
media_subpath: '/posts/20251218'
---

Ao lidar com valores monetários em sistemas de software, a forma como esses dados são armazenados e manipulados é um ponto crítico para garantir precisão, consistência e segurança.

Existem diferentes abordagens para representar valores monetários, cada uma com vantagens e limitações. Os tipos básicos mais usados são String, Inteiro, Float e Decimal, então vamos falar brevemente sobre cada um desses tipos.

## Tipo String

O uso de strings, é comum em contextos de exibição e entrada de dados.

Por exemplo, sua API pode receber uma requisição HTTP com o body:

```json
{
  "amount": "199.99"
}
```
E a serialização da resposta HTTP pode conter um número representado como String:

```json
{
  "payment": {
    "id": "0c37fae9-58b7-43f2-acd2-8caa660d8b2a",
    "amount": "199.99"
  }
}
```

porém **não é adequado para cálculos**. A representação de números em texto pode ser problemática em linguagens fracamente tipadas igual o Javascript onde você consegue fazer algo como `String + Number` sem gerar nenhum aviso, mas gerando um erro lógico silencioso que pode causar problemas. Em linguagens fortemente tipadas como o Ruby, você vai tomar um erro em tempo de execução. Uma única exceção é que Ruby aceita a multiplicação `String * Number`, então é um ponto que merece atenção, mas acredito que com bons testes você não terá problemas, mas caso queira um pouco mais de segurança vale a pena usar a gem `Dry::Validation` para assegurar tipos. Em linguagens estaticamente tipadas esse erro seria pego em tempo de compilação, portanto seria possível identificar e corrigir o problema em uma etapa antes do código ir para produção, essa é grande vantagem dessas linguagens.

## Tipo Float

O tipo Float é impreciso porque utiliza representação binária de ponto flutuante, o que impede que muitos números decimais comuns (como $ 0.1 $ ou $ 0.01 $) sejam representados exatamente. Esses valores são armazenados como aproximações, e pequenas diferenças se acumulam a cada operação matemática.

Então pense em operações onde 0.1 é por exemplo 10% ou $ \frac{10}{100} $ ou 0.01 é 1% ou $ \frac{1}{100} $. Esses valores são muito comuns em contexto financeiro, e eu poderia citar outros vários valores como taxa IOF, taxa DI, etc. Que por serem porcentagem vão estar definidos geralmente entre 0 e 1.

Nesse cenários temos situações que podem levar a bugs muito sérios, principalmente quando fazemos comparações.
Então imagine que você queira fazer uma soma de todas as taxas de um determinado cliente para classificá-lo em uma categoria:

###### Float - exemplo 1
```ruby
anticipation_tax = 0.09 # 9%
account_tax = 0.01 # 1%
threshold = 0.10

# anticipation_tax + account_tax
# => 0.09999999999999999 # impreciso

if (anticipation_tax + account_tax) >= threshold
 "Category A"
else
  "Category B"
end

=> "Category B" # Categoria incorreta
```

Existem vários outros exemplos e alguns podem facilmente te enganar, então vamos explorar um caso que a princípio parece inofensivo: se eu digito $ 0.1 $ no IRB e dou um ENTER ele me retorna $ 0.1 $

###### Float - exemplo 2
```ruby
irb(main):084:0> 0.1
=> 0.1
irb(main):085:0> 0.1.class
=> Float
```

Parece normal, certo? Por que então tivemos um erro de imprecisão no exemplo __Float - exemplo 1__ ? Vamos explorar esse valor várias casas após a vírgula dessa forma:

###### Float - exemplo 3
```ruby
irb(main):086:0> printf("%.30f", 0.1)
0.100000000000000005551115123126
```

Agora podemos ver a imprecisão! Bom, você pode argumentar que existem poucos cenários onde eu precisaria de uma precisão de 30 casas decimais (e você estaria certo). Na prática, se você não se importa com tantas casas decimais assim, significa que você está dando atenção só para as primeiras casas decimais, afinal, não existe um valor monetário como $ 0.100000000000000005551115123126 $, seria somente R$ 0.10 ou 10 centavos. Portanto o valor que o usuário verá em tela foi truncado, mas para esse caso felizmente a imprecisão não alterou o resultado final.

Existem ainda casos onde a imprecisão é ainda mais evidente:

###### Float - exemplo 4
```ruby
irb(main):082:0> 10.99 + 99.99
=> 110.97999999999999

# ESPERADO: 110.98
```

E claro, testes também estão suscetíveis a esse comportamento:
###### Float - exemplo 5
```ruby
it 'causes a failed test because of float imprecision' do
  calculation = 10.99 + 99.99
  expect(calculation).to eq(110.98)
end

=>

Failure/Error: expect(calculation).to eq(110.98)

       expected: 110.98
            got: 110.97999999999999
```

### O problema da truncagem
Dado o exemplo do valor 0.1 (Float - exemplo 3), nós não chamamos o método `truncate` e ainda sim eu argumentei que o valor foi truncado, bom, de forma consciente ou não, estamos truncando os valores para duas casas decimais. Isso é um fato porque o usuário não vai ver em tela $ 0.100000000000000005551115123126 $, então ainda que não tenha sido de forma direta, houve uma truncagem, mas nesse caso sem introdução de imprecisão.

O grande problema é que a truncagem pode levar a um valor final completamente diferente do valor matematicamente correto. Vou dar um exemplo explorando o pior cenário possível deste caso, aquele que todo programador vai falar "ah! mas eu nunca faria assim" ou "ninguém cometeria um erro tão bobo assim". Bom, essa é uma discussão inútil, o ponto é que se existe a possibilidade de acontecer, vai acontecer.

Cenário (realista) \
Valor base: R$ 123,45 \
IOF diário: 0,0082% → 0.000082 \
Período: 30 dias \
Regra: valores monetários com 2 casas decimais

###### Float - exemplo 6
```ruby
# Configurações do cenário
valor_base = 123.45
taxa_iof = 0.000082 # 0,0082%
dias = 30

# --- 1. CENÁRIO COM FLOAT + TRUNCATE DIÁRIO ---
saldo_float = valor_base
iof_total_float = 0

dias.times do
  imposto_do_dia = saldo_float * taxa_iof
  iof_total_float += imposto_do_dia

  # ERRO: Truncar o saldo ou o imposto a cada passo
  # Isso descarta frações de centavos que deveriam existir matematicamente
  iof_total_float = iof_total_float.truncate(2)
end

# IOF Acumulado (Float + Truncate): R$ 0.3000
# IOF Acumulado REAL: R$ 0.3037
```

Efeito Cascata do Truncate: Ao usar `truncate(2)`, você está jogando fora os "micro-centavos". Em um cálculo de 30 dias, isso parece pouco. Mas imagine um banco com 1 milhão de contas e cálculos feitos diariamente por anos, a diferença de centavos se transforma em **milhares de reais de erro contábil**.

Claro, **existem vários números que podem ser perfeitamente representados pelo tipo Float**.
Entenda o ponto mais importante: _O problema dos floats não é que eles sempre erram, mas que você nunca sabe quando vão errar._

Q: Então se o float carrega imprecisão e não podemos truncar, o que devemos fazer? \
R: Não usar Float **para cálculos**. Você ainda pode usar Float para fazer representações de números decimais, mas evite fazer cálculos. Minha opinição sobre isso é que mesmo nesse caso ainda é possível evitar o uso de Float visto que valor monetário tem tipos melhores para representação. A string, como já comentado acima, funciona para representar valores decimais onde não existe necessidade de cálculo, mas um BigDecimal funciona perfeitamente nesse caso também.

## Tipo Inteiro
Qualquer valor monetário pode ser armazenado como inteiro:

```
R$ 0.01 = 1
R$ 0.10 = 10
R$ 1.00 = 100
R$ 100.99 = 10099
...
```

Para mim é a forma mais segura e flexível de armazenar valores monetários além de resolver o problema de imprecisão, existem duas grandes vantagens:

1. Performance: Cálculos matemáticos com inteiros (CPU) são muito mais rápidos do que com decimais (processados via software).
2. Espaço: Um `DECIMAL(15,5)` pode ocupar 18 bytes, enquanto o `BIGINT` ocupa sempre 8 bytes, economizando mais de 50% de espaço em disco e memória em grandes volumes de dados.

## Tipo Decimal

O Ruby fornece suporte a tipos decimais através da classe `BigDecimal`

```ruby
require 'bigdecimal'
```

De acordo com a própria documentação:

> O BigDecimal oferece suporte para números de ponto flutuante muito grandes ou muito precisos.

> A aritmética decimal também é útil para cálculos em geral, pois fornece as respostas corretas que as pessoas esperam – enquanto a aritmética binária de ponto flutuante normal frequentemente introduz erros sutis devido à conversão entre a base 10 e a base 2.

Vamos fazer uma comparação com operações Float:

###### BigDecimal - exemplo 1
```ruby
0.1 + 0.1 + 0.1
=> 0.30000000000000004 # impreciso

result = (BigDecimal('0.1') + BigDecimal('0.1') + BigDecimal('0.1'))
=> 0.3e0
result == 0.3
=> true
```

O Decimal não deu erro de precisão. Esse é o esperado, mas e se nós misturarmos BigDecimal com Float?

###### BigDecimal - exemplo 2
```ruby
0.1 + BigDecimal('0.1')
=> 0.2e0
0.1 + BigDecimal('0.1') == 0.2
=> true
```

Ou seja, para uma operação onde há pelo menos um valor em BigDecimal, o resultado da operação é um BigDecimal. Nesse aspecto o Ruby se comportou mais próximo de uma linguagem fracamente tipada, mas isso acontece porque ambas as classes herdam da classe Numeric

```ruby
1.0.class.ancestors
=> [Float, Numeric, Comparable, Object, PP::ObjectMixin, Kernel, BasicObject]
BigDecimal('1.0').class.ancestors
=> [BigDecimal, Numeric, Comparable, Object, PP::ObjectMixin, Kernel, BasicObject]
```

Ruby converte o Float para BigDecimal, mas isso acontece de forma implícita e com algumas regras importantes.

Detalhe da sintaxe:

```ruby
x + y
```

em Ruby é o mesmo que

```ruby
x.+(y)
```

Significa que a instância "x" tem um método (+) que recebe "y" como parâmetro, então o Ruby vai tentar algo assim:

```ruby
1.0.+(BigDecimal(0.1))
Float#+(BigDecimal)
```

Mas o Float não sabe somar diretamente com um BigDecimal. Quando isso acontece, o Ruby usa um mecanismo chamado `coerce`. Porém BigDecimal não pode ser forçado para FLoat, então é o Float que será forçado para BigDecimal.

Na prática:

```ruby
a, b = BigDecimal('0.1').coerce(0.1)
# => [BigDecimal("0.1"), BigDecimal("0.1")]

# depois
BigDecimal("0.1") + BigDecimal("0.1")
```

### Detalhe da implementação em C

Uma pergunta importante que pode surgir: se eu chamo `coerce` para um número Float, ele será convertido com segurança? Ou seja, se 0.1 é impreciso, BigDecimal(0.1) é impreciso?

Para responder essa pergunta eu tive que ir no código fonte do Ruby e a reposta é: toda vez que uma instância de BigDecimal é criada com um Float o ruby faz um `Float.to_s` e depois cria o BigDecimal.

Caso tenha interesse eu deixei o link exatamente para o método que faz essa conversão:
[https://github.com/ruby/bigdecimal/blob/master/ext/bigdecimal/bigdecimal.c#L2688](https://github.com/ruby/bigdecimal/blob/master/ext/bigdecimal/bigdecimal.c#L2688)

Na prática tanto faz se o parâmetro é String ou Float:

```ruby
BigDecimal('0.1')
```
é o mesmo que
```ruby
BigDecimal(0.1)
```


**O Ruby conseguiu tomar a decisão mais correta possível quando se mistura BigDecimal com Float**, caso contrário poderíamos ter resultados catastróficos.


### A Mágica do Rails
Até aqui tudo que eu mostrei é Ruby puro, então como funciona para o Ruby on Rails?
Os tipos em memória são definidos pelos tipos especificados na migração, então vamos pegar esse exemplo:

###### BigDecimal - exemplo 3
```ruby
create_table "payments",
    t.float "gross_value"
    t.decimal "amount", precision: 15, scale: 5
```

Gross value será representado como Float, então jamais use esse tipo em migrações para representar valor monetário.

Já o amount será instanciado em Ruby automaticamente como BigDecimal, então no geral você não terá **problemas** se fizer cálculos em memória (pelo menos não com tipagem e imprecisão), **a não ser que por algum motivo (inexplicável)** você decida transformar decimais em float para fazer cálculos.

Outro ponto interessante é que no Rails, graças ao ActiveSupport também temos o método `to_d` que converte String ou Float para BigDecimal de uma forma muito mais simples:

```ruby
0.1.to_d
=> 0.1e0
"0.1".to_d
=> 0.1e0
```

## Últimas observações

No geral é muito cômodo e muito comum usar gems no Ruby/Rails para não ter que construir certas coisas. O Ruby e Rails são muito mágicos e muitas vezes a adição de um nova gem no projeto não é só para ganhar tempo, mas porque ninguém do time sabia exatamente como fazer aquela funcionalidade do zero e já que existe uma gem, então vamos usar a gem. Bom, agora que todo mundo consegue entender pelo menos parte das decisões para tratamento monetário (eu não falei sobre suporte para diferentes moedas), deixo aqui a sugestão de uma gem:

[https://github.com/RubyMoney/money-rails](https://github.com/RubyMoney/money-rails)
