[README.md](https://github.com/user-attachments/files/32264527/README.md)
# MVP: Pipeline de Dados na Nuvem — Airbnb Rio de Janeiro

> Trabalho individual — Engenharia de Dados
> Plataforma: Databricks Free Edition

---

## Contexto de Negócio e Perguntas (Etapa 2 e 4.1)

**Problema de negócio:**
Quero entender quais fatores determinam o preço da diária de imóveis Airbnb no Rio de
Janeiro e como esse mercado varia ao longo do ano.

**Perguntas de negócio:**
1. Qual o impacto da localização (bairro) no preço da diária?
2. O tipo de imóvel e a capacidade de hóspedes influenciam mais que a localização?
3. Existe correlação entre avaliação do anúncio (review score) e preço cobrado?
4. Os preços variam significativamente entre alta e baixa temporada?
5. Superhosts conseguem cobrar preços acima da média do mercado?

**Sobre os dados brutos:**
Dataset "Rio de Janeiro Airbnb Open Data", publicado no Kaggle pelo autor `allanbruno`,
com dados coletados seguindo a metodologia do Inside Airbnb.
- **Volume:** 25 arquivos CSV mensais (abril/2018 a maio/2020), totalizando 902.210 registros
- **Estrutura:** 96 colunas originais por arquivo (id, preço, localização, características do
  imóvel, avaliações, dados do host, disponibilidade)
- **Licença:** CC0: Public Domain — uso, redistribuição e modificação livres, para qualquer
  finalidade, sem restrição legal nem exigência formal de atribuição. Por transparência,
  este trabalho credita a fonte: dataset publicado por `allanbruno` no Kaggle.

![Página do dataset no Kaggle](imagens/01_bronze_kaggle_licenca.png)

---

## Carga dos Dados (Etapa 4.2)

Os 25 arquivos CSV mensais foram baixados do Kaggle e enviados para um **Volume do Unity
Catalog** (`/Volumes/mvp_airbnb_rj/bronze/raw_files/`) via upload direto pela interface do
Databricks.

**Achado de qualidade de dados na coleta:** 3 dos 25 arquivos vieram com nomes corrompidos
na fonte original — `maro2019.csv`, `maro2020.csv` (faltando o "ç" de "março") e
`novrmbro2018.csv` (faltando o "e" de "novembro"). Tratamos isso via mapeamento explícito no
código de ingestão (ver `01_bronze_ingestao_airbnb_rj.ipynb`), preservando os arquivos
originais intactos.

![Volume com os 25 arquivos CSV](imagens/01_bronze_volume_arquivos_csv.png)

Script de ingestão: [`01_bronze_ingestao_airbnb_rj.ipynb`](01_bronze_ingestao_airbnb_rj.ipynb)

---

## Modelagem e Catálogo de Dados (Etapa 4.3)

Adotamos o **Esquema Estrela**, com grão da tabela fato = um registro por imóvel por mês.

### Estrutura das tabelas

| Tabela | Papel | Grão |
|---|---|---|
| `fato_diarias_airbnb` | Fato | (sk_imovel, data_referencia) |
| `dim_localizacao` | Dimensão | bairro + cidade |
| `dim_imovel` | Dimensão | imóvel (snapshot mais recente) |
| `dim_host` | Dimensão | host (snapshot mais recente) |
| `dim_tempo` | Dimensão | mês/ano, com estação do ano |

![Schema Gold com as 5 tabelas](imagens/04_gold_catalog_gold_schema.png)

### Catálogo de Dados

O catálogo foi documentado via `COMMENT ON TABLE` e `ALTER TABLE ... ALTER COLUMN COMMENT`
diretamente no notebook de modelagem — garantindo que a documentação fique versionada junto
com o código, e não dependa apenas de preenchimento manual na interface.

**Tabela fato_diarias_airbnb:**
Uma linha por imóvel por mês, com métricas que variam no tempo (preço, avaliações,
disponibilidade, status de superhost no mês).

| Coluna | Tipo | Descrição | Domínio |
|---|---|---|---|
| `sk_imovel` | int | FK para dim_imovel (id original do Airbnb) | — |
| `sk_host` | int | FK para dim_host (host_id original) | — |
| `sk_localizacao` | int | FK para dim_localizacao | — |
| `data_referencia` | date | FK para dim_tempo (1º dia do mês) | 2018-04 a 2020-05 |
| `price` | double | Preço da diária em reais (BRL) | ≥ 0 |
| `flag_price_zero` | boolean | TRUE quando price = 0 (dado inválido) | true/false |
| `flag_price_outlier` | boolean | TRUE quando price > percentil 99 (R$5.618) | true/false |
| `host_is_superhost` | boolean | Status de superhost no mês de referência | true/false/null |
| *(demais colunas)* | — | *(ver código-fonte para lista completa)* | — |

*(Repita esse padrão de tabela para `dim_localizacao`, `dim_imovel`, `dim_host`, `dim_tempo`
— os textos completos das descrições já estão no notebook `04_gold_modelagem_airbnb_rj.ipynb`,
seção 7, é só transcrever.)*

![Catálogo da tabela fato no Unity Catalog](imagens/04_gold_catalogo_fato_diarias.png)
![Catálogo da dim_localizacao no Unity Catalog](imagens/04_gold_catalogo_dim_localizacao.png)

Script de modelagem: [`04_gold_modelagem_airbnb_rj.ipynb`](04_gold_modelagem_airbnb_rj.ipynb)

---

## Pipeline de Dados (Etapa 4.4)

O pipeline segue a **Arquitetura Medalhão**, com um notebook por camada:

| Notebook | Camada | O que faz |
|---|---|---|
| `01_bronze_ingestao_airbnb_rj.ipynb` | Bronze | Ingestão dos 25 CSVs, sem transformação de conteúdo, com metadados de rastreabilidade |
| `02_qualidade_dados_airbnb_rj.ipynb` | — | Análise de completude, consistência, unicidade, acurácia e outliers |
| `03_silver_transformacao_airbnb_rj.ipynb` | Silver | Limpeza, tipagem correta, padronização, flags de qualidade |
| `04_gold_modelagem_airbnb_rj.ipynb` | Gold | Modelagem em Esquema Estrela + Catálogo de Dados |
| `05_analise_airbnb_rj.ipynb` | — | Consultas e visualizações respondendo às 5 perguntas de negócio |

**Transformações principais documentadas:**
- Conversão de campos monetários de texto (`"$1,200.00"`) para decimal, via `try_cast`
  (necessário pelo modo ANSI SQL do Unity Catalog, que quebra com `.cast()` simples em
  valores malformados)
- Correção de tipos inconsistentes entre arquivos mensais (ex: `review_scores_cleanliness`
  vinha como string em vez de integer)
- Padronização de `city` (trim + capitalização), corrigindo duplicação de bairros causada
  por variações como "Rio de Janeiro" vs. "Rio de janeiro"

![Execução do notebook Bronze](imagens/01_bronze_execucao_mes_referencia.png)
![Tabela Bronze persistida](imagens/01_bronze_catalog_listings_bronze.png)
![Validação da camada Silver](imagens/03_silver_validacao_schema_silver.png)
![Tabela Silver persistida](imagens/03_silver_catalog_listings_silver.png)
![Execução das 5 tabelas Gold](imagens/04_gold_execucao_5_tabelas.png)

---

## Qualidade de Dados (Etapa 4.5)

**Metodologia:** avaliamos completude, consistência, unicidade, acurácia e outliers sobre a
camada Bronze, antes de definir as transformações da Silver.

**Principais achados:**
- **Completude:** campos administrativos (`instant_bookable`, `cancellation_policy`, etc.)
  com ~32% de nulos — descartados por serem irrelevantes às perguntas de negócio. Campos de
  avaliação (`review_scores_*`) com ~18% de nulos — ausência legítima (imóveis sem reviews).
- **Unicidade:** 0 duplicatas de (id + mês) na Bronze.
- **Acurácia (preço):** identificados 264 registros com `price = 0` (dado inválido, excluído
  das análises) e forte presença de outliers extremos (percentil 99 = R$5.618, mas com
  valores isolados chegando a dezenas de milhares de reais).
- **Acurácia (geolocalização):** 0 registros com coordenadas fora da faixa esperada do Rio
  de Janeiro.
- **Inconsistência de tipos entre arquivos mensais:** colunas irmãs (ex: notas de avaliação)
  vieram com tipos diferentes dependendo do mês, corrigido via `try_cast` na Silver.
- **Investigação de outliers específicos:** o anúncio `sk_imovel = 12871727` (bairro Bangu)
  apresentou preço persistente de ~R$38.000-39.000 em múltiplos meses — padrão mais
  compatível com erro de cadastro (ex: confusão entre aluguel mensal e diária) do que com
  imóvel de luxo genuíno.

![Completude de nulos por coluna](imagens/02_qualidade_completude_nulos.png)
![Análise de preço (describe)](imagens/02_qualidade_analise_preco_describe.png)
![Checagem de coordenadas](imagens/02_qualidade_coordenadas_validas.png)
![Investigação de outliers em bairros](imagens/05_analise_p1_investigacao_outliers.png)

Script: [`02_qualidade_dados_airbnb_rj.ipynb`](02_qualidade_dados_airbnb_rj.ipynb)

---

## Análise de Dados (Etapa 4.5)

### 1. Qual o impacto da localização (bairro) no preço da diária?

*(TODO: escreva aqui sua interpretação, usando os números reais que apareceram na sua
execução — ranking por mediana, bairros no topo, magnitude da diferença entre o mais caro e
o mais barato)*

![Ranking de bairros por mediana](imagens/05_analise_p1_tabela_bairros_mediana.png)
![Gráfico de bairros](imagens/05_analise_p1_grafico_bairros.png)

### 2. O tipo de imóvel e a capacidade influenciam mais que a localização?

*(TODO: compare a amplitude de variação de preço entre bairros (pergunta 1) com a amplitude
entre tipos/capacidades — qual pesa mais?)*

![Tabela tipo x capacidade](imagens/05_analise_p2_tabela_tipo_capacidade.png)
![Gráfico tipo x capacidade](imagens/05_analise_p2_grafico_tipo_capacidade.png)

### 3. Existe correlação entre avaliação e preço cobrado?

*(TODO: reporte o coeficiente de correlação calculado e discuta o achado contraintuitivo —
a faixa "95-100 excelente" teve o MENOR preço médio de todas as faixas)*

![Tabela por faixa de nota](imagens/05_analise_p3_tabela_faixa_nota.png)
![Gráfico por faixa de nota](imagens/05_analise_p3_grafico_faixa_nota.png)

### 4. Os preços variam entre alta e baixa temporada?

*(TODO: discuta a diferença entre estações e, principalmente, a limitação identificada — o
salto de preço entre fev-maio/2020 coincide com o início da pandemia de COVID-19, um
confundidor que não representa sazonalidade "normal")*

![Gráfico por estação](imagens/05_analise_p4_grafico_estacao_ano.png)
![Série mensal completa](imagens/05_analise_p4_grafico_serie_mensal.png)

### 5. Superhosts conseguem cobrar preços acima da média?

*(TODO: reporte o achado contraintuitivo — superhosts cobraram, em média, MENOS que
não-superhosts (R$404 vs. R$677), padrão que se mantém mesmo controlando por tipo de
imóvel, exceto em Hotel room)*

![Tabela superhost geral](imagens/05_analise_p5_tabela_superhost_geral.png)
![Gráfico superhost por tipo](imagens/05_analise_p5_grafico_superhost_por_tipo.png)

### Discussão geral

*(TODO: conecte as 5 respostas de volta ao problema original. Qual fator mostrou maior
impacto no preço? Quais perguntas tiveram resposta clara vs. inconclusiva? Que limitações
dos dados podem ter afetado as conclusões?)*

Script: [`05_analise_airbnb_rj.ipynb`](05_analise_airbnb_rj.ipynb)

---

## Autoavaliação

*(TODO: discuta se você atingiu os objetivos traçados no início do trabalho, as
dificuldades encontradas ao longo da execução, e trabalhos futuros para enriquecer o
problema e a solução. Ver roteiro de perguntas-guia combinado com seu orientador.)*
