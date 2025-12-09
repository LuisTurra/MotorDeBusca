# 📊 Sistema de Análise de Dados e Recomendação (MySQL + NLP)

Este projeto implementa um sistema de pesquisa avançada e recomendação, integrando múltiplos bancos de dados MySQL com técnicas de Processamento de Linguagem Natural (NLP) e Regras de Associação (Apriori) em Python.

---

## 🛠️ Requisitos e Configuração

Para rodar este projeto, é necessário ter o Python instalado e as seguintes bibliotecas.

### 📦 Bibliotecas Python

As seguintes bibliotecas foram instaladas (conforme o output do `pip install`):

* **Conexão e Dados:** `mysql-connector-python`, `pandas`, `numpy`, `re`, `nltk`, `datetime`.
* **NLP e Text Mining:** `spacy`, `textblob`, `pyspellchecker` (`spellchecker`), `langdetect`.
* **Regras de Associação:** `apyori`.
* **Download de Modelos NLP:** `python -m spacy download en_core_web_sm` (necessário para a função `ner`).

### ⚙️ Conexões com Bancos de Dados (MySQL)

As funções de conexão (`conectaBD`, `worldBD`, `historicoBD`, `tvBD`) esperam as seguintes configurações de banco de dados (BD) local:

| Função | BD (Schema) | Finalidade Principal |
| :--- | :--- | :--- |
| `conectaBD` | `mysq` (Sakila) | Dados de Locadora (Filmes, Atores, Aluguéis) |
| `worldBD` | `world_x` | Dados Geográficos (Cidades, Países) |
| `historicoBD` | `historico` | Armazenamento de Pesquisas dos Usuários |
| `tvBD` | `tv` | Especificações e Preços de TVs |

*Nota: As credenciais de usuário (`root`) e senhas (`***`) e portas (`***`) devem ser configuradas no script para a execução local.*

---

## 🔬 Lógica do Processamento de Texto (NLP)

O coração do sistema é o pré-processamento da *query* de pesquisa para garantir precisão e sugerir termos relacionados.

### 1. Pré-processamento e Lematização (`filtered(search)`)

* **Lematização (`textblob`):** Reduz palavras flexionadas à sua forma raiz (ex: 'running' -> 'run', 'movies' -> 'movie').
* **Remoção de *Stop Words* (`spacy.lang.en.English`):** Elimina palavras comuns e conectores (ex: 'the', 'is', 'de', 'para') para isolar as palavras-chave relevantes.

### 2. Classificação de Entidades Nomeadas (NER) (`ner(search)`)

* Utiliza o modelo `en_core_web_sm` do spaCy para classificar as palavras-chave restantes.
    * **Identificação:** Detecta se a pesquisa contém `GPE` (local/país), `CARDINAL` (número), o que direciona a estratégia de busca (tabelas geográficas, filmes por ano/duração, TVs por tamanho, etc.).

### 3. Histórico e Correção Ortográfica

* **Registro da Pesquisa (`historico(search)`):** Salva a *query* original e o *timestamp* no BD `historico`.
* **Verificação de Texto (`verificadorTexto(search)`):**
    * Detecta o idioma (`langdetect`).
    * Sugere correções ortográficas (`SpellChecker`) com base no idioma detectado (Português, Inglês ou Espanhol).

---

## 📈 Sistema de Recomendação

O sistema utiliza o algoritmo Apriori para analisar o histórico de pesquisas e sugerir termos que tendem a aparecer juntos.

### Regras de Associação (`pesquisaRelacionadas(search)`)

1.  Lê o histórico de pesquisas (BD `historico`).
2.  Aplica a função `filtered` em todas as pesquisas passadas.
3.  Executa o algoritmo **Apriori** (`apyori`) para encontrar regras de associação entre as palavras lematizadas.
4.  Filtra os 10 resultados com maior *Support* (frequência) que contenham a palavra-chave pesquisada.

| Parâmetro Apriori | Valor | Significado |
| :--- | :--- | :--- |
| `min_support` | 0.003 | Frequência mínima de ocorrência dos itens. |
| `min_confidence` | 0.2 | Probabilidade mínima da regra ser válida. |
| `min_lift` | 3 | Grau de interesse da regra (acima de 1 indica associação forte). |

---

## 🔎 Estratégia de Pesquisa (`pesquisa(search)`)

A função principal processa as palavras-chave e as classificações de NER para montar a consulta SQL ideal:

1.  **Filtragem de Termos:** Identifica e remove termos de controle como 'movie', 'tv', 'preço', 'menor', 'maior', 'tamanho' para deixar apenas as palavras-chave de busca.
2.  **Busca de Filmes (`veriMovie == True`):**
    * Realiza `JOIN`s complexos (ex: `filme`, `idioma`, `filme_ator`, `ator`, `filme_categoria`, `categoria`).
    * Pesquisa nas colunas de texto (`titulo`, `classificacao`, `primeiro_nome`, `ultimo_nome`) usando `LIKE '%{}%'`.
    * Se for pesquisado por categoria (ex: 'action movie'), filtra a tabela `categoria`.
3.  **Busca de TVs (`veriTV == True`):**
    * Busca na tabela `tvs`.
    * Aplica cláusulas `ORDER BY` e `LIMIT 1` para encontrar o menor ou maior preço/tamanho, se as palavras de controle (ex: 'menor preco') forem detectadas.
4.  **Agregação:** Agrega os resultados de todas as consultas SQL em um único DataFrame Pandas.
