# Dados

Os arquivos de dados **não são versionados** neste repositório (ver `.gitignore`).
O CSV bruto tem aproximadamente 255 MB, acima do limite de 100 MB do GitHub.

## Como obter a base

Baixe o dataset e salve em `data/raw/`:

- **Fonte:** Los Angeles Open Data
- **Dataset:** `2nrs-mtv8` — *Crime Data from 2020 to Present*
- **URL:** https://data.lacity.org/Public-Safety/Crime-Data-from-2020-to-Present/2nrs-mtv8
- **Download direto:** https://data.lacity.org/api/views/2nrs-mtv8/rows.csv?accessType=DOWNLOAD
- **Nome esperado:** `data/raw/Crime_Data_from_2020_to_Present.csv`

Via linha de comando:

```bash
curl -L "https://data.lacity.org/api/views/2nrs-mtv8/rows.csv?accessType=DOWNLOAD" \
     -o data/raw/Crime_Data_from_2020_to_Present.csv
```

## Estrutura

| Diretório | Conteúdo |
|---|---|
| `raw/` | CSV original, sem modificação |
| `processed/` | Bases tratadas geradas pelos notebooks |

## Observação importante

A série apresenta quebra estrutural em março de 2024 (migração do LAPD para
sistema NIBRS). Análises longitudinais devem ser truncadas em 29/02/2024.
Ver Seção 3 do relatório em `docs/relatorios/`.
