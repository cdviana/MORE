# Fluxo do pipeline

Este documento descreve a ordem de execução dos quatro notebooks, os artefatos
que cada um consome e produz, e as dependências entre eles — incluindo uma que
não é óbvia pela numeração.

## Visão geral

Os notebooks são **sequenciais** e devem ser executados na ordem numérica. Cada
um depende da saída do anterior.

```
BIM ─► casas_materiais.xlsx
       casas_ambientes.xlsx
       dicionario_insumos.xlsx
          │
          │  [01] limpeza_dados.ipynb
          ▼
   consolidado_geral.xlsx
   ├─ Materiais_Padronizados
   └─ Ambientes_Padronizados
          │
          │  [02] calculos.ipynb   (+ DIM_FATORES_EMISSAO.xlsx)
          ▼
   consolidado_geral.xlsx  (mesmas abas + 3 novas)
   ├─ QUANTITATIVOS          (materiais + aço/lajota + CO₂ mín/máx)
   ├─ CUSTO_CALCULADO        (componentes expandidos + custo)
   └─ Consolidado_Pavimento  (métricas por m² por pavimento)
          │
          │  [03] graficos.ipynb
          ├──────────────► figuras (PNG / SVG)
          └──────────────► indices_referencia_por_pavimento.xlsx  ◄─┐
                                                                     │
                                        (gerado na última seção)     │
          ┌──────────────────────────────────────────────────────────┘
          │  [04] estoque_favela.ipynb   (+ shapefile LiDAR)
          ▼
   mapa_eco2_sao_remo.png
   totais_cenarios_todos_indices.png
   estoque total por cenário (mín / central / máx)
```

## A dependência Passo 3 → Passo 4

A numeração sugere que os gráficos (Passo 3) e o estoque (Passo 4) são etapas
independentes. **Não são.** O Passo 4 não lê o `consolidado_geral.xlsx`; ele
depende do `indices_referencia_por_pavimento.xlsx`, que é gerado **apenas na
última seção do Passo 3** ("Índices finais para a favela").

Consequência prática: se você rodar apenas os gráficos iniciais do Passo 3 e
pular a última seção, o Passo 4 falhará por arquivo ausente. **Execute a última
seção do Passo 3 antes de iniciar o Passo 4.**

## Artefatos por passo

| Passo | Lê | Escreve |
|---|---|---|
| 1 · limpeza_dados | `casas_materiais.xlsx`, `casas_ambientes.xlsx`, `dicionario_insumos.xlsx` | `consolidado_geral.xlsx`, `verificacao_ultima_execucao.xlsx`, **e altera** `dicionario_insumos.xlsx` |
| 2 · calculos | `consolidado_geral.xlsx`, `DIM_FATORES_EMISSAO.xlsx` | 3 novas abas em `consolidado_geral.xlsx` |
| 3 · graficos | `consolidado_geral.xlsx`, `DIM_FATORES_EMISSAO.xlsx` | figuras PNG/SVG, `indices_referencia_por_pavimento.xlsx` |
| 4 · estoque_favela | `indices_referencia_por_pavimento.xlsx`, shapefile LiDAR | `mapa_eco2_sao_remo.png`, `totais_cenarios_todos_indices.png` |

## Efeitos colaterais em arquivos de entrada

Dois passos modificam arquivos, atenção ao versionar ou
reexecutar:

- **Passo 1 altera o `dicionario_insumos.xlsx`.** Materiais sem correspondência
  são acrescentados ao dicionário marcados como `VERIFICAR`, para revisão manual.
  Faça backup do dicionário antes da primeira execução.
- **Passo 2 reescreve o `consolidado_geral.xlsx`** em modo *append* de abas: as
  abas do Passo 1 são preservadas e as três de análise são adicionadas ou
  substituídas.

## Ciclo iterativo do Passo 1

O Passo 1 pode exigir duas passadas:

1. Primeira execução. Se terminar com `Linhas sem correspondência > 0`, novos
   materiais foram adicionados ao dicionário como `VERIFICAR`.
2. Preencha manualmente esses itens no `dicionario_insumos.xlsx` (use a aba
   `SEM_CORRESPONDENCIA` do `verificacao_ultima_execucao.xlsx` para rastrear a
   origem).
3. Reexecute o Passo 1 para gerar o consolidado final já completo.

## Pré-requisitos operacionais

- Feche os arquivos `.xlsx` de saída antes de executar cada passo — a escrita
  falha se estiverem abertos no Excel.
- Os caminhos de arquivo estão hoje embutidos no topo das células de cada
  notebook; ajuste-os para o seu ambiente antes de rodar.
- O Passo 4 requer `geopandas` (dependências geoespaciais / GDAL) e pode ser
  substituído por um fluxo equivalente no QGIS (ver o próprio notebook).
