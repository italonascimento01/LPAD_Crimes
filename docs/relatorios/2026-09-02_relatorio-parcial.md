# LPAD Crimes — Relatório Técnico Parcial

**Projeto Integrador III — Ciência de Dados e Machine Learning**
Centro Universitário de Brasília (CEUB)

**Integrantes:** Henrique Valle; Matheus Lopes; Italo Nascimento; Davi Amâncio
**Data:** setembro de 2026
**Etapa:** entrega parcial — análise exploratória e primeiros experimentos de agrupamento

---

## Sumário executivo

Este relatório documenta o estado atual do projeto LPAD Crimes, que investiga padrões de associação em registros de criminalidade do Departamento de Polícia de Los Angeles.

Até o momento foram concluídas a caracterização da base, a análise de qualidade dos dados e dois experimentos de agrupamento com o algoritmo K-Modes. A análise exploratória identificou uma **quebra estrutural na série histórica em março de 2024**, decorrente de migração do sistema de registro do LAPD, que inviabiliza comparações longitudinais cobrindo o período completo.

Os experimentos de agrupamento **não produziram os perfis pretendidos**. O diagnóstico quantitativo mostrou que as partições geradas refletiam predominantemente as variáveis demográficas da vítima, e não o perfil criminal, e que a curva de custo não apresenta ponto de inflexão — indicando ausência de estrutura de agrupamento natural no espaço de atributos utilizado.

Este relatório apresenta esse diagnóstico de forma explícita, identifica cinco falhas metodológicas nos experimentos realizados e estabelece o plano de correção para a próxima etapa. Em paralelo, resultados preliminares obtidos por métodos alternativos (coeficiente V de Cramér e análise de coocorrência) já respondem parcialmente às perguntas de pesquisa e são apresentados na Seção 7.

---

## 1. Contextualização e perguntas de pesquisa

### 1.1 Objeto

O projeto analisa registros de ocorrências criminais da cidade de Los Angeles, Califórnia, disponibilizados pelo Departamento de Polícia local (LAPD) em portal de dados abertos. A escolha se justifica pelo volume (mais de um milhão de registros), pela cobertura temporal contínua e pela natureza predominantemente categórica dos atributos, que impõe restrições metodológicas interessantes do ponto de vista de aprendizado não supervisionado.

### 1.2 Perguntas de pesquisa

O trabalho está organizado em torno de duas perguntas:

**PP1 — Existem tipos de crime que ocorrem associados entre si?**
Ou seja, quando um boletim registra múltiplas tipificações, quais combinações são recorrentes? A hipótese inicial era de que haveria pares estáveis (por exemplo, arrombamento e furto).

**PP2 — Existem associações entre tipo de crime, área geográfica e características demográficas das vítimas?**
Interessa aqui verificar se determinadas divisões policiais concentram tipos específicos de crime, e se o perfil demográfico das vítimas varia sistematicamente entre tipos de crime ou entre áreas.

### 1.3 Delimitação

O trabalho é descritivo e exploratório. Não se propõe a prever ocorrências futuras nem a inferir causalidade. Uma ressalva de escopo, tratada em detalhe na Seção 8, é que os dados registram **ocorrências notificadas e registradas pela polícia**, e não criminalidade efetiva.

---

## 2. Base de dados

### 2.1 Identificação

| Item | Descrição |
|---|---|
| Fonte | Los Angeles Open Data — `data.lacity.org` |
| Identificador | `2nrs-mtv8` ("Crime Data from 2020 to Present") |
| Registros | 1.004.894 |
| Colunas | 28 |
| Período (`DATE OCC`) | 01/01/2020 a 30/12/2024 |
| Volume | ~255 MB |
| Unidade de análise | Boletim de ocorrência |

> **Nota sobre a documentação consultada.** O dicionário de dados que a equipe utilizou como referência (`63jg-8b9z`, "Crime Data from 2010 to 2019") corresponde ao dataset irmão, de período anterior. O esquema de colunas e as tabelas de código (Crm Cd, Vict Descent, MO Codes) são compartilhados entre os dois conjuntos, o que torna a consulta válida. Na versão final, a referência bibliográfica deve citar `2nrs-mtv8`, que é o conjunto efetivamente analisado.

### 2.2 Unidade de análise

Cada linha corresponde a **um boletim de ocorrência**, não a uma vítima nem a uma ofensa individual. A distinção é relevante: um boletim pode registrar até quatro tipificações criminais (`Crm Cd 1` a `Crm Cd 4`), e o dataset traz no máximo um conjunto de atributos de vítima por registro. Contagens de "crimes" derivadas de contagem de linhas são, portanto, contagens de boletins.

### 2.3 Estrutura das colunas

| Grupo | Colunas |
|---|---|
| Identificação | `DR_NO` |
| Temporal | `Date Rptd`, `DATE OCC`, `TIME OCC` |
| Geográfico | `AREA`, `AREA NAME`, `Rpt Dist No`, `LOCATION`, `Cross Street`, `LAT`, `LON` |
| Tipificação | `Part 1-2`, `Crm Cd`, `Crm Cd Desc`, `Crm Cd 1` a `Crm Cd 4`, `Mocodes` |
| Vítima | `Vict Age`, `Vict Sex`, `Vict Descent` |
| Contexto | `Premis Cd`, `Premis Desc`, `Weapon Used Cd`, `Weapon Desc` |
| Desfecho | `Status`, `Status Desc` |

A chave primária `DR_NO` não apresenta duplicatas.

### 2.4 Caracterização descritiva

**Tipificações mais frequentes:**

| Tipificação | Ocorrências |
|---|---|
| VEHICLE - STOLEN | 115.184 |
| BATTERY - SIMPLE ASSAULT | 74.821 |
| BURGLARY FROM VEHICLE | 63.515 |
| THEFT OF IDENTITY | 62.536 |
| VANDALISM - FELONY | 61.086 |
| BURGLARY | 57.871 |

**Distribuição geográfica:** as 21 divisões policiais variam de 33.129 ocorrências (Foothill) a 69.668 (Central). Central é seguida por 77th Street (61.756) e Pacific (59.513). Estas são contagens absolutas — na ausência de dados populacionais por divisão, não constituem taxas, e Central em particular é distorcida pela população flutuante da região central da cidade.

**Desfecho das ocorrências:**

| Status | Registros | % |
|---|---|---|
| Invest Cont (investigação em curso) | 802.772 | 79,9% |
| Adult Other | 109.799 | 10,9% |
| Adult Arrest | 87.152 | 8,7% |
| Juv Arrest | 3.285 | 0,3% |
| Juv Other | 1.879 | 0,2% |

A taxa agregada de prisão (adulto + juvenil) é de 9,0%.

---

## 3. Achado crítico: quebra estrutural na série temporal

Este é o achado mais importante da fase exploratória e condiciona todas as análises subsequentes.

### 3.1 Descrição

A contagem mensal de ocorrências mantém-se estável entre 19.000 e 20.000 de janeiro de 2020 a fevereiro de 2024. A partir de março de 2024, cai abruptamente para a faixa de 8.000, permanecendo nesse patamar até o fim da série.

**Registros por ano:**

| Ano | Registros |
|---|---|
| 2020 | 199.847 |
| 2021 | 209.876 |
| 2022 | 235.259 |
| 2023 | 232.345 |
| **2024** | **127.567** |

### 3.2 Interpretação

A queda não representa redução de criminalidade. Em março de 2024 o LAPD implantou novo sistema de gestão de registros, alinhado ao padrão NIBRS (*National Incident-Based Reporting System*). A partir dessa data, apenas ocorrências ainda processadas no sistema legado continuaram alimentando este conjunto de dados; os registros do novo sistema passaram a ser publicados em datasets NIBRS separados, com esquema distinto.

### 3.3 Evidências de suporte

O efeito não é uniforme entre tipos de crime, o que confirma tratar-se de viés de seleção e não de mudança no fenômeno:

- `VEHICLE - STOLEN` passa de 10,5% para 16,9% do total anual em 2024;
- `ASSAULT WITH DEADLY WEAPON` cai de 5,5% para 2,9%;
- A taxa de prisão cai de 8,8% (2023) para 6,1% (2024), acompanhando a mesma descontinuidade — enquanto a série 2020–2023 apresentava declínio suave e monotônico (10,7% → 9,7% → 8,7% → 8,8%).

### 3.4 Decisão metodológica

**Toda análise longitudinal do projeto será truncada em 29/02/2024.** Análises que não dependam de comparação temporal podem usar a série completa, desde que a limitação seja declarada.

A extensão da cobertura para 2024–2025 exigiria incorporar os datasets NIBRS, cuja unidade de análise é a ofensa (não o boletim) e que não trazem localização em nível de quadra. Essa integração fica fora do escopo desta etapa.

---

## 4. Qualidade dos dados

### 4.1 Valores ausentes

| Coluna | Ausentes | % | Natureza |
|---|---|---|---|
| `Crm Cd 4` | 1.004.830 | 99,99% | Estrutural |
| `Crm Cd 3` | 1.002.580 | 99,77% | Estrutural |
| `Crm Cd 2` | 935.740 | 93,12% | Estrutural |
| `Cross Street` | 850.666 | 84,65% | Estrutural |
| `Weapon Used Cd` / `Weapon Desc` | 677.678 | 67,44% | Estrutural |
| `Mocodes` | 151.598 | 15,09% | Misto |
| `Vict Sex` | 144.631 | 14,39% | Estrutural |
| `Vict Descent` | 144.643 | 14,39% | Estrutural |
| `Premis Desc` | 588 | 0,06% | Aleatório |

A maioria dos ausentes é **estrutural**: a ausência carrega significado. `Weapon Desc` é vazio em crimes sem arma (furto, fraude, vandalismo); `Cross Street` só é preenchido quando o local é um cruzamento; `Vict Sex` e `Vict Descent` são vazios quando não há vítima pessoa física identificada. Tratá-los como dados faltantes a serem imputados seria um erro conceitual.

### 4.2 Valores sentinela

Três colunas usam códigos que aparentam ser valores válidos mas representam ausência ou não-aplicabilidade:

**`Vict Age`** — 26,8% dos registros têm valor 0, que codifica "não aplicável ou desconhecido", não idade zero. O efeito sobre estatísticas descritivas é substancial: a média bruta é 28,9 anos, contra 39,5 anos considerando apenas idades positivas. Há ainda 137 registros com idade negativa (mínimo −4) e um máximo de 120 anos.

**`LAT` / `LON`** — 2.240 registros com coordenada (0, 0), padrão da documentação do LAPD para localização não informada. Sem filtro prévio, produzem um agrupamento espúrio no Golfo da Guiné em qualquer análise espacial.

**`Vict Sex` e `Vict Descent`** — o código `X` significa "desconhecido" e aparece em 97.753 e 106.661 registros respectivamente. Somados aos nulos, isso significa que cerca de 24% dos registros não têm informação demográfica utilizável. Há ainda resíduos de digitação: 114 registros com `H` em `Vict Sex` e um com `-`.

### 4.3 Vieses de mensuração

**Arredondamento de horário.** 39,1% dos valores de `TIME OCC` terminam em `:00`, com pico em 1200 (35.198 registros). O horário é frequentemente estimado pela vítima quando o crime não foi presenciado. Análises horárias de granularidade fina não são confiáveis; recomenda-se agregação em faixas do dia.

**Defasagem de notificação.** A mediana entre `DATE OCC` e `Date Rptd` é de 1 dia, mas 5,9% dos registros apresentam defasagem superior a 30 dias, com máximo de 1.862 dias. Isso implica censura à direita: os meses mais recentes de qualquer recorte estão subcontados.

**Anonimização geográfica.** Os endereços em `LOCATION` são truncados na centena da quadra por política de privacidade. O conjunto apresenta apenas 77.869 pares de coordenadas distintos para mais de um milhão de registros — os pontos são centroides de quadra, não localizações exatas. Isso limita a granularidade de qualquer análise de concentração espacial.

**Divergência interna.** `Crm Cd` difere de `Crm Cd 1` em 1.977 registros. A relação entre os dois campos precisa ser padronizada.

### 4.4 Campo subutilizado: `Mocodes`

O campo `Mocodes` registra códigos de modus operandi, com múltiplos valores por registro separados por espaço (exemplo: `"0416 0334 2004 1822 1414"`). Há 739 tokens distintos no vocabulário, mas 310.912 combinações distintas de tokens. É o campo com maior densidade de informação comportamental do conjunto, e o tratamento adequado (binarização em indicadores por token) é uma das prioridades da próxima etapa.

---

## 5. Metodologia dos experimentos de agrupamento

### 5.1 Justificativa do algoritmo

A quase totalidade dos atributos relevantes é categórica nominal (tipo de crime, área, sexo, ascendência, modus operandi). O K-Means é inaplicável: exige espaço métrico e o cálculo de médias, que não têm significado sobre categorias nominais. A codificação *one-hot* seguida de K-Means introduz distorções conhecidas na estrutura de distâncias.

O **K-Modes** (Huang, 1998) é a adaptação natural: substitui a distância euclidiana pela **dissimilaridade de Hamming** (contagem de atributos discordantes) e o centroide pela **moda** de cada atributo dentro do agrupamento. O custo total é a soma das discordâncias de cada registro em relação à moda de seu agrupamento.

### 5.2 Precisão conceitual sobre o método

Um esclarecimento necessário, pois a formulação inicial do projeto descrevia o K-Modes como um algoritmo de associação.

O K-Modes é um algoritmo **particional**, não associativo. Ele responde à pergunta "quais *registros* se assemelham entre si?", atribuindo cada observação ao centroide mais próximo. Ele não mede coocorrência entre categorias nem gera regras do tipo "se A então B".

Para PP1 (coocorrência entre tipificações), a família correta é a de **regras de associação** (Apriori, FP-Growth). Para PP2 (associação entre variáveis categóricas), a ferramenta é o **teste qui-quadrado de independência**, acompanhado de uma medida de força de associação como o **V de Cramér**.

Essa correção conceitual é um dos principais aprendizados desta etapa e orienta o redesenho descrito na Seção 9.

### 5.3 Experimentos realizados

| | Experimento 1 | Experimento 2 |
|---|---|---|
| Atributos | `AREA`, `Crm Cd`, `Mocodes`, `Vict Sex`, `Vict Descent` | os 5 anteriores + `Crm Cd 1` a `Crm Cd 4` |
| k | 3 (definido a priori) | 7 (via método do cotovelo) |
| Inicialização | Huang | Huang |
| Amostra | 10.000 registros | 10.000 registros |
| Pré-processamento | `fillna(0)` em `Crm Cd 1–4`; `dropna` em `Mocodes`, `Vict Sex`, `Vict Descent` | idem |

### 5.4 Instrumento de diagnóstico

Para avaliar o que cada partição efetivamente capturou, adotou-se a **Informação Mútua Ajustada (AMI)** entre o rótulo de agrupamento e cada atributo de entrada. A AMI mede quanto do rótulo é explicado por uma variável, corrigida para concordância esperada ao acaso. Valores próximos de zero indicam que o atributo não participou da formação dos agrupamentos.

Esta métrica não constava do plano original e foi incorporada durante a análise crítica dos resultados.

---

## 6. Resultados dos experimentos e diagnóstico

### 6.1 Experimento 1 — k = 3

**Composição das partições:**

| Agrupamento | n | Perfil dominante |
|---|---|---|
| 0 | 4.773 | 48,5% W; 60,0% M |
| 1 | 3.257 | 89,0% H; 57,9% M |
| 2 | 1.970 | 48,0% B; 84,3% F |

**Diagnóstico por AMI:**

| Atributo | AMI |
|---|---|
| `Vict Descent` | 0,326 |
| `Vict Sex` | 0,134 |
| `AREA` | 0,133 |
| `Crm Cd` | 0,065 |
| `Mocodes` | 0,033 |

**Interpretação.** A partição foi determinada essencialmente por ascendência e sexo da vítima. Tipo de crime e modus operandi — as variáveis de interesse do projeto — praticamente não participaram. A distribuição de `AREA` dentro de cada agrupamento é indistinguível da distribuição global (todas as áreas entre 0,2% e 0,8% em qualquer agrupamento).

O resultado é circular: as variáveis demográficas foram fornecidas como entrada e o algoritmo devolveu grupos demográficos. Isso não responde nem PP1 nem PP2. O que foi obtido equivale a uma tabela de contingência `Vict Descent × Vict Sex`, produzida por um caminho computacionalmente mais custoso.

### 6.2 Experimento 2 — k = 7

O segundo experimento incorporou o método do cotovelo para seleção de k (testando k de 1 a 50, com `init='Cao'` e `init='Huang'`) e ampliou o conjunto de atributos com `Crm Cd 1` a `Crm Cd 4`.

**Cardinalidade dos atributos utilizados:**

| Atributo | Níveis | Moda cobre | AMI (k=7) |
|---|---|---|---|
| `Mocodes` | 5.717 | 6,3% | 0,059 |
| `Crm Cd` | 108 | 8,8% | 0,275 |
| `Crm Cd 1` | 107 | 8,8% | 0,276 |
| `AREA` | 21 | 13,4% | 0,149 |
| `Vict Descent` | 14 | 37,3% | 0,209 |
| `Vict Sex` | 3 | 49,4% | 0,282 |
| `Crm Cd 2` | 45 | 90,3% | 0,007 |
| `Crm Cd 3` | 5 | 99,6% | 0,000 |
| `Crm Cd 4` | 2 | 100,0% | −0,000 |

**Avanço.** A AMI de `Crm Cd` subiu de 0,065 (k=3) para 0,275. Com maior número de agrupamentos, o tipo de crime passa a participar da partição de forma relevante. Este é um progresso real em relação ao Experimento 1.

**Curva de custo.** O custo cai de forma contínua e sem inflexão, de 38.487 (k=1) a 20.654 (k=50) — redução de 46% distribuída ao longo de 50 agrupamentos. **Não há cotovelo.** Nenhum valor de k se distingue dos vizinhos por critério de custo.

Este é, em si, um resultado interpretável: a ausência de ponto de inflexão indica que os dados **não apresentam estrutura de agrupamento natural** no espaço de atributos utilizado. A escolha de k=7 não é sustentada pela curva.

**Instabilidade numérica.** Com `init='Huang'` e `n_init=5`, o custo **aumenta** em 15 dos 49 incrementos de k (por exemplo, k=6 apresenta custo superior a k=5; k=11 superior a k=10; k=14 superior a k=13). Em uma solução bem convergida, o custo deve ser não-crescente em k. As subidas indicam convergência para ótimos locais, com número de reinicializações insuficiente.

**Agrupamento espúrio.** Um dos sete agrupamentos é composto majoritariamente por registros com valor `X` tanto em `Vict Sex` quanto em `Vict Descent` — ou seja, é um agrupamento de **ausência de informação demográfica**, não um perfil criminal. Não possui interpretação substantiva e não deve ser reportado como perfil.

---

## 7. Resultados preliminares por métodos alternativos

Enquanto o redesenho do agrupamento não é concluído, as perguntas de pesquisa já receberam respostas parciais pelas ferramentas metodologicamente adequadas.

### 7.1 PP1 — Coocorrência entre tipificações

Cruzamento de `Crm Cd 1` × `Crm Cd 2` nos 69.154 boletins com dupla tipificação:

| Ocorrências | % dos boletins com 2 tipificações | Par |
|---|---|---|
| 3.191 | 4,6% | BRANDISH WEAPON + CRIMINAL THREATS |
| 1.287 | 1,9% | CRM AGNST CHLD + BATTERY WITH SEXUAL CONTACT |
| 334 | 0,5% | RAPE, FORCIBLE + CRM AGNST CHLD |
| 252 | 0,4% | INTIMATE PARTNER - SIMPLE ASSAULT + BRANDISH WEAPON |
| 250 | 0,4% | CRM AGNST CHLD + SEXUAL PENETRATION W/FOREIGN OBJECT |
| 245 | 0,4% | THEFT PLAIN - PETTY + INTIMATE PARTNER - SIMPLE ASSAULT |

**Leitura preliminar.** As coocorrências concentram-se em dois eixos: violência interpessoal com escalada (ameaça acompanhada de exibição de arma) e crimes sexuais contra menores. Nenhum par patrimonial aparece entre os mais frequentes, o que **contraria a hipótese inicial** de que pares como arrombamento e furto seriam recorrentes.

Ressalva: a contagem bruta reflete frequência marginal dos tipos envolvidos. A formalização exige medidas de suporte, confiança e *lift*, a serem calculadas na próxima etapa via FP-Growth.

### 7.2 PP2 — Associação entre variáveis categóricas

Coeficiente V de Cramér, calculado sobre a base completa (sem filtro de categorias) e com correção de viés:

| Par de variáveis | Dimensão | V | n |
|---|---|---|---|
| `Crm Cd Desc` × `Vict Sex` | 139 × 5 | **0,286** | 860.246 |
| `Crm Cd Desc` × `Weapon Desc` | 122 × 79 | 0,202 | 327.131 |
| `Crm Cd Desc` × `Vict Descent` | 139 × 20 | 0,144 | 860.246 |
| `Crm Cd Desc` × `Premis Desc` | 139 × 306 | 0,131 | 859.674 |
| `AREA NAME` × `Vict Descent` | 21 × 20 | 0,125 | 860.246 |
| `AREA NAME` × `Vict Sex` | 21 × 5 | 0,102 | 860.246 |
| `Crm Cd Desc` × `AREA NAME` | 139 × 21 | **0,090** | 860.246 |

Todos os testes qui-quadrado retornaram p < 0,001. Com amostras desta magnitude, contudo, o p-valor é praticamente não informativo — qualquer desvio mínimo da independência é estatisticamente significante. **A interpretação deve se apoiar no tamanho do efeito (V), não na significância.**

**Leitura preliminar.** A associação mais forte é entre tipo de crime e sexo da vítima (0,286), o que é consistente com a literatura sobre distribuição por gênero de crimes como violência doméstica e roubo. A associação mais fraca é entre tipo de crime e área geográfica (0,090): **as divisões policiais de Los Angeles não se especializam em tipos de crime**, o que contraria a intuição de "cada bairro com seu crime característico".

Ressalva metodológica: o V de Cramér é normalizado por `min(r−1, k−1)` e portanto não é diretamente comparável entre tabelas de dimensões muito distintas. A ordenação acima deve ser lida como indicativa. Uma versão robusta da comparação está prevista para a próxima etapa.

---

## 8. Erros identificados e plano de correção

Esta seção registra explicitamente as falhas detectadas na revisão crítica dos experimentos. Todas serão corrigidas na próxima entrega.

### E1 — Amostragem não aleatória

**O que ocorreu.** A amostra de trabalho foi obtida por `df_cluster[:10000]`, que seleciona as 10.000 primeiras linhas do arquivo, não uma amostra aleatória.

**Impacto.** As 10.000 primeiras linhas são integralmente de 2020, e a distribuição por divisão policial difere substancialmente da global (as áreas 14 e 18 predominam no recorte; a área 1 predomina no conjunto completo). Os experimentos foram conduzidos sobre um recorte enviesado.

**Correção.** Substituir por `df.sample(n=50000, random_state=42)`. O algoritmo suporta volumes consideravelmente maiores que 10.000 registros.

### E2 — Exclusão desbalanceada de registros

**O que ocorreu.** A remoção de nulos em `Mocodes`, `Vict Sex` e `Vict Descent` elimina 15,1% dos registros, mas não de forma uniforme entre tipos de crime.

**Impacto.** `VEHICLE - STOLEN` perde 99,5% dos registros (de 115.184 para 546); `THEFT FROM MOTOR VEHICLE - PETTY` perde 59,0%. Os demais tipos do top 12 perdem menos de 4%. O crime mais frequente do conjunto (11,5% do total) foi efetivamente removido da análise, porque furto de veículo não possui vítima pessoa física com sexo e ascendência registrados.

**Correção.** Substituir a exclusão por categoria explícita `"NAO_APLICAVEL"`, ou declarar formalmente o escopo como restrito a crimes com vítima pessoa física identificada. A primeira alternativa é preferível.

### E3 — Cardinalidade desbalanceada entre atributos

**O que ocorreu.** `Mocodes` entrou no modelo como cadeia de caracteres única, gerando 5.717 níveis distintos na amostra, contra 3 níveis de `Vict Sex`.

**Impacto.** Na dissimilaridade de Hamming, um atributo com 5.717 níveis discorda em aproximadamente 94% dos pares de registros, tornando-se praticamente constante na função de custo. A AMI de `Mocodes` foi de 0,033 (Exp. 1) e 0,059 (Exp. 2): o campo com maior densidade informacional do conjunto foi o que menos contribuiu.

**Correção.** Binarizar `Mocodes` em 739 indicadores por token (`str.split()`), ou removê-lo do K-Modes e tratá-lo por método adequado a dados em formato de cesta.

### E4 — Atributos redundantes e quase-constantes

**O que ocorreu.** A inclusão de `Crm Cd 1` a `Crm Cd 4` no Experimento 2 adicionou uma variável redundante e três praticamente constantes.

**Impacto.**
- `Crm Cd 1` é idêntico a `Crm Cd` em 99,7% dos registros. Sua inclusão equivale a atribuir peso 2 ao tipo de crime — uma ponderação implícita e não intencional.
- `Crm Cd 3` tem moda cobrindo 99,6% dos registros e AMI de 0,000; `Crm Cd 4` tem moda cobrindo 100,0% e contribui com 2 discordâncias em um custo total de 48.613. Atributos constantes adicionam apenas um deslocamento fixo ao custo e diluem o peso relativo dos atributos informativos.

**Correção.** Manter apenas `Crm Cd`; remover `Crm Cd 1`, `Crm Cd 3` e `Crm Cd 4`. `Crm Cd 2` deve migrar para a análise de coocorrência (PP1), onde tem função legítima, e sair do K-Modes.

### E5 — Saídas obsoletas no notebook

**O que ocorreu.** As curvas de custo apresentadas foram geradas em sessão anterior à alteração do conjunto de atributos.

**Evidência.** O custo reportado em k=1 é 38.487. A decomposição desse custo por atributo — `AREA` 8.665 + `Crm Cd` 9.123 + `Mocodes` 9.371 + `Vict Sex` 5.061 + `Vict Descent` 6.267 — soma exatamente 38.487, correspondendo à configuração de **5 atributos**. Com os 9 atributos efetivamente utilizados no Experimento 2, o custo em k=1 seria 48.613. Adicionalmente, os avisos de execução de células distintas referenciam identificadores de kernel diferentes, confirmando que parte das saídas foi herdada da sessão anterior.

**Impacto.** A curva que fundamentou a escolha de k=7 descreve um espaço de atributos diferente daquele efetivamente utilizado. A escolha de k não está, portanto, sustentada pela evidência apresentada.

**Correção.** Reexecução completa em sessão limpa (*Restart and run all*), com `n_init` elevado para 20 ou mais, e regeração de todas as figuras a partir da configuração final de atributos.

### Síntese

| Código | Falha | Severidade | Status |
|---|---|---|---|
| E1 | Amostragem não aleatória | Alta | A corrigir |
| E2 | Exclusão desbalanceada de registros | Alta | A corrigir |
| E3 | Cardinalidade desbalanceada | Alta | A corrigir |
| E4 | Atributos redundantes/constantes | Média | A corrigir |
| E5 | Saídas obsoletas | Alta | A corrigir |

---

## 9. Limitações

### 9.1 Natureza do dado

Registros policiais medem **criminalidade notificada e registrada**, não criminalidade ocorrida. Três mecanismos afastam um do outro:

- **Subnotificação diferencial.** Nem toda vítima procura a polícia, e a propensão a notificar varia por tipo de crime, por perfil da vítima e por relação da comunidade com a instituição policial.
- **Viés de policiamento.** A distribuição espacial dos registros reflete, em parte, a distribuição do efetivo policial. Áreas mais patrulhadas geram mais registros para o mesmo nível de criminalidade subjacente.
- **Efeito de classificação.** A tipificação depende de decisão do agente que registra a ocorrência.

Toda conclusão do trabalho deve ser enunciada em termos de registros, não de criminalidade.

### 9.2 Uso de variáveis demográficas

O projeto utiliza `Vict Descent` e `Vict Sex` como variáveis de análise. Três esclarecimentos são necessários:

1. **As variáveis descrevem vítimas, não suspeitos.** O conjunto de dados não contém informação sobre autores de crimes. Nenhuma inferência sobre perfil de autoria é possível a partir dele.
2. **Não constituem variáveis explicativas de comportamento.** A distribuição demográfica observada reflete a composição demográfica dos territórios, combinada com os vieses de notificação e policiamento descritos em 9.1.
3. **Agrupamentos organizados por essas variáveis não são perfis criminais.** Quando o Experimento 1 produziu partições determinadas por ascendência, o resultado descreve a composição da base, não uma característica do crime. Reportá-lo como perfil seria incorreto do ponto de vista técnico e problemático do ponto de vista ético.

### 9.3 Limitações de escopo

- Cobertura restrita a 2020–fevereiro de 2024 para análises longitudinais (Seção 3).
- Ausência de denominador populacional por divisão policial impede o cálculo de taxas.
- Coordenadas anonimizadas em nível de quadra limitam a granularidade espacial.
- Ausência de dados socioeconômicos por território impede controle por variáveis de confusão.

---

## 10. Próximos passos

### Etapa imediata — correção

1. Implementar as correções E1 a E5 (Seção 8).
2. Regerar a curva de custo com o conjunto final de atributos, `n_init ≥ 20`, e comparar `init='Cao'` e `init='Huang'`.
3. Reavaliar por AMI se a partição corrigida captura tipo de crime e modus operandi. Se a curva permanecer sem cotovelo — hipótese mais provável —, registrar formalmente a ausência de estrutura latente como resultado do trabalho.

### Etapa seguinte — métodos adequados às perguntas

4. **PP1:** aplicar FP-Growth sobre o conjunto `{Crm Cd 1, ..., Crm Cd 4}` como transação, reportando suporte, confiança e *lift*.
5. **PP2:** consolidar a análise de V de Cramér com tratamento adequado da comparabilidade entre tabelas de dimensões distintas; complementar com análise de resíduos padronizados para identificar quais células específicas concentram a associação.
6. **`Mocodes`:** binarizar em 739 indicadores e explorar agrupamento sobre esse espaço, que é o mais promissor do conjunto para caracterização de modus operandi.

### Extensão possível

7. Formular um problema supervisionado de classificação do desfecho (`Status Desc`: prisão *vs.* investigação em curso), com desbalanceamento de aproximadamente 9/91. A avaliação deve usar precisão-revocação, não acurácia.

---

## 11. Considerações finais

A entrega desta etapa não apresenta os perfis de crime que constavam do plano inicial. O que apresenta é um diagnóstico quantificado de por que o método escolhido não os produziu, e a identificação do caminho metodologicamente correto para as perguntas de pesquisa.

Três resultados consolidados sustentam a etapa:

1. A identificação da quebra estrutural de março de 2024, que invalidaria qualquer análise longitudinal do período completo e que não é sinalizada na documentação da fonte.
2. O diagnóstico, por informação mútua, de que as partições geradas refletiam composição demográfica e não perfil criminal — acompanhado da identificação das cinco causas técnicas.
3. Os resultados preliminares por V de Cramér e coocorrência, que já respondem parcialmente às duas perguntas de pesquisa e, em ambos os casos, contrariam as hipóteses iniciais.

---

## Referências

Los Angeles Police Department. *Crime Data from 2020 to Present*. Los Angeles Open Data. Disponível em: https://data.lacity.org/Public-Safety/Crime-Data-from-2020-to-Present/2nrs-mtv8

HUANG, Z. Extensions to the k-Means Algorithm for Clustering Large Data Sets with Categorical Values. *Data Mining and Knowledge Discovery*, v. 2, n. 3, p. 283–304, 1998.

CAO, F.; LIANG, J.; BAI, L. A new initialization method for categorical data clustering. *Expert Systems with Applications*, v. 36, n. 7, p. 10223–10228, 2009.

CRAMÉR, H. *Mathematical Methods of Statistics*. Princeton University Press, 1946.

VINH, N. X.; EPPS, J.; BAILEY, J. Information Theoretic Measures for Clusterings Comparison. *Journal of Machine Learning Research*, v. 11, p. 2837–2854, 2010.

HAN, J.; PEI, J.; YIN, Y. Mining frequent patterns without candidate generation. *ACM SIGMOD Record*, v. 29, n. 2, p. 1–12, 2000.

---

## Apêndice A — Ambiente e reprodutibilidade

| Item | Especificação |
|---|---|
| Linguagem | Python 3 |
| Bibliotecas | pandas, numpy, scipy, scikit-learn, kmodes 0.12.2, matplotlib, seaborn |
| Semente aleatória | 42 |
| Ambiente | Google Colab |

## Apêndice B — Decisões de tratamento

| Decisão | Justificativa | Seção |
|---|---|---|
| Truncar série em 29/02/2024 | Quebra estrutural por migração de sistema | 3.4 |
| `Vict Age = 0` como ausente | Sentinela, não idade real | 4.2 |
| Excluir `LAT`/`LON` = (0,0) | Sentinela de localização não informada | 4.2 |
| Tratar `X` como ausente | Código de "desconhecido" | 4.2 |
| Agregar `TIME OCC` em faixas | Arredondamento em 39,1% dos registros | 4.3 |
| Manter nulos estruturais | Ausência com significado; imputação seria incorreta | 4.1 |
