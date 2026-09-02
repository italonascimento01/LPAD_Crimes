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
├── notebooks/            # análises em Jupyter
├── src/                  # funções reutilizáveis
├── requirements.txt
└── README.md
```

## Entregas

| Data | Documento | Etapa |
|---|---|---|
| 02/09/2026 | [Relatório Técnico Parcial](docs/relatorios/2026-09-02_relatorio-parcial.pdf) | Exploração, qualidade e primeiros experimentos de agrupamento |

## Estado atual

Concluídas a caracterização da base, a análise de qualidade e dois experimentos de agrupamento com K-Modes.

Principais achados até o momento:

- **Quebra estrutural em março de 2024.** O volume mensal cai de ~19.000 para ~8.000 registros em decorrência da migração do LAPD para sistema alinhado ao NIBRS. Não representa redução de criminalidade. Análises longitudinais devem ser truncadas em 29/02/2024.
- **Os experimentos de K-Modes não produziram os perfis pretendidos.** O diagnóstico por informação mútua mostrou que as partições refletiam variáveis demográficas da vítima, e não perfil criminal. A curva de custo não apresenta ponto de inflexão.
- **Resultados preliminares por métodos alternativos** (V de Cramér e análise de coocorrência) já respondem parcialmente a PP1 e PP2, em ambos os casos contrariando as hipóteses iniciais.

O relatório parcial documenta cinco falhas metodológicas identificadas (E1 a E5) e o respectivo plano de correção.

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
