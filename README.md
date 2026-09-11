# MORE — Cálculo de CO₂ Incorporado a partir de Dados BIM

Repositório de códigos para análise de CO2 em moradias autoconstruídas modeladas em BIM.

Pipeline de processamento que recebe quantitativos de materiais e ambientes
exportados de modelos **BIM**, padroniza os insumos contra um dicionário de
referência, calcula **emissões de CO₂** (mínimo/máximo) e **custos**, gera as
figuras de análise e, por fim, infere o **estoque total de CO₂ incorporado** de
uma comunidade a partir de polígonos de edificações (LiDAR).

Desenvolvido no âmbito do projeto **MORE** (GT Materialização), com aplicação na
Comunidade São Remo (São Paulo).

---

## Sumário

- [Fluxo do pipeline](#fluxo-do-pipeline)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Instalação](#instalação)
- [Dados de entrada](#dados-de-entrada)
- [Como executar](#como-executar)
- [Saídas](#saídas)
- [Premissas metodológicas](#premissas-metodológicas)
- [Roadmap](#roadmap)
- [Licença e citação](#licença-e-citação)

---

## Fluxo do pipeline

Os notebooks são **sequenciais** e devem ser executados na ordem numérica. Há uma
dependência que não é óbvia pela numeração: o **Passo 4 depende de um arquivo
gerado na última célula do Passo 3**, e não do consolidado geral diretamente.

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
          │  [04] estoque_favela.ipynb   (+ shapefile LiDAR São Remo)
          ▼
   mapa_eco2_sao_remo.png
   totais_cenarios_todos_indices.png
   estoque total por cenário (mín / central / máx)
```

> **Atenção:** o `indices_referencia_por_pavimento.xlsx`, insumo do Passo 4, só é
> criado quando a **última seção do Passo 3** ("Índices finais para a favela") é
> executada. Rodar apenas os gráficos iniciais não o produz.

---

## Estrutura do repositório

```
more-co2-bim/
├── README.md
├── LICENSE
├── requirements.txt
├── .gitignore
│
├── notebooks/
│   ├── 01_limpeza_dados.ipynb     # padroniza materiais/ambientes contra o dicionário
│   ├── 02_calculos.ipynb          # aço, lajotas, CO₂, custo, consolidado por m²
│   ├── 03_graficos.ipynb          # figuras + tabela de índices por pavimento
│   └── 04_estoque_favela.ipynb    # inferência do estoque total (opcional: QGIS)
│
├── data/
│   ├── raw/                       # entradas BIM (NÃO versionado — ver seção Dados)
│   ├── external/                  # fatores de emissão + shapefile LiDAR
│   ├── interim/                   # consolidado_geral.xlsx
│   └── processed/                 # indices_referencia_por_pavimento.xlsx
│
├── outputs/
│   ├── figures/                   # PNG/SVG dos gráficos
│   └── maps/                      # mapa de estoque de CO₂
│
└── docs/
    ├── fluxo_pipeline.md
    ├── premissas_metodologicas.md
    └── dicionario_dados.md
```

Os arquivos de dados brutos e o shapefile **não são versionados** (ver
`.gitignore`). Consulte a seção [Dados de entrada](#dados-de-entrada) para saber
como obtê-los.

> **Nota:** os caminhos de arquivo e as premissas de cálculo (traços, fatores,
> mapeamento de tipologias) estão hoje embutidos diretamente nas células dos
> notebooks. A pasta `data/` acima é a organização sugerida para os arquivos;
> ajuste os caminhos no topo de cada notebook para apontar para ela. Uma
> centralização mais robusta dessas configurações está prevista no
> [roadmap](#roadmap).

---

## Instalação

Requer Python 3.10+.

```bash
git clone <url-do-repositorio>
cd more-co2-bim

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

Dependências principais: `pandas`, `numpy`, `openpyxl`, `matplotlib`, `scipy`,
`geopandas`. O `geopandas` (usado apenas no Passo 4) tem dependências geoespaciais
(GDAL); em caso de dificuldade na instalação via pip, recomenda-se `conda`.

Antes de rodar, ajuste os caminhos de arquivo no topo das células de cada
notebook para o seu ambiente (hoje eles apontam para caminhos absolutos locais).

---

## Dados de entrada

| Arquivo | Local esperado | Descrição |
|---|---|---|
| `casas_materiais.xlsx` | `data/raw/` | Materiais por moradia — **uma aba por casa** |
| `casas_ambientes.xlsx` | `data/raw/` | Ambientes por moradia — **uma aba por casa** |
| `dicionario_insumos.xlsx` | `data/raw/` | Aba `dicionario_insumos`; mapeia material BIM → insumo/elemento/categoria |
| `DIM_FATORES_EMISSAO.xlsx` | `data/external/` | Colunas: `insumo`, `fator_emissao_min`, `fator_emissao_max`, `custo_medio`, `unidade` |
| Shapefile São Remo | `data/external/Favela_SaoRemo_SHP/` | Polígonos LiDAR; colunas `area`, `num_pav1` |

Os dados brutos não são disponibilizados no repositório, por conterem informação
que não deve ir para o controle de versão. Apresenta-se um template esperado de cada arquivo, que no fluxo atual ainda dependem de edição e conferência manual das planilhas individuais de cada moradia exportadas do BIM.

> **Efeito colateral importante do Passo 1:** ao encontrar materiais sem
> correspondência no dicionário, a rotina **acrescenta esses itens ao próprio
> `dicionario_insumos.xlsx`**, marcados como `VERIFICAR`. Trate esse arquivo como
> mutável e faça backup antes da primeira execução. Verifique as adições manualmente antes de segui para as etapas seguintes.

---

## Como executar

Execute os notebooks em ordem. Feche os arquivos `.xlsx` de saída antes de rodar
(a escrita falha se o Excel estiver aberto).

1. **`01_limpeza_dados.ipynb`** — gera `consolidado_geral.xlsx` e
   `verificacao_ultima_execucao.xlsx`.
   Se a execução reportar `Linhas sem correspondência > 0`, preencha os novos
   itens marcados `VERIFICAR` em `dicionario_insumos.xlsx` (use a aba
   `SEM_CORRESPONDENCIA` do arquivo de verificação para rastrear a origem) e
   **execute novamente**.

2. **`02_calculos.ipynb`** — adiciona as abas `QUANTITATIVOS`, `CUSTO_CALCULADO`
   e `Consolidado_Pavimento` ao `consolidado_geral.xlsx`.

3. **`03_graficos.ipynb`** — gera as figuras e, na última seção, exporta
   `indices_referencia_por_pavimento.xlsx` (**necessário para o Passo 4**).

4. **`04_estoque_favela.ipynb`** — cruza os índices por pavimento com os
   polígonos das edificações e calcula o estoque total. *Este passo pode ser
   substituído por um fluxo equivalente no QGIS:* join tabular
   índices ↔ polígonos e campo calculado com a mesma regra de empilhamento por
   pavimento.

---

## Saídas

| Passo | Arquivo | Conteúdo |
|---|---|---|
| 1 | `consolidado_geral.xlsx` | Materiais e ambientes padronizados |
| 1 | `verificacao_ultima_execucao.xlsx` | Diagnóstico: sem correspondência, duplicatas, dados brutos |
| 2 | `consolidado_geral.xlsx` (abas novas) | Quantitativos, custos, consolidado por m² |
| 3 | `outputs/figures/*.png / *.svg` | Gráficos de CO₂, materiais, custo por tipo e pavimento |
| 3 | `indices_referencia_por_pavimento.xlsx` | Tabela de índices (mín/central/máx) por nível |
| 4 | `outputs/maps/mapa_eco2_sao_remo.png` | Mapa coroplético de estoque por cenário |
| 4 | `totais_cenarios_todos_indices.png` | Totais da comunidade por índice e cenário |

---

## Premissas metodológicas

Documentação completa em [`docs/premissas_metodologicas.md`](docs/premissas_metodologicas.md).
Em resumo:

- **Unidade amostral:** a edificação (casa), não o pavimento. Os pavimentos de
  cada `id_moradia` são somados antes de qualquer estatística.
- **Envelope de incerteza:** cada insumo tem fator de emissão mínimo e máximo; o
  CO₂ é reportado como faixa (mín–máx), não como valor único.
- **Intervalos de confiança:** bootstrap percentil 95% sobre as edificações;
  amostras com n=1 não recebem IC (marcadas com `*`).
- **Aço e lajotas:** o aço é estimado a partir do volume de concreto
  (fator padrão: 50 kg/m³); lajotas de enchimento por área
  (fator padrão: 13 un/m²). *Fonte dos fatores: (preencher).*
- **Expansão de componentes:** insumos "pai" (concreto e argamassa São Remo) são
  decompostos em brita, areia e cimento segundo traços locais definidos no bloco
  de configuração no início da função `processar_analise_consolidada`
  (Passo 2). *Fonte dos traços: análise de entrevistas com pedreiros e lojistas.*
- **Regra de empilhamento (Passo 4):** térreo sempre presente; cada pavimento
  adicional soma o índice do nível correspondente; a partir do 3º pavimento
  aplica-se o índice de "≥ Pavimento 3".
- **Custo:** calculado apenas para os insumos listados em `INSUMOS_COM_CUSTO`;
  insumos fora da lista entram no CO₂ mas não no custo.

---

## Roadmap

Melhorias planejadas para versões futuras. **Nenhuma é necessária para rodar o
pipeline** — os notebooks funcionam como estão. A lista serve para registrar a
dívida técnica conhecida e orientar quem for evoluir o repositório.

### Higiene de repositório (não exige alterar o código)

- [ ] **Limpar as saídas dos notebooks antes de versionar.** `graficos.ipynb`
      (~1,2 MB) e `estoque_favela.ipynb` (~634 KB) devem quase todo o peso às
      figuras embutidas. Rodar
      `jupyter nbconvert --clear-output --inplace notebooks/*.ipynb` antes de
      commitar, ou instalar [`nbstripout`](https://github.com/kynan/nbstripout)
      como filtro do Git, mantém os diffs legíveis e o repositório enxuto.
- [ ] **Não versionar dados brutos e shapefile** (via `.gitignore`); documentar
      onde obtê-los.
- [ ] **Preencher os campos pendentes** deste README e dos docs: fontes dos
      coeficientes (aço, lajota, traços), licença, local dos dados, citação.

### Refatoração de código (para quando houver tempo de mexer nos notebooks)

- [ ] **Externalizar caminhos de arquivo.** Substituir os caminhos absolutos
      `/Users/.../` por caminhos relativos à pasta `data/`, ou por um pequeno
      bloco de configuração no topo de cada notebook.
- [ ] **Centralizar `config_tipos`.** O mapeamento id_moradia → tipologia está
      redefinido em ~5 células do Passo 3. Unificar numa única definição (célula
      de configuração no início do notebook, ou arquivo externo) evita
      dessincronização ao atualizar a amostra.
- [ ] **Remover o código de diagnóstico do Passo 1.** A função
      `padronizar_bim_multi_casa` ainda contém instruções `DEBUG n` e um bloco
      `try/except` de depuração de índice duplicado, mantidos temporariamente.
- [ ] **Adicionar docstrings** às funções principais (`padronizar_bim_multi_casa`
      e helpers), com `Args`/`Returns`/`Raises`.
- [ ] **Revisar a lógica do `divisor`** no Passo 4 (célula de totais): a variável
      é calculada condicionalmente mas sempre resulta em `1000`, e o valor fixo
      `divisor=1000` é o que de fato é usado — a linha condicional é código morto.

---

## Licença e citação

Licença: MIT.

Se utilizar este pipeline, cite: (preencher futuramente com a referência do projeto MORE /
publicação associada).
