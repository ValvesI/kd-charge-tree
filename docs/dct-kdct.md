# DCT/KDCT: Dimensional Charge Tree

## 1. Escopo e terminologia

A **Dimensional Charge Tree (DCT)** é uma instanciação espacial da ideia de Charge Tree para elementos representados por vetores em \(\mathbb{R}^d\). O nome **KDCT**, ou *k-dimensional Charge Tree*, enfatiza essa relação com espaços de \(k\) dimensões e com a divisão por eixos herdada das KD-trees.

Neste documento, **DCT** e **KDCT** referem-se à mesma ideia central. DCT será usado como nome curto.

O objetivo não é apresentar resultados experimentais nem todas as variações exploradas. O texto descreve:

- a DCT básica;
- sua manutenção por carga;
- suas Estruturas Internas;
- o uso de boxes para consultas;
- a extensão **DCT com Box Adjust (DCTBA)**.

A definição geral de Charge Tree está em [charge-tree.md](charge-tree.md).

## 2. Ideia central

A DCT combina dois mecanismos:

1. **partição espacial por eixos**, inspirada em uma KD-tree;
2. **manutenção local por carga**, proveniente das Charge Trees.

Em vez de construir uma árvore completa para um conjunto estático, a DCT começa como uma folha. As instâncias são inseridas em sua **Estrutura Interna (EI)** e aumentam sua carga. Quando a capacidade é atingida, somente aquela folha é dividida. Se remoções reduzirem suficientemente a carga de regiões irmãs, elas podem ser fundidas novamente.

O ciclo fundamental é:

```text
inserção
    → aumento de carga
    → split local
    → novas regiões

remoção
    → redução de carga
    → merge local
    → região recomposta
```

Assim, a geometria descreve **como** dividir e rotear, enquanto a carga decide **quando** a estrutura deve mudar.

## 3. Elementos e espaço de atributos

Cada elemento armazenado possui um vetor usado pela árvore:

\[
x=(x_0,x_1,\ldots,x_{d-1})\in\mathbb{R}^d.
\]

O elemento também pode carregar informações que não pertencem ao espaço indexado, como:

- identificador;
- instante de chegada;
- classe;
- peso;
- outros metadados.

Esses dados auxiliares podem ser usados pela aplicação, mas não precisam participar do roteamento, das boxes ou da distância. Em classificação por kNN, por exemplo, a classe acompanha a instância, mas não constitui uma dimensão de busca.

## 4. Estrutura dos nós

### 4.1 Folha

Uma folha \(v\) representa uma região do espaço e contém:

- uma Estrutura Interna \(EI_v\);
- sua carga \(\chi_v\);
- sua região de roteamento;
- opcionalmente, uma box ajustada aos elementos vivos;
- referência para o pai e outros metadados de manutenção.

Os elementos ficam nas folhas. O tipo concreto da EI é configurável e não altera a definição estrutural da DCT.

### 4.2 Nó interno

Um nó interno registra uma divisão espacial. Na forma binária, ele mantém pelo menos:

- o eixo de corte \(a_v\);
- o valor de corte \(t_v\);
- o filho correspondente a valores inferiores;
- o filho correspondente a valores superiores;
- as regiões ou boxes associadas aos filhos.

Uma convenção de roteamento possível é:

\[
\rho_v(x)=
\begin{cases}
v_{\mathrm{left}}, & x_{a_v}<t_v,\\
v_{\mathrm{right}}, & x_{a_v}\ge t_v.
\end{cases}
\]

A escolha entre `<` e `≤` não é essencial, mas deve ser consistente durante a redistribuição do split e nas inserções futuras.

## 5. Carga da DCT

Seja \(S_v\) o conjunto ou multiconjunto de instâncias armazenadas na folha \(v\). Como em qualquer Charge Tree, a carga da DCT é definida por uma função configurável:

\[
\chi_v=\phi(S_v).
\]

A forma mais direta consiste em contar todas as instâncias:

\[
\phi(S_v)=|S_v|.
\]

Para dados que contêm muitas repetições, uma alternativa considerável é usar a quantidade de vetores de atributos diferentes:

\[
\phi_{\mathrm{distintos}}(S_v)
=\left|\operatorname{distinct}(S_v)\right|.
\]

Nessa configuração:

- instâncias duplicadas continuam armazenadas;
- todas as ocorrências permanecem disponíveis para consulta e remoção;
- inserir outra ocorrência de um vetor já presente não aumenta a carga estrutural;
- remover uma ocorrência somente reduz a carga quando ela era a última daquele vetor.

Essa opção pode evitar que muitas repetições de um mesmo ponto provoquem splits incapazes de criar partições espaciais úteis. Ela não é, entretanto, obrigatória para que uma estrutura seja considerada uma DCT.

## 6. Parâmetros estruturais

Os parâmetros principais são:

- \(d\): número de dimensões indexadas;
- \(C\): capacidade ou carga que dispara o split;
- \(SN\): número de sub-regiões produzidas por um split;
- \(Y\): limiar opcional para merge, com \(Y<C\).

Na forma binária:

\[
SN=2.
\]

Uma folha torna-se candidata a divisão quando:

\[
\chi_v\ge C.
\]

O split somente deve ocorrer se for possível produzir uma partição válida e não vazia. Uma folha pode atingir a capacidade sem ser separável no eixo atualmente considerado. Nesse caso, a implementação precisa tentar outro eixo, usar outra política válida ou manter temporariamente a folha acima da capacidade.

## 7. Seleção do eixo

A política mais diretamente relacionada à KD-tree percorre ciclicamente os eixos:

\[
a_v=\operatorname{depth}(v)\bmod d.
\]

Assim:

```text
profundidade 0 → eixo 0
profundidade 1 → eixo 1
⋮
profundidade d-1 → eixo d-1
profundidade d → eixo 0 novamente
```

Essa é uma política simples e determinística. A definição da DCT também comporta outras políticas, como selecionar um eixo separável ou um eixo com maior dispersão, desde que a decisão produza regiões consistentes para o roteamento futuro.

## 8. Split

### 8.1 Forma binária

Para uma folha \(v\) e um eixo \(a_v\):

1. obtêm-se os valores distintos presentes naquele eixo;
2. escolhe-se um corte, normalmente próximo da mediana;
3. criam-se duas folhas;
4. redistribuem-se todas as instâncias segundo o corte;
5. substitui-se a folha original por um nó interno;
6. calculam-se as cargas e boxes dos filhos.

Em notação:

\[
S_{\mathrm{left}}=\{x\in S_v:x_{a_v}<t_v\},
\]

\[
S_{\mathrm{right}}=\{x\in S_v:x_{a_v}\ge t_v\}.
\]

O corte deve produzir dois conjuntos não vazios. Valores duplicados exigem uma convenção estável, para que a regra instalada no nó reproduza a mesma distribuição em inserções posteriores.

### 8.2 Forma com \(SN\) filhos

Uma forma mais geral ordena os valores no eixo selecionado, determina \(SN-1\) limites e cria até \(SN\) intervalos contíguos. Cada intervalo se torna uma sub-região.

O split multiway continua obedecendo aos mesmos requisitos:

- nenhum filho criado deve ser vazio;
- toda instância deve pertencer a exatamente um filho;
- os limites precisam rotear corretamente novas instâncias;
- a operação permanece local à folha dividida.

### 8.3 Custo do split

Com uma EI não ordenada, selecionar a divisão por ordenação custa tipicamente:

\[
O(C\log C).
\]

Depois, a distribuição entre os filhos custa \(O(C)\). A EI pode empregar seleção linear, ordenação parcial ou outra estratégia, mas isso é uma decisão de implementação.

## 9. Inserção

```text
insert(x):
    v ← raiz
    enquanto v for interno:
        v ← filho selecionado pelo eixo e pelo corte de v

    EI_v.insert(x)
    atualize o estado necessário para calcular a carga de v
    atualize boxes vivas, se a variante as mantiver

    se χ_v ≥ C e existir uma partição válida:
        split(v)
```

O caminho usa apenas os cortes estáveis instalados nos nós internos. Inserir novos valores não desloca automaticamente cortes antigos.

## 10. Remoção

Uma remoção precisa identificar uma ocorrência específica ou um elemento equivalente:

```text
remove(id ou x):
    localize a folha que contém o elemento
    remova uma ocorrência da EI
    atualize multiplicidade e carga
    ajuste as boxes vivas, se existirem
    avalie um merge local
```

A localização pode repetir o roteamento espacial. Em uma janela FIFO, também é possível manter um mapa auxiliar:

```text
identificador da instância → folha atual
```

Esse mapa evita uma nova busca estrutural e precisa ser atualizado quando split ou merge moverem elementos.

Quando há duplicatas, remover uma ocorrência não deve apagar todas as demais. O efeito sobre a carga depende de \(\phi\). Na carga por instâncias, toda remoção reduz a carga; na carga por vetores diferentes, ela só diminui quando a última ocorrência daquele vetor deixa a folha.

## 11. Merge

O merge é a operação inversa de manutenção. Na forma binária, duas folhas irmãs podem substituir seu pai quando sua carga combinada fica abaixo do limiar \(Y\):

\[
\phi\!\left(S_{\mathrm{left}}\uplus S_{\mathrm{right}}\right)
\le Y.
\]

O merge:

1. combina todas as ocorrências das folhas;
2. constrói a EI da nova folha;
3. recalcula a carga por meio de \(\phi\);
4. remove a decisão de corte do pai;
5. recalcula a box viva, quando aplicável;
6. atualiza referências externas para a nova folha.

O limiar separado \(Y<C\) reduz a oscilação entre split e merge.

## 12. Regiões de roteamento e boxes vivas

Uma distinção importante é a existência de dois tipos de limite espacial.

### 12.1 Região estável de roteamento

A região de roteamento é definida pelos cortes dos ancestrais. Ela responde:

> Por qual caminho uma nova instância deve seguir?

Esses limites devem permanecer estáveis enquanto o nó existir. Encolhê-los com base apenas nos elementos atuais poderia tornar uma futura instância incompatível com o roteamento que originou a subárvore.

### 12.2 Box dos elementos vivos

A box viva resume os vetores atualmente armazenados sob um nó:

\[
B_v=\prod_{i=0}^{d-1}[\ell_{v,i},u_{v,i}],
\]

onde:

\[
\ell_{v,i}=\min_{x\in S_v}x_i,
\qquad
u_{v,i}=\max_{x\in S_v}x_i.
\]

Ela responde a outra pergunta:

> Qual é a menor distância que uma instância desta subárvore pode ter da consulta?

A box viva pode crescer com inserções e encolher com remoções. Ela serve para consulta e poda, não para redefinir o caminho estrutural de inserção.

## 13. DCT base e DCT com Box Adjust

### 13.1 DCT base

A DCT base pode usar diretamente as regiões induzidas pelos cortes para orientar a busca. Essas regiões são corretas, mas podem incluir grandes partes vazias do espaço.

Como os limites permanecem ligados ao histórico de splits, a remoção de elementos não necessariamente torna a box de consulta mais apertada.

### 13.2 DCTBA

A **DCT com Box Adjust (DCTBA)** mantém, além das regiões estáveis de roteamento, boxes ajustadas aos elementos vivos de cada subárvore.

Em uma inserção, os limites podem ser expandidos incrementalmente:

\[
\ell_{v,i}\leftarrow\min(\ell_{v,i},x_i),
\qquad
u_{v,i}\leftarrow\max(u_{v,i},x_i).
\]

Em uma remoção, se o elemento removido definia um mínimo ou máximo, o limite afetado precisa ser obtido novamente a partir dos elementos vivos ou de um resumo auxiliar.

O princípio da DCTBA é:

```text
cortes estáveis → roteamento correto
boxes vivas       → poda mais apertada
```

Box Adjust não altera quais elementos são vizinhos nem transforma a consulta em aproximada. Ele apenas melhora o limite usado para decidir quais regiões ainda podem conter resultados.

## 14. Limite inferior de uma box

Para uma consulta \(q\) e uma box \(B\), define-se:

\[
LB(q,B)=\min_{x\in B}d(q,x).
\]

Para métricas separáveis por coordenada, pode-se construir o ponto da box mais favorável à consulta:

\[
p_i=
\begin{cases}
\ell_i, & q_i<\ell_i,\\
q_i, & \ell_i\le q_i\le u_i,\\
u_i, & q_i>u_i.
\end{cases}
\]

Então:

\[
LB(q,B)=d(q,p).
\]

Para a distância euclidiana:

\[
LB_2(q,B)=\sqrt{\sum_i(q_i-p_i)^2}.
\]

Para Canberra:

\[
LB_C(q,B)=
\sum_i
\frac{|q_i-p_i|}{|q_i|+|p_i|},
\]

adotando contribuição zero quando numerador e denominador são ambos zero.

Se \(q_i\) já está dentro do intervalo da box, aquela dimensão contribui com zero para o limite inferior. Por isso, boxes largas ou muito sobrepostas produzem bounds fracos mesmo quando a consulta exata continua correta.

## 15. Busca kNN exata

A busca kNN pode usar *branch-and-bound* em ordem *best-first*. Existem duas filas com papéis diferentes:

1. **fila de regiões**, um min-heap ordenado por \(LB(q,B_v)\);
2. **heap de vizinhos**, um max-heap limitado aos \(k\) melhores elementos encontrados.

O pior elemento do heap de vizinhos define o raio atual \(\tau\). Enquanto ainda não existem \(k\) candidatos, usa-se \(\tau=+\infty\).

```text
knn(q, k):
    coloque a raiz na fila de regiões
    vizinhos ← vazio
    τ ← +∞

    enquanto a fila de regiões não estiver vazia:
        v ← remova a região com menor LB

        se já existem k vizinhos e LB(q, B_v) > τ:
            descarte v
            continue

        se v for interno:
            para cada filho u de v:
                calcule LB(q, B_u)
                se u ainda puder melhorar o resultado:
                    insira u na fila de regiões
        senão:
            para cada elemento x da EI_v:
                calcule d(q, x)
                atualize o heap dos k melhores
                atualize τ

    retorne os k melhores
```

Quando o menor limite inferior restante já é maior que \(\tau\), nenhuma outra região pode conter um vizinho estritamente melhor e a busca pode terminar.

Empates a distância \(\tau\) exigem uma convenção. Se a aplicação precisa preservar todos os empatados ou uma ordem determinística específica, regiões com \(LB=\tau\) não devem ser eliminadas prematuramente.

## 16. Consulta por raio

O mesmo limite inferior permite uma consulta por raio \(r\):

\[
LB(q,B_v)>r
\quad\Longrightarrow\quad
\text{a subárvore pode ser podada}.
\]

Um limite superior seguro pode mostrar que toda uma região cabe no raio, mas ele não é necessário para a poda básica do kNN. Para o kNN exato, o limite inferior é a informação essencial.

## 17. Invariantes

Uma DCT deve preservar os seguintes invariantes:

1. **Roteamento:** todo elemento armazenado sob um nó satisfaz as decisões de corte do caminho até sua folha.
2. **Pertencimento:** cada ocorrência ativa pertence a uma única folha.
3. **Multiplicidade:** duplicatas são preservadas, independentemente da maneira como contribuem para a carga.
4. **Carga:** \(\chi_v\) corresponde ao valor atual de \(\phi(S_v)\).
5. **Split:** uma divisão conserva todas as ocorrências e não cria filhos vazios.
6. **Merge:** uma fusão conserva todas as ocorrências dos descendentes substituídos.
7. **Região estável:** remoções não alteram retroativamente as decisões de roteamento.
8. **Box viva:** quando presente, \(B_v\) contém todos os elementos vivos da subárvore de \(v\).
9. **Poda segura:** uma região só é eliminada quando seu limite inferior prova que ela não pode melhorar a resposta atual.

## 18. Custos básicos

Sejam:

- \(h\): altura da árvore;
- \(d\): dimensionalidade;
- \(C\): capacidade da folha;
- \(V\): número de nós visitados em uma consulta;
- \(L\): número de folhas visitadas;
- \(n_j\): quantidade de ocorrências na folha visitada \(j\).

### 18.1 Inserção

O roteamento custa \(O(h)\) quando cada nó realiza uma comparação de eixo. Inserir numa EI vetorial pode custar \(O(1)\) amortizado. Atualizar uma box viva ao longo do caminho custa até \(O(hd)\).

Sem split:

\[
O(h+I_{EI})
\]

ou \(O(hd+I_{EI})\) quando boxes de todos os ancestrais são atualizadas explicitamente.

### 18.2 Split

Com ordenação completa da EI:

\[
O(C\log C).
\]

### 18.3 Remoção

Com mapa direto para a folha, a localização pode ser \(O(1)\) esperado. O restante depende da EI e da atualização das boxes. Se um extremo removido precisar ser redescoberto por varredura, o ajuste local pode custar \(O(Cd)\) numa folha.

### 18.4 Merge

Combinar duas folhas com carga total limitada por \(Y<C\) custa tipicamente \(O(C)\), além do custo de reconstrução da EI e da box.

### 18.5 kNN

O custo observado depende de quantas boxes e instâncias não puderam ser podadas. Uma expressão direta é:

\[
O\!\left(V(d+\log V)
+\sum_{j=1}^{L}n_jd
+K\log k\right),
\]

onde \(K\) é a quantidade de candidatos que provocam operações no heap de vizinhos.

No pior caso, as boxes não permitem poda e a consulta examina todas as instâncias:

\[
O(Nd).
\]

A DCT não deve ser descrita como tendo busca garantida em \(O(\log N)\). A efetividade do *branch-and-bound* depende da distribuição, da dimensionalidade, da métrica, da capacidade e da representação dos dados.

## 19. Relação com a KD-tree

As duas estruturas compartilham:

- vetores em um espaço de dimensões;
- cortes por eixos;
- roteamento por comparações coordenadas;
- possibilidade de poda espacial.

A diferença conceitual principal é a unidade de adaptação.

Uma KD-tree tradicional costuma associar uma instância a cada nó e pode depender de uma construção balanceada ou de reconstruções para recuperar qualidade após muitas atualizações.

Na DCT:

- os elementos permanecem nas folhas;
- cada folha possui uma EI;
- a carga da folha controla sua granularidade;
- o split substitui somente a folha sobrecarregada;
- o merge recompõe regiões localmente;
- a estrutura acompanha o fluxo sem exigir reconstrução global como operação normal.

Portanto, a DCT não é apenas uma KD-tree com folhas maiores. Sua característica definidora é a combinação entre partição dimensional e ciclo local de carga, split e merge.

## 20. O que pertence ao núcleo

Pertencem à DCT básica descrita aqui:

- vetores em \(\mathbb{R}^d\);
- função de carga \(\phi(S_v)\);
- capacidade \(C\);
- Estruturas Internas nas folhas;
- splits locais por eixos;
- regiões estáveis de roteamento;
- armazenamento das instâncias nas folhas.

São mecanismos opcionais ou variantes:

- merge e limiar \(Y\);
- carga por todas as instâncias ou apenas por vetores diferentes;
- boxes vivas e Box Adjust;
- busca kNN e métrica concreta;
- mapa de referências para remoção;
- escolha adaptativa do eixo;
- splits com \(SN>2\);
- EIs especializadas;
- representações ou conversões dos atributos.

## 21. Definição curta

> Uma DCT ou KDCT é uma Charge Tree para vetores multidimensionais. Cada folha armazena suas instâncias em uma Estrutura Interna e mede sua carga por meio de uma função \(\phi(S_v)\). Contar apenas vetores diferentes é uma configuração considerável quando existem muitas duplicatas, mas não é uma exigência da estrutura. Ao atingir a capacidade, a folha é dividida localmente por cortes em eixos, produzindo regiões estáveis para o roteamento futuro. Remoções podem provocar merges, enquanto boxes associadas às regiões permitem consultas espaciais exatas por *branch-and-bound*. Na DCT com Box Adjust, o roteamento permanece estável, mas as boxes de consulta acompanham apenas os elementos vivos, fornecendo limites de poda mais apertados.
