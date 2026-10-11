# Projeto Final — Clusterização dos Estilos de Jogo do Brasileirão Série A

> **Objetivo:** O objetivo deste trabalho é desenvolver uma solução de agrupamento de dados
(clustering) a partir de uma problemática. Problemática definida: identificar perfis táticos dos clubes do Campeonato Brasileiro Série A por meio de técnicas de *clustering* e comparar os grupos em relação ao desempenho e ao risco esportivo.

## Informações acadêmicas

- **Faculdade:** CESAR School
- **Alunos:** Heitor Dornelas Câmara Cocino da Costa e Weslley Guilherme de Melo
- **Turma:** 2026.1-A
- **Cadeira/Disciplina:** Modelos Descritivos
- **Professor(a):** Matheus Soares de Lacerda
- **Semestre/Ano:** 2026.2

---

## 1. Sobre o projeto

Este trabalho investiga se os clubes do Brasileirão apresentam estilos de jogo identificáveis a partir das estatísticas acumuladas ao longo de uma temporada. Como os estilos táticos não estão previamente rotulados no conjunto de dados, foi utilizada uma abordagem de aprendizado não supervisionado.

A análise procura responder a três perguntas:

1. Quais perfis de jogo predominam entre os clubes do Brasileirão Série A?
2. Os perfis continuam distinguíveis quando as métricas são ajustadas pela força relativa do adversário (Elo) e pelo mando de campo?
3. Como os perfis se relacionam com o retorno esportivo — pontos, vitórias e saldo de gols — e com indicadores de risco, como derrotas por dois ou mais gols?

**Período analisado no modelo:** temporadas de **2018 a 2023**. Embora as tabelas derivadas cubram 2015–2023, as temporadas de 2015–2017 foram excluídas da clusterização devido a ausências e inconsistências em métricas relevantes de finalização e precisão de passes.

## 2. Fonte dos dados e tratamento

O conjunto de dados de origem é o [Brasileirao_Dataset, de adaoduque no GitHub](https://github.com/adaoduque/Brasileirao_Dataset), também publicado no [Kaggle](https://www.kaggle.com/datasets/adaoduque/campeonato-brasileiro-de-futebol).

Arquivos originais utilizados na preparação das tabelas derivadas:

- `campeonato-brasileiro-full.csv`: informações das partidas, formações, treinadores e placares.
- `campeonato-brasileiro-estatisticas-full.csv`: estatísticas dos clubes por partida.
- `campeonato-brasileiro-gols.csv`: informações dos gols, incluindo minuto e tipo.
- `campeonato-brasileiro-cartoes.csv`: foi baixado, mas **não foi utilizado** na geração das tabelas descritas na análise.

A análise documenta a geração das tabelas com o script `derivar_metricas_brasileirao.py`.

### Tabelas derivadas

| Arquivo | Granularidade | Tamanho descrito na análise | Finalidade |
|---|---|---:|---|
| `data/team_match.csv` | Um registro por clube em cada partida | 6.838 linhas × 84 colunas | Auditoria e análise jogo a jogo. Cada partida gera uma linha para cada um dos dois clubes. |
| `data/team_season.csv` | Um registro por clube e temporada | 180 linhas × 57 colunas | Base principal do projeto; utilizada para construir os clusters e comparar desempenho/risco. |
| `data/team_manager.csv` | Um registro por clube, temporada e treinador | 553 linhas × 56 colunas | Análise complementar de treinadores. |

> **Importante para executar:** o notebook carrega os três arquivos acima com caminhos relativos (`data/team_manager.csv`, `data/team_match.csv` e `data/team_season.csv`). Eles devem estar dentro de uma pasta chamada `data/`, na raiz do projeto. O notebook, isoladamente, não contém esses CSVs.

## 3. Colunas utilizadas no clustering

A tabela `team_season.csv` é filtrada para `temporada >= 2018`. O conjunto final possui **120 registros clube-temporada**: 20 clubes por temporada durante seis temporadas. O modelo recebe apenas as 12 métricas táticas abaixo, todas com o sufixo `_aj` (ajustadas).

| Coluna | Descrição | Interpretação |
|---|---|---|
| `posse_de_bola_aj` | Métrica de posse de bola ajustada | Representa o controle da bola considerando o ajuste estatístico aplicado. |
| `passes_aj` | Quantidade de passes ajustada | Representa o volume de circulação da bola. |
| `precisao_passes_aj` | Precisão de passes ajustada | Percentual de passes certos, ajustado. |
| `chutes_aj` | Quantidade de finalizações ajustada | Representa o volume total de finalizações. |
| `chutes_no_alvo_aj` | Finalizações no alvo ajustadas | Representa o volume de chutes na direção do gol. |
| `escanteios_aj` | Escanteios ajustados | Indicador complementar de presença ofensiva. |
| `faltas_aj` | Faltas ajustadas | Indicador de frequência de faltas cometidas. |
| `impedimentos_aj` | Impedimentos ajustados | Indicador de frequência de impedimentos. |
| `part_chutes_aj` | Participação relativa nas finalizações, ajustada | Parcela das finalizações da partida atribuída ao clube em relação ao total dos dois clubes, após o ajuste. |
| `part_passes_aj` | Participação relativa nos passes, ajustada | Parcela dos passes da partida atribuída ao clube em relação ao total dos dois clubes, após o ajuste. |
| `passes_por_posse_aj` | Passes por unidade de posse, ajustados | *Proxy* de ritmo/circulação da bola; a própria análise ressalta que essa interpretação não foi validada independentemente. |
| `chutes_por_posse_aj` | Chutes por unidade de posse, ajustados | Indica quanto da posse se converte em finalizações; é uma métrica derivada usada como *proxy*. |

### O que significa o sufixo `_aj`?

Conforme documentado no projeto, as métricas ajustadas são calculadas por meio do resíduo de uma regressão linear da métrica em função de `dif_elo` (diferença entre a força Elo do clube e a do adversário) e `casa` (mando de campo), somado à média geral. A intenção é reduzir a influência dessas duas condições nas comparações de estilo.

O Elo utilizado foi construído com parâmetros definidos no projeto e não calibrados externamente. Assim, o ajuste reduz certos efeitos estimados, mas não garante que toda diferença de qualidade entre elencos tenha sido removida.

### Colunas de identificação e de desempenho

As colunas seguintes **não são utilizadas como features do K-Means**. Elas são preservadas para identificar as observações, descrever os grupos e analisar os resultados depois do agrupamento.

| Coluna | Uso na análise |
|---|---|
| `temporada` | Identifica a temporada do registro. |
| `time` | Identifica o clube. |
| `pts_jogo` | Média de pontos conquistados por jogo; usada como indicador de retorno. |
| `pct_vitorias` | Proporção de vitórias na temporada. |
| `saldo_jogo` | Saldo médio de gols por jogo. |
| `gp_jogo` / `gc_jogo` | Médias de gols marcados e sofridos por jogo. |
| `risco_pts_dp` | Desvio-padrão dos pontos por jogo; usado como indicador de irregularidade. |
| `risco_saldo_dp` | Desvio-padrão do saldo de gols por jogo. |
| `risco_derrota_2mais` | Proporção de jogos perdidos por dois ou mais gols. |
| `pct_sem_marcar` | Proporção de jogos sem marcar gols. |
| `pct_sem_sofrer` | Proporção de jogos sem sofrer gols. |

Outras colunas de `team_season`, como `jogos`, `jogos_com_stats`, `n_tecnicos` e as métricas sem sufixo `_aj`, existem na tabela derivada, mas não integram as 12 features usadas no ajuste final do K-Means.

## 4. Metodologia

O notebook executa as seguintes etapas:

1. **Carregamento:** leitura das tabelas derivadas em CSV com pandas.
2. **Análise exploratória:** inspeção de valores ausentes, duplicidades, distribuições, valores extremos e correlações.
3. **Recorte temporal:** seleção das temporadas de 2018 a 2023.
4. **Seleção das features:** uso exclusivo das 12 métricas táticas ajustadas listadas acima.
5. **Padronização:** transformação Z-score com `StandardScaler`, para colocar as métricas em escalas comparáveis.
6. **Clusterização:** aplicação de K-Means com `k=3`, `random_state=42` e `n_init=10`.
7. **Avaliação:** análise do cotovelo, coeficiente de Silhouette, índice de Dunn, Davies–Bouldin, comparação com agrupamento hierárquico de Ward, ANOVA e testes de estabilidade com diferentes sementes e subamostras.
8. **Interpretação:** comparação do retorno e do risco por grupo; os indicadores de desempenho não são usados para formar os clusters.

Os nomes dos grupos foram atribuídos a partir da leitura das médias das métricas táticas após o agrupamento. Eles são interpretações dos perfis encontrados, não rótulos táticos oficiais.

## 5. Principais resultados

O modelo identificou três perfis entre as 120 observações clube-temporada:

| Perfil interpretado | Observações | Pontos médios por jogo | Vitórias médias | Saldo médio por jogo | Derrotas por 2+ gols |
|---|---:|---:|---:|---:|---:|
| Ataque Direto / Transição Rápida | 35 | 1,51 | 41,7% | +0,23 | 12,1% |
| Reativo / Bloco Baixo | 44 | 1,35 | 35,6% | -0,03 | 15,4% |
| Propositivo / Controle de Posse | 41 | 1,25 | 31,8% | -0,16 | 15,5% |

Principais leituras registradas no notebook:

- **Ataque Direto / Transição Rápida:** apresentou, nesta amostra, a maior média de pontos, maior percentual de vitórias e menor proporção de derrotas por dois ou mais gols.
- **Reativo / Bloco Baixo:** apresentou retorno intermediário; o exemplo do Palmeiras mostra que esse perfil também pode aparecer em temporadas campeãs.
- **Propositivo / Controle de Posse:** apresentou a menor média de pontos e menor volume médio de finalizações, apesar da circulação de bola.
- O Silhouette médio para `k=3` foi de aproximadamente **0,209**. A análise registra sobreposição entre os grupos; os perfis devem ser entendidos como predominantes, não como categorias totalmente separadas.

Esses resultados são descritivos e se referem ao período e à amostra analisados. Associação estatística não implica causalidade, e os achados não devem ser generalizados automaticamente para outras temporadas, divisões ou competições.

## 6. Como executar

### Pré-requisitos

- Python 3.10 ou superior (recomendado).
- Os arquivos `team_manager.csv`, `team_match.csv` e `team_season.csv` dentro da pasta `data/`.
- Git, caso o projeto seja obtido clonando o repositório.

### Estrutura esperada

```text
.
├── README.md
├── requirements.txt
├── clusterizacao_estilos_jogo_brasileirao.ipynb
├── derivar_metricas_brasileirao.py
└── data/
    ├── team_manager.csv
    ├── team_match.csv
    └── team_season.csv
```

### Instalação e execução local

1. Clone o repositório e entre na pasta do projeto:

   ```bash
   git clone https://github.com/Weslley-Melo/Projeto-Final-Agrupamento-de-Dados.git
   ```

2. Crie e ative um ambiente virtual.

   **Windows — PowerShell:**

   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

   **Linux/macOS:**

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

3. Instale as dependências:

   ```bash
   python -m pip install --upgrade pip
   pip install -r requirements.txt
   ```

4. Confira se a pasta `data/` contém os três CSVs exigidos pelo notebook.

5. Abra o Jupyter Notebook:

   ```bash
   jupyter notebook
   ```

6. Abra `clusterizacao_estilos_jogo_brasileirao.ipynb` e execute as células em ordem, de cima para baixo. No Jupyter, a opção **Kernel → Restart Kernel and Run All Cells** ajuda a garantir que as células sejam executadas na sequência correta.

## 7. Tecnologias utilizadas

- **Python** — linguagem de programação.
- **pandas / NumPy** — manipulação de dados e cálculos numéricos.
- **Matplotlib / Seaborn** — gráficos e análise visual.
- **SciPy** — testes estatísticos, distâncias e clusterização hierárquica.
- **scikit-learn** — padronização, K-Means e métricas de avaliação.
- **Jupyter Notebook** — ambiente de execução e documentação da análise.

## 8. Limitações e cuidados de interpretação

- As temporadas de 2015 e 2016 apresentam inconsistências em estatísticas de finalização; de 2015 a 2017 há ausência de precisão de passes. Por isso, o modelo foi limitado a 2018–2023.
- Métricas como `passes_por_posse_aj` e `chutes_por_posse_aj` são *proxies* e não foram validadas independentemente.
- Algumas variáveis táticas são correlacionadas. A padronização iguala escalas, mas não elimina redundância nem garante contribuição independente de cada variável.
- O agrupamento apresenta sobreposição (Silhouette médio próximo de 0,21) e sensibilidade a subamostras; não deve ser interpretado como uma classificação definitiva dos clubes.

## 9. Fontes e licença

### Fonte dos dados

- Duque, Adão. [Brasileirao_Dataset — GitHub](https://github.com/adaoduque/Brasileirao_Dataset).
- Duque, Adão. [Campeonato Brasileiro de futebol — Kaggle](https://www.kaggle.com/datasets/adaoduque/campeonato-brasileiro-de-futebol).

## 10. Créditos

Projeto desenvolvido como trabalho acadêmico para a disciplina indicada na seção **Informações acadêmicas**. Agradecemos ao mantenedor do conjunto de dados original pela disponibilização dos registros utilizados na análise.

---
