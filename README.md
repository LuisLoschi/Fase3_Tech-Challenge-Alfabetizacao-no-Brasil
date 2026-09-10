# Tech Challenge — Fase 3: Alfabetização no Brasil

![Data Science](https://img.shields.io/badge/Area-Data%20Science-blue)
![Python](https://img.shields.io/badge/Language-Python-blue)
![Static Badge](https://img.shields.io/badge/Status-done-green)

Grupo: Eduardo Rossi | Luis Loschi | Luiza Santos | Vitória Santos | Vyctor Correia

Projeto desenvolvido para o **Tech Challenge da Fase 3 da FIAP (IA Scientist)**, com o objetivo de construir uma pipeline de dados
utilizando soluções analíticas e modelos de Machine Learning capazes de gerar inteligência aplicada à tomada de decisão. 

**Resultados exploratórios e retrospectivos, com as limitações declaradas em texto e em código.**
O classificador estima risco contextual individual; a aplicação municipal entrega rankings,
perfis e cenários. Comece pelas [limitações do projeto](#limitações-do-projeto) e pelo
[guia de execução](GUIA_DE_EXECUCAO.md).

> **Status de validação, em uma linha:** a seleção supervisionada de atributos consultou a
> coorte inteira de 2024, então a reserva de teste **não é independente** dessa seleção e
> nenhum número aqui é confirmação prospectiva. O status está fixado em
> `src/evaluation/protocolo.py` e acompanha o modelo em `carregar_campeao()`.

## 📌 Sumário

- [Contexto do problema](#contexto-do-problema)
- [Objetivo analítico](#objetivo-analítico)
- [Descrição da base utilizada](#descrição-da-base-utilizada)
- [Etapas de modelagem](#etapas-de-modelagem)
- [Escolha do algoritmo](#escolha-do-algoritmo)
- [Métricas de avaliação](#métricas-de-avaliação)
- [Interpretação dos resultados](#interpretação-dos-resultados)
- [Insights encontrados](#insights-encontrados)
- [Aplicação prática para políticas públicas](#aplicação-prática-para-políticas-públicas)
- [Limitações do projeto](#limitações-do-projeto)
- [Possíveis evoluções futuras](#possíveis-evoluções-futuras)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Como reproduzir](#como-reproduzir)
- [Entregáveis](#entregáveis)
- [Entrega Executiva](#entrega-executiva)

<a id="contexto-do-problema"></a>
## 🔎 Contexto do problema

A alfabetização infantil é um dos principais indicadores de desenvolvimento educacional e social de uma nação. No Brasil, garantir que crianças estejam alfabetizadas na idade certa é um desafio complexo que envolve disparidades regionais, socioeconômicas e infraestruturais.

Compreender apenas os dados históricos passados não é suficiente para a tomada de decisão estratégica no setor público. Gestores educacionais necessitam antecipar riscos, identificar municípios vulneráveis e compreender quais fatores possuem maior impacto direto nos indicadores educacionais. A Ciência de Dados e a Inteligência Artificial desempenham um papel fundamental nesse cenário, transformando dados públicos brutos em inteligência analítica aplicável para formulação e ajuste de políticas públicas.

O projeto usa dados de 2023 e 2024 da Fase 2.
Predição e associação apoiam investigação; não identificam efeitos causais de políticas.

<a id="objetivo-analítico"></a>
## 🎯 Objetivo analítico

Desenvolver uma **pipeline completa de Machine Learning supervisionado** capaz de prever se um aluno (ou indicador municipal/escolar) será considerado **Alfabetizado** ou **Não Alfabetizado**, utilizando variáveis educacionais, territoriais, populacionais e socioeconômicas e bases públicas complementares.

### Perguntas de Negócio Orientadoras
- **Fatores Determinantes:** Quais variáveis socioeconômicas e infraestruturais têm maior peso na probabilidade de alfabetização?
- **Mapeamento de Risco:** Quais municípios ou regiões apresentam maior risco de descumprimento das metas educacionais?
- **Agrupamento Territorial:** Quais regiões possuem padrões educacionais e socioeconômicos semelhantes?
- **Suporte à Decisão:** Como utilizar as predições do modelo para alocação eficiente de recursos do FUNDEB e ações preventivas do Ministério da Educação e Secretarias Estaduais/Municipais?

<a id="descrição-da-base-utilizada"></a>
## 📝 Descrição da base utilizada

Sete fontes: 
- Gold em grão aluno; 
- microdado INEP; 
- agregados de município e UF;
- metas municipais, 
- estaduais e nacionais. 

Veja [origens e contagens](data/README.md) e [manifesto SHA-256](data/manifesto_fontes.json).

São 20 atributos: histórico educacional de 2023, proveniência e cadastro da coorte de 2024.
Não é correto dizer que todos são de 2023. Não foi integrada dimensão socioeconômica;
essa divergência do objetivo do enunciado permanece pendente de complemento ou alinhamento acadêmico.

<a id="etapas-de-modelagem"></a>
## 📑 Etapas de modelagem

CSV → Parquet → agregados históricos → dataset → comparação e tuning → classificador salvo
→ interpretação → produtos municipais. Código em `src/`, entrada/preparação em `scripts/`.

A Gold foi reconciliada com o microdado. Foram identificados IDs reciclados entre anos,
retirados registros sem medida válida e excluídos proficiência contemporânea, identificadores
e participação da própria prova dos preditores. Os pesos são usados nas agregações populacionais.

`ColumnTransformer` integra imputação, escala e encoding ao modelo. O ajuste usa apenas
o treino de cada fold. A reserva contém municípios diferentes dos do desenvolvimento.
**A seleção de atributos, porém, consultou toda a coorte:** as métricas não são uma
avaliação confirmatória independente. A ablação em `scripts/experimento_b5.py` separa a
reserva antes de qualquer seleção, o que preserva a partição para decisões futuras sem
tornar independente o resultado já publicado.

<a id="escolha-do-algoritmo"></a>
## 🛠 Escolha do algoritmo

Foram comparados Dummy, duas heurísticas, logística, Random Forest e LightGBM padrão/tunado. O LightGBM salvo foi mantido para preservar o artefato analisado e sua interpretação. Seu empate com Random Forest foi discutido em termos de custo; não se reivindica ganho comprovado do tuning. A isotônica não foi aplicada pelo critério de Brier no desenvolvimento.

Resultados históricos e comparações pareadas estão em [modelagem](reports/modelagem.md).

<a id="métricas-de-avaliação"></a>
## 📊 Métricas de avaliação

Reserva retrospectiva: 330836 alunos em 1104 municípios. Os intervalos abaixo vêm do bootstrap municipal registrado no artefato histórico; não corrigem a participação prévia da reserva na seleção.

| Métrica | Valor | IC95 registrado |
|---|---:|---|
| ROC-AUC | 0,6599 | [0,6314; 0,6853] |
| Average Precision (`pr_auc` no código) | 0,5360 | [0,5056; 0,5672] |
| Brier | 0,21794 | [0,21174; 0,22321] |

No limiar de capacidade definido no desenvolvimento: precisão 62,01%, recall 21,06%, F1 0,3144 e taxa de alerta 12,95% no teste. Matriz: VN 188.438, FP 16.270, FN 99.568, VP 26.560. São estimativas pontuais; não foram calculados ICs para esse limiar. Um corte escolhido para 20% do desenvolvimento não garante 20% em outra distribuição.

<a id="interpretação-dos-resultados"></a>
## 📝 Interpretação dos resultados

O modelo usa principalmente contexto municipal e de UF. SHAP e permutação em famílias consideram atributos correlacionados; coeficientes da logística fornecem uma leitura adicional. (Veja [interpretabilidade](reports/interpretabilidade.md)). 

O valor de histórico escolar verdadeiro não foi testado porque as chaves não permitem acompanhamento longitudinal.

<a id="insights-encontrados"></a>
## 🔖 Insights encontrados

- O vínculo longitudinal de aluno e escola por ID não é válido nesta base.
- O desempenho varia por cobertura do histórico; não se deve ocultar essa diferença numa métrica nacional.
- Ordenações por taxa e por volume respondem a perguntas diferentes sobre prioridade territorial.
- Os cenários de metas discriminam pouco; isso limita os métodos testados, sem provar imprevisibilidade geral.

<a id="aplicação-prática-para-políticas-públicas"></a>
## ✔ Aplicação prática para políticas públicas

As cinco perguntas são respondidas em [aplicação estratégica](reports/aplicacao_estrategica.md).

| Produto | Conteúdo e limite |
|---|---|
| [Ranking](reports/ranking_risco_municipal.csv) | Risco contextual e volume; posições sem incerteza quantificada. |
| [Perfis](reports/clusters_municipais.csv) | Agrupamentos descritivos de 2024, com sobreposição. |
| [Metas](reports/projecao_metas_municipios.csv) | Gap observado e cenários condicionais entre 0% e 100%. |

A projeção de metas foi avaliada em cinco folds municipais: ROC-AUC
0,5455 [0,5283; 0,5631].
Cobertura dos intervalos: 94,60%
[93,95%; 95,21%].
É avaliação territorial na mesma transição, não validação de ano futuro.
O ganho de Brier sobre prevalência do treino inclui zero no intervalo.

<a id="limitações-do-projeto"></a>
## ❗ Limitações do projeto

**Sobre a validade do que foi medido.** A seleção supervisionada de atributos consultou a coorte
inteira de 2024: a reserva de teste não é independente dessa seleção, e trocar a seed do split
não a torna independente. Não há validação temporal do classificador completo — features de lag
para 2023 exigiriam 2022, que não existe —, e a única checagem out-of-time possível usa um modelo
reduzido a UF, região e rede. Toda métrica publicada é retrospectiva e exploratória.

**Sobre o que a base não contém.** Não há nenhuma variável sobre a criança: nível socioeconômico,
cor/raça, idade, frequência ou histórico escolar. O modelo é territorial, e atribuir a uma criança
a característica do seu município é falácia ecológica. As chaves de escola são recicladas entre
edições, o que impede acompanhamento longitudinal e torna **não verificável** — não negativa — a
pergunta sobre o valor do histórico escolar. Roraima não está na base. O calendário de publicação
das fontes não foi auditado.

**Sobre quem entra na medição.** Só alunos presentes com medida válida. Ausência não é sinônimo de
não alfabetização, e a taxa municipal de um território com baixa presença é otimista; a ponderação
por `peso_aluno` corrige não resposta sob hipóteses que não são verificáveis aqui. O modelo regride
para a média e subestima o risco justamente onde ele é maior.

**Sobre os produtos municipais.** Não há intervalo de confiança em torno do risco de cada
município, nem validação temporal do ranking: com duas edições da prova não existe um 2025 contra
o qual conferir a ordenação de 2024. A incerteza das posições não foi quantificada em nenhum
produto. Os cenários de metas são **condicionais** à distribuição adotada — normal censurada em
0–100, uma decisão deste projeto e não uma garantia dada por biblioteca — e não garantem
resultados em 2025; a validação deles é territorial, na mesma transição 2023 → 2024, e não é teste
de ano futuro. O porte de referência da dispersão é descritivo e **não** é regra de elegibilidade:
não deve restringir acesso de município nenhum a política pública. Nada no projeto sustenta
conclusão causal.

<a id="possíveis-evoluções-futuras"></a>
## 🚀 Possíveis evoluções futuras

Três pontos estão abertos e dependem de coisas que não se resolvem escrevendo texto: 
- **Uma amostra ainda não consultada**, sem a qual não há confirmação prospectiva; 
- A **documentação da disponibilidade temporal** de cada fonte, que é o que autorizaria uso prospectivo; 
- A **integração de uma dimensão socioeconômica**, hoje ausente por falta de fonte integrada. Depois deles vêm medir a estabilidade do ranking e pactuar capacidade e custo com gestores. Novos algoritmos e tuning adicional não são a prioridade antes desses pontos — o ganho do modelo sobre uma regra de uma variável é de 0,027 de ROC-AUC, e o teto não está no algoritmo.

<a id="estrutura-do-repositório"></a>
## 📁 Estrutura do repositório

```text
├── data/                         # Fontes, referências e datasets
│
├── notebooks/                    # Análises exploratórias e experimentos
│
├── scripts/                      # Refinamento e execução do projeto
│   └── etl/                      # Tratamento inicial dos dados
│
├── src/
│   ├── data/                      # Loader de dados
│   ├── preprocessing/             # Limpeza, imputação e encoders
│   ├── modeling/                  # Treinamento dos modelos
│   ├── evaluation/                # Métricas e validação
│   └── visualization/             # Gráficos e relatórios visuais
│
├── reports/                       # Documentação, métricas e resultados
│   ├── metrics/                   # Datasets modelados
│   └── images/                    # Imagens dos relatórios
│
├── .gitignore                     # Arquivos ignorados pelo Git
├── GUIA_DE_EXECUCAO.md            # Documentação execução do projeto
├── README.md                      # Documentação principal
└── requirements.txt               # Dependências do projeto  
```

<a id="como-reproduzir"></a>
## ⚙ Como reproduzir

Use Python 3.14 e um ambiente virtual. No PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe scripts/verificar_insumos.py
.\.venv\Scripts\python.exe scripts/reproduzir.py --etapa dados
.\.venv\Scripts\python.exe scripts/reproduzir.py --etapa relatorios
.\.venv\Scripts\python.exe -m pytest -q
```

`--etapa relatorios` requer o classificador e as métricas de interpretação já gerados.
Para refazer tudo, use `--etapa tudo`: inclui treinos e experimentos demorados.
`TC_DATA_DIR` permite mudar a pasta de dados. Um clone requer os sete CSVs fornecidos pelo grupo;
o manifesto confere a versão, mas não substitui o fornecimento. Veja o guia para a sequência completa.

<a id="entregáveis"></a>
## Entregáveis

Código, notebooks, métricas, figuras, [documentação técnica](reports/documentacao_tecnica.md),
[limites de validade](reports/limites_de_validade.md) e [contrato temporal](reports/contrato_temporal.md).
A execução completa depende dos sete CSVs de origem, que não são versionados; o
[manifesto SHA-256](data/manifesto_fontes.json) confere a versão de cada um.

<a id="entrega-executiva"></a>
## 🎥 Entrega Executiva

Este projeto também inclui materiais voltados para stakeholders e lideranças, focados em tomada de decisão:

* **[Slides de Apresentação](apresentacao/PPT_Tech_Challenge_Fase_3.pptx):** Material visual com o contexto do problema, os principais insights, o valor estratégico da solução e como os modelos poderiam apoiar políticas públicas educacionais.