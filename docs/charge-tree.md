# Charge Trees: definição e fundamentos

## 1. Escopo

Este documento registra a ideia geral das **Charge Trees** independentemente de uma geometria, métrica ou tarefa específica. A **Dimensional Charge Tree (DCT/KDCT)** é uma instanciação desse modelo e será descrita separadamente.

O objetivo aqui não é reivindicar originalidade nem apresentar resultados experimentais. A proposta é formalizar uma família de árvores cuja estrutura se adapta localmente a um fluxo de dados por meio de uma noção explícita de **carga**.

## 2. Ideia central

Uma Charge Tree separa duas decisões que muitas estruturas de dados tratam de maneira acoplada:

1. **Quando uma região deve mudar?** — decidido pela função de carga.
2. **Como essa região deve mudar?** — decidido pela política de divisão, fusão ou reorganização.

Em notação compacta:

\[
\underbrace{\phi(S_v)}_{\text{quando reorganizar}}
\qquad\text{e}\qquad
\underbrace{\Pi(S_v)}_{\text{como dividir}}.
\]

Essa separação é a essência da família. Uma KD-tree, uma árvore métrica ou uma árvore de classificadores locais podem originar variantes diferentes de Charge Tree se adotarem manutenção local controlada explicitamente pela carga.

## 3. Definição geral

Uma **Charge Tree** é uma árvore enraizada e dinamicamente mantida na qual:

- cada folha \(v\) representa uma região lógica e armazena um conjunto ou multiconjunto de elementos \(S_v\);
- cada nó interno representa uma decomposição de sua região entre nós filhos;
- uma função de carga \(\phi\) associa o estado local de um nó a um valor de carga \(\chi_v\);
- uma capacidade superior \(C\), ou uma condição equivalente, determina quando uma folha está sobrecarregada;
- uma política de divisão \(\Pi\) substitui uma folha sobrecarregada por uma decomposição local;
- opcionalmente, um limiar inferior \(Y<C\) e uma política de fusão recompõem regiões cuja carga diminuiu;
- inserções e remoções modificam somente o caminho e a vizinhança estrutural afetados, salvo quando uma implementação escolhe realizar uma reconstrução mais ampla.

Para uma folha \(v\):

\[
\chi_v = \phi(S_v, A_v),
\]

onde \(A_v\) representa estado auxiliar opcional, como estatísticas, resumos, histogramas, modelos locais ou informação temporal.

A forma mais simples é:

\[
\phi(S_v)=|S_v|.
\]

Entretanto, a carga não precisa ser apenas uma contagem de elementos.

## 4. Componentes fundamentais

### 4.1 Elemento

Um elemento \(x\) é a unidade armazenada pela árvore. Dependendo da instanciação, ele pode ser:

- um vetor de atributos;
- um objeto em um espaço métrico;
- uma amostra rotulada;
- um evento de fluxo com identificador e instante de chegada;
- outra representação aceita pelas funções de roteamento e consulta.

A identidade do elemento e sua representação para roteamento não precisam ser a mesma coisa. Um identificador pode ser usado para remoção enquanto atributos, distâncias ou regras aprendidas determinam o caminho na árvore.

### 4.2 Região

Cada nó representa uma região lógica \(R_v\) do domínio. O significado de região pertence à instanciação:

- intervalo ou hiper-retângulo em uma árvore dimensional;
- faixa definida por centro e raio em uma árvore métrica;
- subconjunto de condições em uma árvore de classificação;
- qualquer partição para a qual existam regras de roteamento consistentes.

Um nó interno não precisa armazenar elementos diretamente. Seu papel mínimo é guardar a decisão que encaminha um elemento a um de seus filhos.

### 4.3 Carga

A carga mede a pressão estrutural ou semântica exercida sobre uma região. Exemplos de função de carga incluem:

- quantidade de elementos;
- quantidade de representações distintas;
- peso ou volume acumulado;
- impureza de classes;
- erro de um modelo local;
- instabilidade temporal ou evidência de *concept drift*;
- combinação de massa, impureza, erro e tempo.

Um exemplo supervisionado seria:

\[
\phi(v)=\alpha\,\operatorname{massa}(v)
       +\beta\,\operatorname{impureza}(v).
\]

Assim, duas folhas com o mesmo número de elementos podem ter cargas diferentes. Uma região grande e homogênea pode permanecer intacta, enquanto uma região menor, mas conflitante, pode ser dividida.

### 4.4 Capacidade e histerese

A capacidade \(C\) é o limiar superior de carga. A forma básica da condição de split é:

\[
\chi_v \ge C.
\]

Uma implementação deve declarar com precisão se a divisão ocorre em \(\chi_v=C\) ou somente quando \(\chi_v>C\). Esta convenção afeta o número máximo de elementos transitórios na folha, mas não altera o modelo geral.

Quando há fusão, usa-se um limiar inferior \(Y\), normalmente com:

\[
Y<C.
\]

A diferença entre \(Y\) e \(C\) cria **histerese**. Ela evita que pequenas variações de carga provoquem uma alternância contínua entre split e merge.

### 4.5 Política de roteamento

A função de roteamento de um nó interno seleciona o filho que deve receber um elemento:

\[
\rho_v(x) \in \operatorname{children}(v).
\]

Para que a estrutura permaneça correta, a regra usada em inserções futuras deve ser compatível com a partição criada durante o split.

Em aplicações supervisionadas, rótulos podem participar da carga ou da escolha do split durante o treinamento. Contudo, o roteamento de uma amostra desconhecida deve usar apenas informação disponível no momento da predição.

### 4.6 Política de divisão

A política de divisão recebe o estado de uma folha e produz uma decisão de roteamento e dois ou mais filhos:

\[
\Pi(v,S_v)=(\theta_v,v_1,\ldots,v_m),
\]

onde \(\theta_v\) descreve a regra instalada no novo nó interno.

Uma divisão válida deve:

1. produzir pelo menos dois grupos não vazios, salvo se a variante definir explicitamente outro comportamento;
2. atribuir cada elemento da folha a um filho;
3. criar uma regra capaz de encaminhar inserções futuras;
4. preservar os invariantes da instanciação;
5. terminar mesmo quando os dados não forem separáveis.

O último item é essencial. Se todos os elementos forem indistinguíveis para a política de split, a árvore não deve repetir divisões vazias indefinidamente. Possíveis tratamentos são aceitar temporariamente uma folha acima da capacidade, armazenar multiplicidades ou selecionar outra regra de divisão.

### 4.7 Política de fusão

O merge é opcional na definição mais ampla, mas é especialmente importante para fluxos com expiração de elementos.

Uma política de fusão \(M\) pode substituir um nó interno e seus descendentes imediatos por uma folha quando a carga combinada se torna suficientemente baixa:

\[
\phi\!\left(\bigcup_i S_{v_i}\right) \le Y.
\]

O merge deve preservar todos os elementos ativos, reconstruir ou combinar a EI e atualizar os resumos locais. A fusão pode ser limitada a irmãos, a uma subárvore pequena ou a outra unidade estrutural definida pela variante.

### 4.8 Estrutura Interna (EI)

A **Estrutura Interna (EI)** é o mecanismo local usado por uma folha para armazenar, organizar e acessar seus elementos. Ela é uma abstração, não uma estrutura ou um índice fixo.

A separação entre a árvore e a EI permite que:

- a Charge Tree cuide da organização entre regiões;
- a EI cuide das operações dentro de uma folha;
- diferentes cargas de trabalho usem representações locais diferentes sem alterar a definição da árvore.

Uma interface conceitual mínima para a EI é:

```text
insert(x)             insere um elemento
remove(id ou x)       remove uma ocorrência ou identidade específica
elements()            percorre os elementos ativos
size()                informa a quantidade armazenada
chargeState()         fornece o estado necessário para calcular a carga
partition(θ)          distribui os elementos segundo uma decisão de split
merge(EI_1, ..., EI_m) combina as EIs durante uma fusão
```

Operações opcionais incluem:

```text
contains(x)
count(x)
query(q)
summary()
compact()
```

Possíveis implementações da EI incluem vetor, lista, tabela de dispersão, multiconjunto, buffer ordenado, pequeno índice local ou modelo especializado. A escolha altera os custos locais, mas não a identidade da Charge Tree.

## 5. Invariantes básicos

Uma Charge Tree correta deve explicitar e preservar pelo menos os seguintes invariantes.

### 5.1 Pertencimento único

Cada elemento ativo pertence logicamente a uma única folha, exceto quando uma variante declara replicação como parte de seu modelo.

### 5.2 Roteamento consistente

Para todo elemento \(x\) armazenado sob um nó interno \(v\), a decisão \(\rho_v(x)\) deve selecionar o descendente que contém \(x\).

### 5.3 Conservação no split e no merge

Split e merge não podem criar nem perder elementos:

\[
S_v = \biguplus_i S_{v_i},
\]

considerando multiplicidades quando \(S_v\) for um multiconjunto.

### 5.4 Carga coerente

Depois de uma inserção, remoção, divisão ou fusão, a carga armazenada ou calculada deve corresponder ao estado atual da folha.

### 5.5 Divisão realizável

Ultrapassar a capacidade não implica que uma divisão válida sempre exista. A implementação precisa distinguir **folha sobrecarregada** de **folha separável**.

### 5.6 Resumos seguros

Se um resumo de nó for usado para poda exata, ele deve fornecer um limite comprovadamente seguro. Resumos heurísticos podem orientar a ordem de visita, mas não devem eliminar resultados potencialmente válidos em uma consulta declarada exata.

## 6. Operações fundamentais

### 6.1 Localização de folha

```text
locateLeaf(x):
    v ← raiz
    enquanto v for interno:
        v ← ρ_v(x)
    retorne v
```

O custo depende da altura \(h\) e do custo \(R\) de uma decisão de roteamento:

\[
T_{route}=O(hR).
\]

### 6.2 Inserção

```text
insert(x):
    v ← locateLeaf(x)
    EI_v.insert(x)
    atualize carga e resumos no caminho afetado
    se v estiver sobrecarregada e for separável:
        split(v)
```

A inserção ordinária custa:

\[
O(hR + I_{EI} + U),
\]

onde \(I_{EI}\) é o custo de inserir na EI e \(U\) é o custo de atualizar carga e resumos. Quando ocorre split, soma-se o custo \(P(|S_v|)\) da política de partição.

### 6.3 Split

```text
split(v):
    decisão, grupos ← Π(v, EI_v.elements())
    se a partição não for válida:
        mantenha v como folha não separável por essa política
        retorne
    crie os filhos e suas EIs
    distribua todos os elementos
    substitua v pela decisão de roteamento
    recompute carga e resumos locais
```

O split é **local**: ele não exige, por definição, reconstruir a árvore inteira.

### 6.4 Remoção

Se a remoção recebe apenas o elemento, pode ser necessário localizá-lo novamente. Em fluxos FIFO ou janelas deslizantes, um mapa externo opcional pode associar o identificador do elemento à folha que o contém.

```text
remove(id ou x):
    v ← localize o elemento
    EI_v.remove(id ou x)
    atualize carga e resumos
    avalie fusões a partir de v
```

Sem referência direta, o custo inclui a busca pela folha ou pelo elemento. Com um mapa `id → folha`, a localização pode ser constante em média, restando o custo de remoção da EI e da manutenção estrutural.

O mapa de referências é uma otimização operacional, não parte obrigatória da definição de Charge Tree.

### 6.5 Merge

```text
tryMerge(v):
    p ← pai de v
    enquanto p puder ser substituído por uma folha:
        se carga combinada dos descendentes elegíveis ≤ Y:
            p ← merge(descendentes)
        senão:
            pare
```

Uma variante deve declarar:

- quais descendentes podem ser fundidos;
- se todos precisam ser folhas;
- como combinar suas EIs;
- como reconstruir carga e resumos;
- se a fusão pode continuar em direção à raiz.

### 6.6 Consulta

A definição geral de Charge Tree não impõe uma consulta específica. Uma consulta pode:

- seguir uma única rota até uma folha;
- visitar várias regiões por *branch-and-bound*;
- consultar um modelo ou histograma local;
- combinar respostas de várias folhas.

Consequentemente, o algoritmo e o custo de consulta pertencem à instanciação, não ao núcleo abstrato da Charge Tree.

## 7. Manutenção em fluxo de dados

Charge Trees foram concebidas com fluxos de dados em mente. Em uma janela deslizante, cada novo elemento pode provocar uma inserção e cada expiração uma remoção:

```text
para cada novo elemento x_t:
    insert(x_t)
    se a janela exceder o limite:
        remove(elemento_mais_antigo)
```

Os efeitos estruturais permanecem locais:

- crescimento de carga pode provocar split;
- redução de carga pode provocar merge;
- mudanças de distribuição aparecem como reorganizações nas regiões afetadas;
- a estrutura não depende de reconstruções globais obrigatórias.

Isso não garante balanceamento global nem elimina a possibilidade de reconstruções ocasionais. A garantia conceitual é mais modesta: a unidade normal de adaptação é local.

## 8. Custos: forma paramétrica

Como a Charge Tree é um framework, seus custos devem ser expressos em função das escolhas concretas. Sejam:

- \(h\): altura da árvore;
- \(R\): custo de uma decisão de roteamento;
- \(I_{EI}\): custo de inserção na EI;
- \(D_{EI}\): custo de remoção na EI;
- \(Q_{EI}\): custo de consulta local;
- \(P(n)\): custo de dividir uma folha com \(n\) elementos;
- \(M(n)\): custo de fundir regiões com \(n\) elementos totais.

Então:

| Operação | Custo geral |
|---|---:|
| Roteamento até uma folha | \(O(hR)\) |
| Inserção sem split | \(O(hR+I_{EI}+U)\) |
| Inserção com split | \(O(hR+I_{EI}+U+P(n))\) |
| Remoção com referência direta | \(O(D_{EI}+U)\), mais eventual merge |
| Remoção sem referência | custo de localização + \(O(D_{EI}+U)\) |
| Merge | \(O(M(n))\) |
| Consulta | custo de navegação da variante + \(Q_{EI}\) nas folhas visitadas |

Não se deve afirmar \(O(\log N)\) para operações gerais sem antes estabelecer propriedades de altura e balanceamento da variante concreta.

## 9. Propriedades e limites do modelo

### 9.1 Adaptação local

A unidade básica de mudança é uma região. Isso favorece fluxos em que somente parte do espaço muda em determinado momento.

### 9.2 Reversibilidade estrutural

Quando split e merge são ambos definidos, uma região pode especializar-se e posteriormente voltar a uma representação mais simples. A reversão não precisa restaurar a mesma representação em memória; precisa preservar o conjunto lógico de elementos e os invariantes.

### 9.3 Independência entre carga e partição

A mesma política geométrica pode ser combinada com cargas diferentes. De modo inverso, a mesma função de carga pode controlar árvores com regras de partição distintas.

### 9.4 Ausência de garantia automática de balanceamento

Limitar a carga das folhas não implica, sozinho, limitar a altura. A política de split precisa produzir partições adequadas se a variante desejar garantias de balanceamento.

### 9.5 Separação entre exatidão e heurística

Uma heurística pode escolher o filho mais promissor ou ordenar a exploração. Para podar uma região em uma consulta exata, entretanto, é necessária uma condição segura que prove que ela não contém uma resposta melhor.

## 10. Instanciações possíveis

### 10.1 Charge Tree dimensional

Usa vetores, cortes em eixos e regiões espaciais. Essa é a família da DCT/KDCT.

### 10.2 Charge Tree métrica

Usa distâncias, representantes e raios para particionar um espaço métrico. Uma VP-tree dinamizada por carga, split local e merge seria uma possível instanciação. Isso não torna "VP Charge Tree" um nome estabelecido na literatura.

### 10.3 Charge Tree de classificação

Cada folha pode manter um histograma ou um *expert* local. A carga pode incorporar massa, impureza, erro ou drift. Os rótulos orientam a manutenção durante o treino, mas não podem ser exigidos para rotear uma amostra ainda não classificada.

## 11. O que pertence ao núcleo da definição

Pertencem ao núcleo:

- folhas com elementos ou representações locais;
- uma função explícita de carga;
- uma condição de sobrecarga;
- uma política local de reorganização;
- roteamento consistente após a reorganização;
- preservação dos elementos e dos invariantes.

São extensões ou escolhas de variante:

- merge e limiar \(Y\);
- tipo concreto da EI;
- mapa `id → folha`;
- resumos de região;
- *branch-and-bound*;
- métrica de distância;
- boxes, pivôs, histogramas ou experts;
- políticas de compactação e reconstrução.

## 12. Definição curta

> Uma Charge Tree é uma árvore adaptativa em que uma função explícita de carga decide quando uma representação local deve ser reorganizada, enquanto uma política independente determina como dividi-la, fundi-la ou substituí-la. Seus elementos são mantidos em folhas por meio de uma Estrutura Interna configurável, e sua manutenção normal ocorre localmente em resposta a inserções, remoções ou mudanças do estado observado.
