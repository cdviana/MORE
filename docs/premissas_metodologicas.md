# Premissas metodológicas

Este documento reúne as decisões de método que definem os resultados do pipeline.
São convenções do projeto MORE aplicadas à Comunidade São Remo; ao adaptar o
pipeline a outra base construtiva, revise cada uma delas.

> Os fatores, traços e demais coeficientes de referência são detalhados no
> **manual MORE**, que é a fonte primária. Este documento registra como cada
> premissa é aplicada no código e onde ajustá-la.

## Decisão de encadeamento

A versão inicial dos códigos considera apenas o fluxo de trabalho estabelecido para a amostra
da São Remo, e portanto possui registrado as premissas dessa comunidade específica. 
Diante da variabilidade de casos amostrados, o ciclo de análise encerrou sugerindo a manutenção
de um fluxo semi automatizado, que exige verificação e validação manual a cada nova moradia adicionada à base.
Esse resultado condicionou a entrega do fluxo em notebooks independentes e segmentados, favorecendo a
rastreabilidade de erros e inconsistências. Espera-se que com o aumento da base de casas modeladas, algumas etapas possam
ser integradas e automatizadas completamente futuramente, mitigando possíveis erros humanos no processo.

## As duas amostras (n=19 e n=39)

O projeto trabalhou com **duas amostras distintas, por desenho** — não é uma
inconsistência. A escolha entre elas depende da unidade de análise:

- **n=19 — moradias medidas integralmente.** Usada nas **análises por
  edificação** (bloco "por tipo construtivo" do Passo 3). Somar os pavimentos
  para obter o total de uma casa só faz sentido quando a edificação inteira foi
  levantada po escaneamento e modelagem; por isso apenas as 19 moradias completas entram aqui.
- **n=39 — total de moradias visitadas.** Usada nas **análises por nível de
  pavimento**. Inclui pavimentos de edificações que não foram medidas
  integralmente: como a unidade de análise é o pavimento, cada pavimento medido é
  uma observação válida, mesmo isolado.

Ao ler qualquer figura, verifique qual é a unidade amostral — as duas famílias de
análise respondem a perguntas diferentes e não são diretamente comparáveis.

## Envelope de incerteza (faixa mín–máx)

Cada insumo tem um fator de emissão **mínimo** e um **máximo**. O CO₂ é sempre
reportado como uma faixa (mín–máx), nunca como valor único. As figuras deixam
explícito, em rodapé, que essa faixa reflete a incerteza **dos fatores de
emissão** — separada da variabilidade amostral entre edificações/pavimentos, que
é tratada por bootstrap.

## Intervalos de confiança (bootstrap)

Onde há estatística sobre a amostra, o intervalo de confiança (IC) é estimado por **bootstrap percentil
95%** (2000 reamostragens). Amostras com uma única observação (n=1) não recebem
IC e são marcadas com `*` nas figuras. Este IC captura a variabilidade entre
unidades amostrais (casas ou pavimentos), e **não** a incerteza dos fatores de
emissão.

## Insumos derivados (Passo 2)

Dois insumos não vêm diretamente do BIM e são estimados por regra:

- **Aço:** estimado a partir do volume de concreto São Remo, gerando linhas de
  `Vergalhão de Aço CA-50`. Fator padrão no código: **50 kg por m³** de concreto.
- **Lajotas de enchimento:** estimadas por área. Fator padrão: **13 unidades por
  m²**.

Ambos os fatores estão no bloco de configuração no início da função
`processar_analise_consolidada`. Valores e justificativa: ver manual MORE.

## Expansão de insumos-pai em componentes (Passo 2)

Concretos e argamassas ("insumos-pai") são decompostos em brita, areia e cimento
para o cálculo de custo, segundo os traços locais (em kg por m³) definidos no
dicionário `COMPOSICAO`. Os valores aplicados estão detalhados no manual MORE.

Consequência importante: a coluna de **cimento** só existe **após** essa
expansão. Por isso, no resumo por m² (`Consolidado_Pavimento`), o total de aço é
somado a partir de `QUANTITATIVOS` (o aço é criado como linha própria antes da
expansão), enquanto o total de cimento vem de `CUSTO_CALCULADO` (pós-expansão).

## Insumos com custo (Passo 2)

O custo é calculado **apenas** para os insumos listados em `INSUMOS_COM_CUSTO`. Um
insumo fora dessa lista **é contabilizado no CO₂ mas não no custo**. Ao
interpretar a aba `CUSTO_CALCULADO` ou os totais de custo, lembre que ela não
cobre todos os insumos do modelo.

## Ajuste de pavimento — *shift down* (Passo 1)

Alguns materiais estruturais de cobertura são modelados no BIM um pavimento acima do que
representam fisicamente. Para edificações com mais de um pavimento, os materiais
que casam com os padrões `ESTRUTURA CONCRETO VIGAS`, `TERÇA MADEIRA` e
`TELHA FIBROCIMENTO` têm o pavimento **reduzido em um nível**
(`SHIFT_DOWN_PATTERNS`). É uma convenção de modelagem local — revise os padrões
ao usar outra base.

## Regra de empilhamento por pavimento (Passo 4)

O estoque de CO₂ de cada edificação é a soma das contribuições de cada nível,
sempre multiplicadas pela área de projeção (footprint):

- **1 pavimento:** índice do térreo × área.
- **2 pavimentos:** + índice do Pavimento 1 × área.
- **3 pavimentos:** + índice do Pavimento 2 × área.
- **≥ 4 pavimentos:** + (nº pav − 3) × índice do "≥ Pavimento 3" × área.

Do 4º pavimento em diante, todos os níveis extras usam o mesmo índice de
referência ("≥ Pavimento 3"), mas isso pode ser revisto de acordo com
o nível de verticalizarão das edificações representado na amostra. 
Edificações sem número de pavimentos válido são
excluídas.

## Cenários de referência (Passo 3 → Passo 4)

A tabela `indices_referencia_por_pavimento.xlsx` resume cada índice, por nível,
em três cenários derivados da dispersão amostral:

- **Cenário mínimo:** média − 1 desvio-padrão.
- **Central:** média.
- **Cenário máximo:** média + 1 desvio-padrão.

Esses três cenários são propagados no Passo 4 para produzir os totais mínimo,
central e máximo da comunidade.
