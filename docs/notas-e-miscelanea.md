# Notas e miscelânea

## Como a ideia surgiu

Como curiosidade, a linha de raciocínio que levou a esta estrutura começou com uma tentativa de generalizar árvores de busca binária para uma base \(k\) qualquer. Depois de algum tempo e várias tentativas malsucedidas, não cheguei a uma função de roteamento que realmente compensasse: um simples operador \(\ge\), afinal, é uma ferramenta de decisão difícil de superar.

A segunda tentativa surgiu quando me deitei para dormir naquele mesmo dia. A princípio, eu pretendia apenas guardar a ideia para outra hora, mas acabei continuando a pensar sobre ela. Em resumo, essa nova proposta convergiu para uma estrutura que já existia: a *radix tree*. Curiosamente, antes mesmo de pesquisar estruturas semelhantes, cheguei de forma independente a melhorias que de fato aparecem em *radix trees*, como o uso de cache, além de considerações semelhantes sobre elas só se tornarem relevantes a partir de um \(n\) consideravelmente elevado.

Deixando de lado essa ideia já existente, a terceira iteração consistiu em tentar usar divisões por “quadrantes”. Os sinais das dimensões funcionariam como mecanismo de roteamento para uma árvore de busca binária unidimensional. A ideia, por si só, não me parecia ruim. No entanto, os quadrantes sofriam essencialmente do mesmo problema de poda das KD-trees convencionais, além de aparentemente já existirem estruturas semelhantes.

Ainda nessa última linha de raciocínio, enquanto brincava com árvores unidimensionais, surgiu a ideia de usar vetores com subdivisão periódica. Ela veio do mesmo caminho que anteriormente havia me levado a convergir para as *radix trees*. Após um raciocínio rápido, apareceu a pergunta:

> Os quadrantes não seguem, basicamente, a mesma ideia?

A partir daí surgiu a proposta de combinar árvores controladas por carga e *splits* locais com a técnica de divisão por eixos das KD-trees. A intuição era que, ao reunir uma quantidade relevante de entradas em um bloco, seria possível realizar um corte mais bem informado, permitindo que a busca efetuasse uma poda realmente útil.

Ao mesmo tempo, como divisões em um eixo produzem blocos com comportamento previsível, uma perda significativa de elementos em um dos lados poderia ser tratada simplesmente pela reconstrução do bloco original. O uso de blocos também torna a exclusão mais simples e limpa, dependendo da maneira como esses blocos são implementados.

Hoje percebo alguma semelhança entre essa linha de raciocínio e as árvores B. Essa aproximação, porém, nunca foi a intenção original da proposta.

## Notas sobre experimentos

Embora os experimentos realizados durante o desenvolvimento da ideia não estejam disponíveis neste repositório, algumas observações recorrentes sobre a estrutura das DCTs merecem ser registradas:

1. **Sensibilidade à dimensionalidade.** As DCTs sofrem com o aumento da dimensionalidade, assim como diversas outras estruturas de indexação espacial. À medida que o número de dimensões cresce, as regiões tendem a se sobrepor mais durante a busca e os limites usados para poda se tornam menos informativos.

2. **Balanceamento observado nos fluxos.** Em diferentes fluxos de dados, as DCTs permaneceram relativamente bem balanceadas mesmo sem um mecanismo explícito de balanceamento global. Esse comportamento não foi investigado formalmente, embora pareça intuitivamente relacionado aos splits locais realizados quando as folhas atingem determinada carga.

3. **Tempo e espaço em comparação com a KD-tree online.** Na maioria dos testes realizados, as DCTs foram mais rápidas e utilizaram menos espaço que a implementação de KD-tree online empregada como referência.

Essas observações devem ser entendidas apenas como registros informais da experimentação. Como os códigos, protocolos e resultados correspondentes não estão disponíveis, elas não constituem evidência reproduzível nem permitem conclusões gerais sobre o desempenho da estrutura.
