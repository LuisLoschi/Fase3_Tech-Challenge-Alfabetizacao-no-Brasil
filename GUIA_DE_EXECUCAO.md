# Guia de execução

Este guia mostra como reproduzir o projeto, em que ordem os passos rodam e o que cada parte faz.

O [README](README.md) apresenta a proposta e os resultados, aqui está a execução prática.

## 📌 Sumário

- [Comece aqui](#comece-aqui)
- [Pré-requisitos](#pré-requisitos)
- [Instalação](#instalação)
- [Pipeline](#pipeline)
- [Outputs](#Outputs)
- [Notebooks](#os-notebooks)
- [Testes](#os-testes)
- [Problemas comuns](#problemas-comuns)

<a id="comece-aqui"></a>
## 🚀 Comece aqui

Escolha o caminho conforme o tempo e o objetivo:

- **10 minutos:** leia o [README](README.md) até as limitações e vá para
  [`reports/aplicacao_estrategica.md`](reports/aplicacao_estrategica.md). Não é preciso rodar nada.
- **1 hora:** siga os notebooks de 01 a 05 e depois veja [`reports/documentacao_tecnica.md`](reports/documentacao_tecnica.md).
- **Executar o projeto:** siga este guia a partir da próxima seção. Reserve cerca de duas horas, e use [`scripts/reproduzir.py`](scripts/reproduzir.py) para a sequência completa.

<a id="pré-requisitos"></a>
## ✅ Pré-requisitos

| Item | Versão | Observação |
|---|---|---|
| Python | 3.14.3 | versão validada para `lightgbm`, `shap` e `numba` |
| Memória | 16 GB | os dados têm 1,85 milhão de linhas e o Parquet é lido em memória |
| Núcleos | >2 | ajuda bastante na comparação de modelos |
| Databricks | - | necessário para gerar a camada Gold, que não está versionada |

Para rodar o projeto, você precisa das sete fontes de dados originais e da camada Gold **gerada em Databricks**. O código implementado no Databricks está em [`scripts/etl/`](scripts/etl), mas regenerá-la exige acesso a um workspace. Clonar o repositório não traz esse arquivo, e o manifesto confere a versão sem substituir o fornecimento. Os links do Drive para a Gold e para o microdado INEP estão em [`data/README.md`](data/README.md#origem-de-cada-base), na seção "Origem de cada base", rodar `python scripts/prepare_data.py --load_data` baixa os dois direto para `data/raw/`.

 O código em [`scripts/etl/`](scripts/etl) gera essa base, mas não substitui o acesso ao workspace. Os links para as fontes e para o microdado INEP estão em [`data/README.md`](data/README.md#origem-de-cada-base).

Os seis CSVs do INEP são públicos e ficam em `data/raw/inep/`. Antes de qualquer execução, confira a integridade das fontes com: `python scripts/verificar_insumos.py`

Esse passo calcula o SHA-256 de cada arquivo e compara com o [`data/manifesto_fontes.json`](data/manifesto_fontes.json). Se houver diferença, a execução para para evitar misturar versões incompatíveis de dados. Se os arquivos estiverem fora da pasta `data/`, use `TC_DATA_DIR` para apontar a localização correta.

<a id="instalação"></a>
## 🛠️ Instalação

```bash
python -m venv .venv
source .venv/Scripts/activate            # PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
python -m ipykernel install --user --name tc-fase3
```

Esse passo cria o ambiente virtual, instala as dependências e registra o kernel `tc-fase3` para os notebooks. Sem esse kernel, o Jupyter pode abrir no Python do sistema e falhar ao importar `lightgbm` e `shap`.

Se quiser rodar novamente os notebooks sem instalar outro kernel, use:

```bash
python scripts/executar_notebooks.py notebooks/05_aplicacao_estrategica.ipynb
```

Para confirmar que a instalação está correta, rode:

```bash
python -c "import sys, lightgbm, shap; print(sys.prefix)"
pytest -q -k dados
```

<a id="pipeline"></a>
## 🔄 Pipeline

A ordem importa: cada linha consome o que a anterior escreveu.

| # | Comando | O que faz | Tempo* |
|---|---|---|---|
| 1 | `python scripts/verificar_insumos.py` | SHA-256 das sete fontes contra o manifesto | segundos |
| 2 | `python scripts/prepare_data.py` | CSV → Parquet, por ano | 9 s |
| 3 | `python scripts/build_dim_municipio.py` | dimensão `município → UF/região` | segundos |
| 4 | `python -m src.preprocessing.build_dataset` | coorte 2024 + atributos de 2023 e cadastro de 2024 | não cronometrado |
| 5 | `python scripts/experimento_b5.py` | ablação de atributos, só no desenvolvimento | ~35 min em 16 núcleos |
| 6 | `python -m src.modeling.train comparacao` | 7 candidatos nos mesmos folds | ~13 min em 16 núcleos |
| 7 | `python -m src.modeling.train tuning` | 40 configurações de LightGBM | **> 45 min** |
| 8 | `python -m src.modeling.train campeao` | refit, reserva, serialização | 189 s |
| 9 | `python -m src.evaluation.rigor` | as cinco provas de robustez | não cronometrado |
| 10 | `python -m src.evaluation.interpret` | SHAP, permutação, famílias | 618 s |
| 11 | `python -m src.modeling.strategic` | ranking, clusters, metas | 131 s |
| — | `python scripts/executar_notebooks.py notebooks/*.ipynb` | regrava as saídas e as 55 figuras | não cronometrado |
| — | `pytest -q` | a suíte inteira | 42 s |

*Os tempos podem variar

**O orquestrador.** [`scripts/reproduzir.py`](scripts/reproduzir.py) executa esses comandos em três
blocos, com `PYTHONHASHSEED=42` e `MPLBACKEND=Agg` fixados, e para na primeira falha:

```bash
python scripts/reproduzir.py --etapa dados        # passos 1 a 4
python scripts/reproduzir.py --etapa modelos      # passos 5 a 10
python scripts/reproduzir.py --etapa relatorios   # passos 11
python scripts/reproduzir.py --etapa tudo         # todos de uma unica vez
```

`--etapa relatorios` é o padrão porque é o bloco que se roda com frequência: ele supõe o
classificador e as métricas de interpretação já em disco. 

`--etapa modelos` é o que custa mais de
uma hora e meia.

**Atalho legítimo.** Os passos 5, 6 e 7 produzem evidência comparativa e hiperparâmetros, e juntos
consomem quase todo o tempo do pipeline. Os CSVs que eles geram já estão versionados em
`reports/metrics/`, e o passo 8 lê os hiperparâmetros do JSON, não da busca. Para reproduzir só o
modelo e os produtos finais, rode 1, 2, 3, 4, 8, 9, 10, 11.

<a id="Outputs"></a>
## 🎯 Outputs

Estes são os arquivos que um gestor usaria. Cada um traz as colunas de ressalva ao lado das de
resultado, de propósito.

| Arquivo | Linhas | Para que serve |
|---|---:|---|
| [`reports/ranking_risco_municipal.csv`](reports/ranking_risco_municipal.csv) | 5.517 | duas ordenações de risco: por intensidade e por volume de alunos |
| [`reports/clusters_municipais.csv`](reports/clusters_municipais.csv) | 5.461 | os três perfis municipais, para política por perfil em vez de por UF |
| [`reports/projecao_metas_municipios.csv`](reports/projecao_metas_municipios.csv) | 5.352 | gap de esforço até a meta de 2025 e o cenário condicional para ela |

Colunas de ressalva que acompanham o resultado, e o que cada uma quer dizer:

| Coluna | Onde | O que sinaliza |
|---|---|---|
| `modelo_avaliado` | ranking | `False` nos 23 municípios de AC e DF, onde o escore existe e a posição não; o restante do ranking os ignora |
| `faixa_shap_taxa_municipal` | ranking | recorte descritivo do achatamento do SHAP; **não** é um julgamento da ordenação inteira |
| `incerteza_posicao_quantificada` | ranking | é `False` em todas as linhas: a estabilidade das posições não foi medida |
| `natureza_resultado` | projeção | `cenario_condicional_sem_validacao_em_ano_futuro`, escrito por extenso na própria linha |
| `meta_dentro_do_intervalo` | projeção | a meta do município cai dentro do intervalo de 95% |
| `porte_abaixo_referencia_dispersao` | projeção | porte abaixo da referência usada para modelar dispersão; é descrição, não regra de elegibilidade |
| `taxa_presenca_2024` | ambos | a cobertura sobre a qual o número foi calculado |

`faixa_shap_taxa_municipal` indica apenas onde a contribuição SHAP da taxa municipal se achata.
Isso não significa que o ranking completo do modelo seja frágil nessa faixa, pois os demais
atributos continuam contribuindo.

<a id="os-notebooks"></a>
## 📓 Notebooks

Leia na ordem. Cada um abre com a tabela de perguntas que ele fecha e numera os achados.

| Notebook | Fase CRISP-DM | O que decide |
|---|---|---|
| [`01_eda.ipynb`](notebooks/01_eda.ipynb) | 2, *Data Understanding* | identifica os riscos da base, o sinal municipal e as regras para evitar vazamento |
| [`02_feature_engineering.ipynb`](notebooks/02_feature_engineering.ipynb) | 3, *Data Preparation* | fecha o dataset com 20 atributos válidos e remove blocos redundantes ou pouco confiáveis |
| [`03_modelagem.ipynb`](notebooks/03_modelagem.ipynb) | 4, *Modeling* | escolhe o modelo operacional, define os limiares e mede seus limites de generalização |
| [`04_interpretabilidade.ipynb`](notebooks/04_interpretabilidade.ipynb) | 5, *Evaluation* | explica a importância das variáveis por família e compara SHAP, permutação e logística |
| [`05_aplicacao_estrategica.ipynb`](notebooks/05_aplicacao_estrategica.ipynb) | 5 → 6, *Deployment* | transforma os resultados em rankings, perfis e cenários municipais, com ressalvas de uso |

**Todos os cinco já estão executados**, sem erro e com as saídas gravadas. Para reexecutar,
rode na ordem:

```bash
python scripts/executar_notebooks.py 
notebooks/01_eda.ipynb 
notebooks/02_feature_engineering.ipynb     
notebooks/03_modelagem.ipynb 
notebooks/04_interpretabilidade.ipynb     
notebooks/05_aplicacao_estrategica.ipynb
```

O notebook 01 é o mais demorado, porque é o único que lê as bases inteiras em vez de artefatos.
Abrir no Jupyter continua funcionando; aí sim é preciso o kernel `tc-fase3`.

<a id="os-testes"></a>
## 🧪 Testes

Comandos:
```bash
pytest -q                       # 95 testes, cerca de 42 s
pytest -q tests/test_dados.py   # só a integridade das bases
```

Eles não testam se o código roda. Testam se as decisões continuam válidas.

| Arquivo | Casos | O que trava |
|---|---:|---|
| `test_dados.py` | 24 | contagens de referência, códigos de rede, e que `id_aluno` não é chave longitudinal |
| `test_features.py` | 17 | interseção vazia com `COLS_PROIBIDAS`, cobertura do lag, VIF do subconjunto podado |
| `test_modelagem.py` | 10 | que o artefato em disco reproduz o ROC-AUC publicado, conferido contra o hash do dataset real |
| `test_interpretabilidade.py` | 12 | que o SHAP soma exatamente a predição, e o mapa de 89 colunas para 5 famílias |
| `test_estrategia.py` | 22 | que o escore é out-of-fold, e que o CSV de metas não contém rótulo binário |
| `test_protocolo.py` | 10 | reserva separada antes da seleção, projeção dentro de 0–100, e o carregador anexando o limite da validação |

<a id="problemas-comuns"></a>
## 🛠️ Problemas comuns

**`ModuleNotFoundError: lightgbm` no notebook:** selecione o kernel `tc-fase3`, criado com
`python -m ipykernel install --user --name tc-fase3`.

**Erro de hash nos passos 7 ou 9:** o dataset ou o modelo não correspondem à mesma execução.
Refaça o pipeline a partir do passo 3.

**Falha nos testes de contagem:** alguma fonte está ausente ou veio de outra extração. Rode
`python scripts/verificar_insumos.py` e consulte as contagens esperadas em
[`data/README.md`](data/README.md).

**Fontes diferentes do manifesto:** não use `--gravar-manifesto` para contornar o erro, pois isso
desatualiza as métricas publicadas. Obtenha a extração correta dos dados com o nosso grupo.

**O notebook falha no meio, ou o kernel não sobe:** Use
`python scripts/executar_notebooks.py <caminho>` em vez de rodar pelo Jupyter. Ele usa o Python
corrente, sem depender de um kernel registrado com `--user`, e para na primeira célula com erro em
vez de gravar um notebook meio executado.

**O notebook rodou sem erro e saiu sem nenhuma figura:** É `MPLBACKEND=Agg` vazando para o kernel.
As células chamam `salvar_figura` e imprimem o caminho, então quem exibe a figura no notebook é a
exibição automática do backend inline — com `Agg` ela não acontece, o PNG em `images/` sai igual e
o `.ipynb` fica mudo. `executar_notebooks.py` fixa o backend inline justamente por isso; não
sobrescreva `MPLBACKEND` ao chamá-lo.

**`FileNotFoundError` em `dim_municipio.csv`:** execute o passo 2 antes das etapas seguintes.

**Busca de hiperparâmetros sem progresso:** ela leva mais de 45 minutos e grava o resultado apenas
ao final. Se necessário, use o atalho do pipeline e pule os passos 4 e 5.