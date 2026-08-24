# artigo-llm

## Calculo de Preço

A maioria dos serviços de LLM's como da OpenAI ou Anthropic usam um modelo de cálculo de consumo baseado em quantidades de tokens, em geral, 4 modalidades:

- Tokens de Entrada ou *Input Tokens* ($IT$): Tokens que são adicionados na janela de contexto, seja por entrada do usuário ou por arquivos lidos. Ou seja, todo token que entra na janela de contexto que não precisa ser gerado pela LLM.
- Tokens de Saída ou *Output Tokens* ($OT$): Tokens que são gerados pela LLM, seja em resposta ou pensamento interno. Ou seja, necessariamente tokens que precisam ser calculados pela inferência do modelo.
- Leituras em Cache ou *Cache Read* ($CR$): A quantidade de tokens lidas do cache.
- Escritas em Cache ou *Cache Write* ($CW$): A quantidade de tokens que é escrita para poder ser reaproveitada em requisições futuras.

Considerando que cada serviço usa preços diferentes podemos equacionar o custo geral de uma LLM como:

$$ P = \rho_{it} IT + \rho_{ot} OT + \rho_{cr} CR + \rho_{cw} CW $$

Onde, $\rho_{n}$ é o custo por token de $n$. Neste cenário existem ainda alguns tokens que já são carregados no ínicio da sessão que podem ou não estarem em cache. É comum que alguns recursos estáveis sejam sempre reaproveitados em todas as sessões abertas e por isso são muitas vezes colocados em cache. Denotaremos esse recursos como base cacheada ($CB$) e base não cacheada ($UB$). Desta forma, base cacheada soma futuramente como novas leituras de cache, enquanto base não cacheada será contabilizada rapidamente como conteúdo que será cacheado nas primeiras requisições. Esses custos, em geral, podem ou não estar inclusos como Tokens de Entrada. Isso traz um pouco de incerteza para equação que iremos tratar nas seções seguintes.

## Hipóteses de Uso e Carga de Trabalho

Apesar da simplicidade, a equaçõe pode não ser suficientemente útil para tomar decisões em contextos práticos. Contudo, podemos assumir várias hipóteses afim de simplificar as análises e chegar a conclusões práticas que valem para grande maioria das situações e serviços de LLM do mercado.

### Hipótese 1: Cache nunca inválida e toda entrada e saída são cacheados

A primeira hipótese ($H_1$) é bem simples e as simplificações que partem dela são válidas se o usuário nunca permite que o cache seja invalida e perdido. Usando *Claude Code* numa versão paga, por exemplo, conteúdos permanescem em Cache por até 1 hora e tem tempo restabelecido em cada interação com a LLM.Desta maneira, todo token de entrada ou saída serão escrito em cache em algum momento.

Além disso, em muitos sistemas, o conteúdo só é cacheado quando ele precisa ser reutilizado pela primeira vez. Isso implica que existe tokens de saída da última requisição ($UR$) que nunca irão para o cache. Outras lógicas podem ser aplicadas, por isso chamamos essa variáveis de resíduo não cacheado.

Tomando essa hipótese temos:

$$ H_1 \implies P = (\rho_{it} + \rho_{cr}) IT + (\rho_{ot} + \rho_{cr}) OT + \rho_{cw} CW - (\rho_{cr}) (CB + UR) $$

Ou seja, o preço da escrita em cache é embutida nos Tokens de Entrada. Em alguns serviços isso já acontece naturalmente. E surgem conteúdos que agora são calculados duas vezes, dado isso, existe um termo negativo que desconta conteúdo que nunca vai para cache ou que já estava em cache. Note que depender do sistema, o resíduo não cacheado ou a base cacheada podem ser efetivamente zero.

### Hipótese 2: A sessão é longa o suficiente

A segunda hipótese ($H_2$) implica que a sessão é longa o suficiente para que o residuo e o conteúdo base sejam desprezíveis. Neste caso podemos aproximar o preço da sessão considerando que $CB, UB, UR \approx 0$. Note que para uma sessão longa, isso não afeta $IT$ que é grande o suficiente para podermos ignorar os tokens de base que o compõe. Isso também simplifica cálculos onde o conteúdo base não é considerado como Token de Input. Também podemos definir um preço único de escrita em cache e geração de entrada ou saída da seguinte forma:

$$ H_1 \land H_2 \implies P \approx \rho_i IT + \rho_o OT + \rho_{cw} CW $$

### Hipótese 3: Entrada e saída compõe o contexto

A terceir hipótese aponta um cenário comum onde tudo que está na janela de contexto $C$ é praticamente constuida da entrada e saída do modelo, ou seja, assumir que $C \approx IT + OT$. Essa hipótese é impossível se existe compactação da janela de contexto, comum em alguns fluxos de trabalho.

Em muitos contextos de uso podemos aproximar, com um erro controlável que $\rho_i IT + \rho_o OT \lessapprox \rho (IT + OT) \approx \rho C$. Essa aproximação é util para definir um limite superior, considerando que geralmente o custo do token de saída e maior que o token de entrada. De forma geral tomamos $\rho = max(\rho_i, \rho_o)$. Unindo a terceira hipótese temos:

$$H_1 \land H_2 \land H_3 \implies P \lessapprox \rho C + \rho_{cw} CW $$

### Hipótese 4: Uso comum do cache

O funcionamento comum da leitura de cache pode surpreender alguns usuários, uma vez que uma única interação do usuário pode levar várias requisições que podem ser desde pensamento do modelo quando leitura de arquivos, escrita em arquivos, requisições http buscando conteúdo na internet etc. E, em geral entre os serviços de LLM, a leitura de cache ocorre por todo conteúdo anterior a cada requisição. Isso significa que podemos ter várias leituras dos mesmos tokens várias vezes. Podemos propor uma aproximção eficiente para leituras de cache em termos dos tamanhos da adição de conteúdo entre as requisições. Considerando $r_n$ o consumo de tokens da n-ésima requisição, e considerando a última requisição desprezível graças a $H_2$, podemos escrever o preço considerando $R$ requisições como:

$$H_1 \land H_2 \land H_3 \land H_4 \implies P \lessapprox \rho C + \rho_{cw} \sum_{k=0}^R \sum_{j=0}^k r_j $$

### Hipótese 5: Pior caso de requisições

Outra aproximação de limite superior útil é analisar o pior caso de uso do cache, onde $r_0$ é gigantesca e todas as outras são desprezíveis. Nesse contexto podemos tomar $r_0 = C$ e $ \forall k > 0 r_k = 0$. Assim, podemos calcular o limite superior de custo como:

$$ H_1 \land H_2 \land H_3 \land H_4 \land H_5 \implies P \lessapprox \rho C + \rho_{cw} \sum_{k=0}^R C = \rho C + \rho_{cw} R C $$


### Hipótese 6: Custo K para 1

Com a sexta hipótese podemos definir a hipótese comum:

$$H_c = H_1 \land H_2 \land H_3 \land H_4 \land H_5 \land H_6$$

Onde $H_6$ diz que o custmo médio superior da geração e leitura de conteúdo é K vezes maior que o custo para leitura de cache. Com ela introduzimos um conceito de carga de trabalho $L$ que traz uma aproximação proporcional ao limite superior de custo de uma sessão que pode ser definida como:

$$H_c \implies P \lessapprox \rho_{cw} L$$

Onde:

$$L = (K + R) C$$

Assim podemos usar $L$ como uma medida extremamente útil para estimar custos de sessões. Para muitos casos realistas, um bom valor para K é 50.

Essa equação também mostra o caracter quadrático do consumo de recursos em função da quantidade de trabalho que é feito na sessão, já que R e C sobem juntas e correlacionadas com o uso. Por consequência, dado um fator de trabalho $\omega$:

$$L \in O(\omega^2)$$

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
