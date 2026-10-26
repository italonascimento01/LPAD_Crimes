# LPAD Crimes

Estudo exploratório da base de criminalidade do Departamento de Polícia de Los Angeles (LAPD).

**Projeto Integrador III** — Ciência de Dados e Machine Learning, CEUB.

**Integrantes:** Henrique Valle · Matheus Lopes · Italo Nascimento · Davi Amâncio

O objetivo do projeto é realizar um estudo da base de dados, apresentando ao final do semestre uma página web com o relatório do que foi desenvolvido. As questões relacionadas ao pipeline de dados são tratadas de forma a garantir um processamento profissional.

## Perguntas de pesquisa

1. **PP1** — Existem tipos de crime que ocorrem associados entre si?
2. **PP2** — Existem associações entre tipo de crime, área geográfica e características demográficas das vítimas?

## Base de dados

| Item | Valor |
|---|---|
| Fonte | [Los Angeles Open Data](https://data.lacity.org/Public-Safety/Crime-Data-from-2020-to-Present/2nrs-mtv8) |
| Identificador | `2nrs-mtv8` |
| Registros | 1.004.894 |
| Colunas | 28 |
| Período | 01/01/2020 a 30/12/2024 |

Os dados **não são versionados** neste repositório. Instruções de download em [`data/README.md`](data/README.md).

## Estrutura

```
LPAD_Crimes/
├── data/
│   ├── raw/              # CSV original (não versionado)
│   ├── processed/        # bases tratadas (não versionado)
│   └── README.md         # instruções de download
├── docs/
│   ├── relatorios/       # entregas do projeto
│   └── figuras/          # gráficos exportados
├── notebooks/            # cadeia de três notebooks (ver LEIA-ME.md)
├── src/                  # funções reutilizáveis
├── requirements.txt
└── README.md
```

## Entregas

| Data | Documento | Etapa |
|---|---|---|
| 26/10/2026 | [Relatório Técnico — Segunda Entrega](docs/relatorios/2026-10-26_relatorio-segunda-entrega.pdf) | Reestruturação do pipeline e validação formal do agrupamento |
| 02/09/2026 | [Relatório Técnico Parcial](docs/relatorios/2026-09-02_relatorio-parcial.pdf) | Exploração, qualidade e primeiros experimentos de agrupamento |

## Notebooks

A análise está organizada em três notebooks encadeados, com contratos explícitos entre etapas. Devem ser executados na ordem, a partir da mesma pasta de trabalho — ver [`notebooks/LEIA-ME.md`](notebooks/LEIA-ME.md).

```
01_limpeza.ipynb        → data/processed/lapd_clean.csv
02_clusterizacao.ipynb  → data/processed/amostra_clusters.csv
03_visualizacoes.ipynb  → docs/figuras/*.png
```

## Estado atual

Pipeline reconstruído em três notebooks. As cinco falhas apontadas no relatório parcial (E1 a E5) foram corrigidas, e a cadeia foi executada de ponta a ponta com as saídas preservadas.

Principais achados até o momento:

- **Quebra estrutural em março de 2024.** O volume mensal cai de ~19.000 para ~8.000 registros em decorrência da migração do LAPD para sistema alinhado ao NIBRS. Não representa redução de criminalidade. A base é truncada em 29/02/2024, restando 913.647 registros (90,9% do total).
- **Não há estrutura de agrupamento nos atributos do fato criminal.** O modelo nulo por permutação mostra que 87,5% da redução de custo obtida pelo K-Modes é reproduzida por dados embaralhados. A silhueta fica em 0,080 e nenhum valor de k atinge o limiar de estabilidade.
- **A partição obtida não é reproduzível.** Variando apenas a semente aleatória, o ARI médio entre execuções é de 0,226, com custo praticamente equivalente.
- **O agrupamento de modus operandi é o resultado mais promissor.** `Mocodes` tratado como texto (TF-IDF → SVD → K-Means) atinge silhueta de 0,40 e produz grupos coerentes.
- **O sinal mais forte da base está na taxa de prisão por tipo de crime**, que varia de 0,0% a 36,2% — amplitude que a partição não captura.

A segunda entrega documenta seis novas falhas (E6 a E11), uma delas já resolvida, e o plano de correção correspondente.

## Reprodução

```bash
git clone https://github.com/italonascimento01/LPAD_Crimes.git
cd LPAD_Crimes
pip install -r requirements.txt
# baixar a base conforme data/README.md
jupyter notebook
```

## Licença

MIT — ver [LICENSE](LICENSE).
