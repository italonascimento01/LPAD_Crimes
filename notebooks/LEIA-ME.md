# LAPD Crimes — cadeia de três notebooks

    01_limpeza.ipynb  →  data/processed/lapd_clean.csv
    02_clusterizacao.ipynb  →  data/processed/amostra_clusters.csv (+ metricas_escolha_k.csv, associacao_clusters.csv)
    03_visualizacoes.ipynb  →  docs/figuras/*.png

Rode na ordem 01 → 02 → 03, sempre **a partir da mesma pasta de trabalho** (as pastas `data/raw`, `data/processed` e `docs/figuras` são criadas automaticamente).
No Google Colab, use a mesma sessão (ou monte o Drive) para que os CSVs sobrevivam entre notebooks.

- **01** baixa a base bruta (uma vez), aplica corte em 29/02/2024, remove duplicatas, trata sentinelas e transforma ausências estruturais em categorias (nenhuma linha é descartada por isso).
- **02** lê `lapd_clean.csv` e roda o K-Modes (atributos do fato criminal) e o agrupamento de modus operandi. Exporta também `LAT` e `LON`.
- **03** só lê os CSVs do 02 e produz as figuras. Se `LAT`/`LON` não estiverem no CSV, pula o mapa.
