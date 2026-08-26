## Calculo de Preço

A maioria dos serviços de LLM's como da OpenAI ou Anthropic usam um modelo de cálculo de consumo baseado em quantidades de tokens, em geral, 4 modalidades:

- Tokens de Entrada ou *Input Tokens* ($IT$): quantidade de tokens que compõem a janela de contexto fornecida à inferência. Neste artigo, $IT$ é uma grandeza conceitual e não necessariamente corresponde ao campo denominado input tokens no sistema de cobrança de um provedor específico.
- Tokens de Saída ou *Output Tokens* ($OT$): Tokens que são efetivamente gerados pela inferência, seja em resposta ou pensamento interno. Ou seja, necessariamente tokens que precisam ser calculados pela inferência do modelo.
- Leituras em Cache ou *Cache Read* ($CR$): A quantidade de tokens recuperadas do mecanismo de cache.
- Escritas em Cache ou *Cache Write* ($CW$): A quantidade de tokens que é persistida no mecanismo de cache.

Tokens podem ainda participar de operações de cache, representadas independentemente por $CR$ e $CW$. Assim, neste modelo, cache e entrada são dimensões distintas: um token pode fazer parte do contexto sem necessariamente corresponder a uma operação de leitura ou escrita de cache na mesma requisição.

Podemos calcular o custo $P$ de uma sessão somando os cutos pelas requisições desses 4 recursos. Uma única iteração do usuário com a LLM gera inumeras requests que podem ser pensamento, letura de arquivos, requisições, chamadas de ferramentas etc. Deste modo, considerando que $IT_k$, $OT_k$, $CR_k$ e $CW_k$ são os tokens, respectivamente, lidos, gerados, recuperados e persistidos na k-ésima requisição, podemos calcular $P$ da seguinte forma, considerando R requisições feitas durante toda sessão:

$$ P = \rho_{it} \sum_{k=0}^R IT_k + \rho_{ot} \sum_{k=0}^R OT_k + \rho_{cr} \sum_{k=0}^R CR_k + \rho_{cw} \sum_{k=0}^R CW_k $$

Ou considerando em valores totais:

$$ P = \rho_{it} IT + \rho_{ot} OT + \rho_{cr} CR + \rho_{cw} CW $$

Onde, $\rho_{n}$ é o custo por token de $n$. Neste cenário existem ainda alguns tokens que já são carregados no ínicio da sessão que podem ou não estarem em cache. É comum que alguns recursos estáveis sejam sempre reaproveitados em todas as sessões abertas e por isso são muitas vezes colocados em cache. Denotaremos esse recursos como prefixo cacheado preexistente ($CP$) e prefixo não cacheada ($UP$). Desta forma, base cacheada soma futuramente como novas leituras de cache, enquanto base não cacheada será contabilizada rapidamente como conteúdo que será cacheado nas primeiras requisições. Esses custos, em geral, podem ou não estar inclusos como Tokens de Entrada. Isso traz um pouco de incerteza para equação que iremos tratar nas seções seguintes.

## Hipóteses de Uso e Carga de Trabalho

Apesar da simplicidade, a equaçõe pode não ser suficientemente útil para tomar decisões em contextos práticos. Contudo, podemos assumir várias hipóteses afim de simplificar as análises e chegar a conclusões práticas que valem para grande maioria das situações e serviços de LLM do mercado.

### Hipótese 1: Cache nunca inválida e toda entrada e saída são cacheados

A primeira hipótese ($H_1$) é bem simples e as simplificações que partem dela são válidas se o usuário nunca permite que o cache seja invalida e perdido. Usando *Claude Code* numa versão paga, por exemplo, conteúdos permanescem em Cache por até 1 hora e tem tempo restabelecido em cada interação com a LLM. Desta maneira, todo conteúdo que compõe a sessão e que possa ser reutilizado em requisições futuras é eventualmente incorporado ao cache.

Além disso, em muitos sistemas, o conteúdo só é cacheado quando ele precisa ser reutilizado pela primeira vez. Isso implica que existe tokens de saída da última requisição ($UR$) que nunca irão para o cache. Outras lógicas podem ser aplicadas, por isso chamamos essa variáveis de resíduo não cacheado.

Tomando essa hipótese temos:

$$ H_1 \implies P = (\rho_{it} + \rho_{cw}) \sum_{k=0}^R IT_k + (\rho_{ot} + \rho_{cw}) \sum_{k=0}^R OT_k + \rho_{cr} \sum_{k=0}^R CR_k - (\rho_{cw}) (UR + \sum_{k=0}^R CB_k) $$

Ou seja, o preço da escrita em cache é embutida nos Tokens de Entrada. Em alguns serviços isso já acontece naturalmente. E surgem conteúdos que agora são calculados duas vezes, dado isso, existe um termo negativo que desconta conteúdo que nunca vai para cache ou que já estava em cache. Note que depender do sistema, o resíduo não cacheado ou a base cacheada podem ser efetivamente zero.

### Hipótese 2: A sessão é longa o suficiente

A segunda hipótese ($H_2$) implica que a sessão é longa o suficiente para que o residuo e o conteúdo base sejam desprezíveis. Neste caso podemos aproximar o preço da sessão considerando que $CP, UP, UR \approx 0$. Note que para uma sessão longa, isso não afeta $IT$ que é grande o suficiente para podermos ignorar os tokens de base que o compõe. Isso também simplifica cálculos onde o conteúdo base não é considerado como Token de Input.

$$ H_1 \land H_2 \implies P \approx (\rho_{it} + \rho_{cw}) \sum_{k=0}^R IT_k + (\rho_{ot} + \rho_{cw}) \sum_{k=0}^R OT_k + \rho_{cr} \sum_{k=0}^R CR_k $$

### Hipótese 3: Entrada e saída compõe o contexto

A terceir hipótese aponta um cenário comum onde tudo que está na janela de contexto $C$ é praticamente constuida da entrada e saída do modelo, ou seja, assumir que $C \approx \sum_{k=0}^R IT_k + \sum_{k=0}^R OT_k$. Essa hipótese é impossível se existe compactação da janela de contexto, comum em alguns fluxos de trabalho.

Em muitos contextos de uso podemos aproximar, com um erro controlável que $(\rho_{it} + \rho_{cw}) \sum_{k=0}^R IT + (\rho_{ot} + \rho_{cw}) \sum_{k=0}^R OT_k \lessapprox \rho_m(IT + OT) \approx \rho_mC$. Essa aproximação é util para definir um limite superior, considerando que geralmente o custo do token de saída e maior que o token de entrada. De forma geral tomamos $\rho_m= max((\rho_{it} + \rho_{cw}), (\rho_{ot} + \rho_{cw}))$. Unindo a terceira hipótese temos:

$$H_1 \land H_2 \land H_3 \implies P \lessapprox \rho_mC + \rho_{cr} \sum_{k=0}^R CR_k $$

### Hipótese 4: Pior caso de requisições

Em geral entre os serviços de LLM, a leitura de cache ocorre por todo conteúdo anterior a cada requisição. Isso significa que podemos ter várias leituras dos mesmos tokens em uma unica iteração do usuário, desde que essa tenha várias requisições a compondo. Podemos propor uma aproximção eficiente para leituras de cache em termos dos tamanhos da adição de conteúdo entre as requisições. Considerando $r_n$ o consumo de tokens da n-ésima requisição, e considerando a última requisição desprezível graças a $H_2$, podemos escrever o preço como:

$$H_1 \land H_2 \land H_3 \implies P \lessapprox \rho_mC + \rho_{cr} \sum_{k=0}^R \sum_{j=0}^k r_j $$

Outra aproximação de limite superior útil é analisar o pior caso de uso do cache, onde $r_0$ é gigantesca e todas as outras são desprezíveis. Nesse contexto podemos tomar $r_0 = C$ e $\forall k > 0 r_k = 0$. Assim, podemos calcular o limite superior de custo como:

$$ H_1 \land H_2 \land H_3 \land H_4 \implies P \lessapprox \rho_mC + \rho_{cr} \sum_{k=0}^R C = \rho_mC + \rho_{cr} R C $$


### Hipótese 5: Custo K para 1

Com a sexta hipótese podemos definir a hipótese comum:

$$H_c = H_1 \land H_2 \land H_3 \land H_4 \land H_5$$

Onde $H_5$ diz que o custmo médio superior da geração e leitura de conteúdo é K vezes maior que o custo para leitura de cache. Ou seja:

$$\rho = \rho_{cr} = { \rho_m \over K }$$

Com ela introduzimos um conceito de carga de trabalho $L$ que traz uma aproximação proporcional ao limite superior de custo de uma sessão que pode ser definida como:

$$H_c \implies P \lessapprox \rho L$$

Onde:

$$L = (K + R) C$$

Assim podemos usar $L$ como uma medida extremamente útil para estimar custos de sessões. Ao analisar os custos de serviços como o *Claude* podemos chegar a valores tipicos de K:

|Serviço/Modelo|$\rho_{it}$|$\rho_{ot}$|$\rho_{cw}$|$\rho_{cr}$|
|--------------|-----------|-----------|-----------|-----------|
| Claude Fable | $10$      | $50$      | $20$      | $1$       |
| Claude Opus 5| $5$       | $25$      | $10$      | $0.50$    |

Assim $K = { \rho_m \over \rho } = { max((\rho_{it} + \rho_{cw}), (\rho_{ot} + \rho_{cw})) \over \rho_cr } = { max(70, 30) \over 1 } = 70$

Considerando o valor de pior caso. Muitas vezes usar valores médios para $\rho_m$ ao invés do máximo, ou seja, tomando $K = 50$ trás aproximações mais realistas, mas que por vezes podem subestimar os custos. Na prática existe um $\alpha$ de tal forma que:

$$P \approx \alpha L$$

Essa equação também mostra o caracter quadrático do consumo de recursos em função da quantidade de trabalho que é feito na sessão, já que R e C sobem juntas e correlacionadas com o uso. Por consequência:

$$L \in O(R C)$$

## Custo Exploratório e Limpeza de Sessão

Um tipo de fluxo de trabalho comum é o trabalho exploratório, onde o modelo precisa conhecer todo o contexto antes de começar a trabalhar. Esse tipo de fluxo precisa de muita entrada e leva a pouca entrada. Um processo eficiente pode levar a poucas requisições se bem feita. O caso perfeito é onde todo conteúdo é passado totalmente de tal modo que o modelo não precise fazer buscas e realizar várias requisições.

Quando uma sessão cresce muito, uma decisão comum é a de iniciar uma nova sessão, repassando conteúdo realizando uma processo exploratório. De tal forma, podemos considerar dois cenários possíveis: 1.Continuar a trabalhar na mesma sessão; 2.Limpar a sessão e começar do zero. Podemos avaliar os custos de ambos os cenários:

- Continuar na mesma sessão: Temos um custo inicial $L_0$ e fazemos uma nova interação que resulta em novas requisições $\Delta R$ e novo contexto $\Delta C$. Assim:

$$ L_{keep} = (K + R_0 + \Delta R) (C_0 + \Delta C) $$

- Limpar a sessão: Limpa a sessão passando um novo conteúdo exploratório $E$ e depois realizando a interação:

$$ L_{clear} = L_0 + L' = (K + R_0) C_0 + (K + R_0)(E + \Delta C) $$

Assim, o ganho de limpar a sessão é:

$$ G = L_{clear} - L_{keep}$$

$$ G = (K + R_0) C_0 + (K + R_0)(E + \Delta C) - (K + R_0 + \Delta R) (C_0 + \Delta C) $$

Analisando os termos positivos e negativos:

$$ G_+ = K C_0 + R_0C_0 + KE + K \Delta C + R_0 E + R_0 \Delta C $$

$$ G_- = K C_0 + K \Delta C + R_0 C_0 + R_0 \Delta C + \Delta R C_0 + \Delta R \Delta C $$

Assim:

$$ G = KE + R_0 E - \Delta R C_0 - \Delta R \Delta C $$

$$ G = (K + R_0) E -  (C_0 + \Delta C) \Delta R $$

Queremos descobrir se $G > 0$, ou seja, se vale a pena limpar a sessão, ou seja, descobrir se:

$$ (K + R_0) E > (C_0 + \Delta C) \Delta R $$

## Testes Práticos

*Temporário*

> ls "$HOME\.claude\projects" -r -Filter *.jsonl | sort LastWriteTime | select -Last 1 | % { gc $_.FullName } | % { ConvertFrom-Json $_ } | ? { $_.type -eq 'assistant' } | % requestId | sort -Unique | measure | % Count

Abaixo testes retirados de casos de uso real.

| P (US$) | C (tokens) |  R  |   L   | $\alpha$ |
| :-----: | :--------: | :-: | :---: | :------: |
|   15,40 |       288k | 117 |  48 M |     0,32 |
|   64,40 |       500k | 348 | 199 M |     0,32 |
|    2,58 |       100k |  20 |   7 M |     0,37 |
|   27,40 |       371k | 182 |  86 M |     0,32 |
|    3,02 |       104k |  46 |  10 M |     0,30 |
|   10,30 |       200k | 110 |  32 M |     0,32 |