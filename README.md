# APIs, energias renováveis e aprendizado de máquina

## Objetivo

Consultar duas APIs públicas de energia/clima, gerar dois conjuntos de dados e resolver duas tarefas de aprendizado de máquina, comparando **três algoritmos em cada uma**:

1. **Classificação:** prever a fonte de um empreendimento de geração (**Solar, Eólica ou Hidráulica**) a partir da potência outorgada e da localização.
2. **Regressão:** estimar a **radiação solar global horizontal** (W/m²) em Petrolina (PE) a partir de variáveis meteorológicas e da hora do dia.

## Origem e período dos dados

| Arquivo | Fonte | Recorte | Colunas |
|---|---|---|---|
| `aneel_classificacao_orange.csv` | [ANEEL — SIGA](https://dadosabertos.aneel.gov.br/dataset/siga-sistema-de-informacoes-de-geracao-da-aneel) (API CKAN `datastore_search`, recurso `11ec447d-698d-4ab8-977f-b424d5deee6a`) | Até 1.200 registros por sigla (`UFV`, `EOL`, `UHE`, `PCH`, `CGH`); UHE+PCH+CGH = Hidráulica | `potencia_kw`, `latitude`, `longitude`, `fonte` |
| `meteo_regressao_orange.csv` | [Open-Meteo — API histórica](https://open-meteo.com/en/docs/historical-weather-api) | Petrolina (PE), −9,39 / −40,50, de 01/04/2025 a 30/06/2025, fuso `America/Recife`, horas de 7h a 17h | `data_hora`, `temperatura_c`, `umidade_pct`, `nuvens_pct`, `vento_kmh`, `hora`, `radiacao_w_m2` |
| `meteo_treino_orange.csv` / `meteo_teste_orange.csv` | Gerados pelo notebook | Primeiras 80% (800 h) e últimas 20% (201 h) das horas | Mesmas colunas, para uso no Orange |

As duas consultas são públicas e **não exigem token**. Os dados da ANEEL registram empreendimentos em diferentes fases (não medem energia gerada) e as quantidades por classe refletem o limite da consulta, não a matriz energética brasileira. Os dados do Open-Meteo são estimativas de modelos/reanálise, não medições de um painel.

## Como executar

1. Abra `Aula_APIs_Energia_Renovavel_ML.ipynb` no Google Colab ou no Jupyter (Python 3.10+).
2. Se necessário, instale as dependências: `pip install pandas numpy matplotlib scikit-learn`.
3. Execute **Executar tudo** (*Run all*). As células consultam as APIs e salvam os CSVs; em seguida vêm a análise, o treinamento e a comparação dos seis modelos.
4. Se alguma API estiver indisponível, o notebook informa e baixa automaticamente a cópia publicada do CSV gerado pelo mesmo código, continuando a execução.

Como o cadastro da ANEEL é atualizado com frequência, uma execução em outra data pode trazer registros diferentes e valores de métricas ligeiramente diferentes dos registrados abaixo.

## Tarefa 1 — Classificação (ANEEL)

- **X:** `potencia_kw`, `latitude`, `longitude` · **y:** `fonte`.
- **Limpeza:** remoção de 47 linhas com coordenada (0, 0) → 3.829 empreendimentos.
- **Avaliação:** divisão **estratificada 80/20** com `random_state=42`; Precision, Recall e F1 com **média `macro`**; conferência por validação cruzada estratificada de 5 partes no treino.
- **Pré-processamento:** `log10(potencia_kw)` e `StandardScaler` dentro de um `Pipeline` (ajustado só no treino) para Regressão Logística e kNN.

| Modelo | Accuracy | Precision (macro) | Recall (macro) | F1 (macro) |
|---|---|---|---|---|
| **Random Forest** (300 árvores) | **0,977** | **0,977** | **0,976** | **0,977** |
| kNN (k = 5) | 0,965 | 0,965 | 0,964 | 0,965 |
| Regressão Logística | 0,832 | 0,847 | 0,828 | 0,825 |

**Conclusões:** o Random Forest foi o melhor modelo. A Regressão Logística confunde principalmente **Solar com Eólica** (63 solares previstas como eólicas), porque fronteiras lineares não separam regiões que se intercalam no mapa. Nos modelos não lineares, os poucos erros são sobretudo **solares previstas como hidráulicas**. O acerto alto é favorecido por empreendimentos de um mesmo complexo, com coordenadas quase iguais, presentes no treino e no teste (cerca de 32% dos casos de teste têm um vizinho a menos de ~1 km no treino) e pelo valor padrão de 1 kW em centenas de registros solares. Potência e localização sozinhas não definem a fonte; uma aplicação real precisaria de atributos físicos (irradiação, vento, hidrografia) e de validação espacial.

## Tarefa 2 — Regressão (Open-Meteo)

- **X:** `temperatura_c`, `umidade_pct`, `nuvens_pct`, `vento_kmh`, `hora` · **y:** `radiacao_w_m2` · `data_hora` usada apenas para ordenar e dividir.
- **Avaliação:** **divisão temporal**, com as primeiras 80% das horas para treino (01/04 7h a 12/06 14h) e as últimas 20% para teste (12/06 15h a 30/06 17h), sem embaralhar.

| Modelo | MAE (W/m²) | MSE ((W/m²)²) | R² |
|---|---|---|---|
| **Random Forest** (300 árvores) | **67,2** | **7.419,6** | **0,842** |
| Árvore de Decisão (profundidade 6) | 87,1 | 14.005,9 | 0,701 |
| Regressão Linear | 145,2 | 30.034,2 | 0,360 |

**Conclusões:** o Random Forest foi o melhor modelo. A **hora** é o atributo mais importante (representa a posição do Sol), mas sua relação com a radiação tem forma de sino, por isso a Regressão Linear não a aproveita e chega a prever valores negativos. Os maiores erros ocorrem perto do meio-dia. O período de teste (fim de junho, perto do solstício de inverno) tem radiação média menor que o treino, o que torna a avaliação temporal mais exigente e realista. **Radiação não é geração elétrica:** a energia de um sistema fotovoltaico depende ainda de área e potência instalada, eficiência, inclinação e orientação dos módulos, temperatura das células, sombreamento, sujeira e perdas elétricas.

## Estrutura do repositório

```
├── README.md
├── Aula_APIs_Energia_Renovavel_ML.ipynb
├── aneel_classificacao_orange.csv
├── meteo_regressao_orange.csv
├── meteo_treino_orange.csv
└── meteo_teste_orange.csv
```
