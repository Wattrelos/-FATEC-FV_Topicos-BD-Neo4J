## 👨‍🏫 Metadados e Informações Acadêmicas
- **Instituição:** Faculdade de Tecnologia de Ferraz de Vasconcelos (FATEC-FV)
- **Curso:** Análise e Desenvolvimento de Sistemas (ADS)
- **Disciplina:** Tópicos em Banco de Dados e Big Data
- **Professor Orientador:** Prof. Francisco Abreu
- **Pesquisador / Desenvolvedor:** Josias Sobrinho
- **Projeto Base:** Atividade prática com Neo4j e Bancos de Dados Orientados a Grafos

# 🚀 Atividade Prática: Introdução ao Neo4j com Docker

> [!IMPORTANT]
> **Ambiente de Execução:** Os comandos deste roteiro foram elaborados e testados em ambiente Linux. Caso utilize Windows ou macOS, os comandos do Docker e as consultas na linguagem Cypher permanecem idênticos, com eventuais adaptações apenas nos comandos de terminal (PowerShell / WSL2).

---

## 🎯 Objetivos da Atividade
1. Provisionar uma instância local do **Neo4j** utilizando **Docker Compose**.
2. Compreender a arquitetura do **Property Graph Model** (Nós, Rótulos, Relacionamentos, Direções e Propriedades).
3. Conectar e interagir com o banco via **Neo4j Browser** (Web UI) e via linha de comando com o **cypher-shell**.
4. Praticar o ciclo de vida dos dados (CRUD) utilizando a linguagem declarativa **Cypher** (`CREATE`, `MATCH`, `SET`, `DELETE`, `REMOVE`).
5. Explorar o diferencial dos grafos: travessias de múltiplos saltos (*traversals*), recomendação (*Friends of Friends*), menor caminho (*Shortest Path*) e a cláusula idempotente `MERGE`.
6. Criar restrições de integridade (*Constraints*) e índices de performance.

---

## 📋 1. Configuração do Docker Compose

O **Neo4j** é o banco de dados NoSQL orientado a grafos mais utilizado no mercado. Diferente dos bancos relacionais, que simulam conexões por meio de chaves estrangeiras e operações custosas de `JOIN`, o Neo4j implementa o conceito de **Adjacência Livre de Índices** (*Index-Free Adjacency*): cada nó armazena ponteiros diretos em disco e memória para os nós com os quais se relaciona.

No diretório do projeto, o arquivo `docker-compose.yml` provisiona o container com persistência de dados e o conjunto de extensões **APOC**:

```yaml
services:
  neo4j:
    image: neo4j:latest
    container_name: meu_neo4j
    ports:
      - "7474:7474" # Porta HTTP para a interface web (Neo4j Browser)
      - "7687:7687" # Porta nativa do protocolo Bolt (drivers de aplicação e cypher-shell)
    volumes:
      - neo4j_data:/data                 # Dados persistidos do grafo (bancos de dados)
      - neo4j_logs:/logs                 # Registros e logs de execução
      - neo4j_import:/var/lib/neo4j/import # Diretório padrão para importação de CSVs
      - neo4j_plugins:/plugins           # Plugins e extensões
    environment:
      - NEO4J_AUTH=neo4j/senha_segura_123 # Credenciais padrão (usuário/senha)
      - NEO4J_PLUGINS=["apoc"]            # Ativa os procedimentos avançados da biblioteca APOC

volumes:
  neo4j_data:
  neo4j_logs:
  neo4j_import:
  neo4j_plugins:
```

> [!NOTE]
> **Explicação das Portas e Diretivas:**
> - **Porta `7474` (HTTP)**: Interface gráfica interativa (Neo4j Browser) para visualização e consulta gráfica.
> - **Porta `7687` (Bolt)**: Protocolo binário de alta performance sobre TCP utilizado por drivers (Python, Java, Node.js, C#) e pelo CLI `cypher-shell`.
> - **Volume `neo4j_data`**: Garante a integridade e durabilidade dos nós e arestas caso o container seja recriado.
> - **Variável `NEO4J_PLUGINS=["apoc"]`**: Faz o download e instalação automática do pacote *Awesome Procedures on Cypher* (funções extras de grafos, importação de dados e utilitários).

---

## 🐳 2. Execução e Acesso ao Ambiente

### 2.1. Inicializar o serviço em segundo plano

No terminal, dentro da pasta onde reside o `docker-compose.yml`, execute:

```bash
docker compose up -d
```

### 2.2. Acompanhar os logs de inicialização

O Neo4j inicializa seus subsistemas em alguns segundos. Você pode verificar o progresso com:

```bash
docker compose logs -f
```

Assim que a mensagem `Bolt enabled on 0.0.0.0:7687` e `Started.` aparecerem, o banco estará pronto para receber conexões. Pressione `Ctrl + C` para sair dos logs.

### 2.3. Formas de Conexão

Você possui duas formas principais de interagir com o Neo4j:

#### Opção A: Interface Web (Neo4j Browser) Recomendada para visualização
1. Abra o navegador e acesse: [http://localhost:7474](http://localhost:7474)
2. Preencha os dados de autenticação configurados no compose:
   - **Connect URL:** `neo4j://localhost:7687` (ou `bolt://localhost:7687`)
   - **Authentication type:** `Username / Password`
   - **Username:** `neo4j`
   - **Password:** `senha_segura_123`
3. Clique em **Connect**.

#### Opção B: Terminal Interativo (`cypher-shell`) Ideal para scripts e CLI
Caso deseje executar comandos diretamente pelo terminal do host sem abrir o navegador:

```bash
docker exec -it meu_neo4j cypher-shell -u neo4j -p senha_segura_123
```

---

## 🧠 3. Conceitos Fundamentais: Property Graph Model

O modelo de dados do Neo4j é fundamentado no padrão **Property Graph**, composto por três pilares essenciais:

```mermaid
graph LR
    A["(:Pessoa {nome: 'Alice', idade: 30})"] -->|"[:TRABALHA_NA {cargo: 'Dev'}]"| B["(:Empresa {nome: 'Google'})"]
```

1. **Nós (Nodes):** Representam as entidades do domínio (ex: uma `Pessoa`, um `Curso`, um `Produto`). Nós podem possuir zero, um ou múltiplos **Rótulos (Labels)** (ex: `:Pessoa:Usuario`).
2. **Relacionamentos (Relationships):** Conectam pares de nós com significado semântico direto (ex: `-[:TRABALHA_NA]->`). Todo relacionamento possui obrigatoriamente um **tipo**, uma **direção** (origem $\rightarrow$ destino) e pode conter **propriedades**.
3. **Propriedades (Properties):** Pares chave-valor associados tanto a nós quanto a relacionamentos (ex: `nome: "Alice"`, `desde: 2022`).

### 📊 Comparativo: Relacional (SQL) vs. Grafos (Neo4j)

| Aspecto | Bancos Relacionais (MySQL, PostgreSQL) | Neo4j (Orientado a Grafos) |
| :--- | :--- | :--- |
| **Estrutura Básica** | Tabelas com linhas e colunas fixas. | Nós, Rótulos, Relacionamentos e Propriedades. |
| **Conexões** | Chaves Estrangeiras (*Foreign Keys*) e tabelas de junção (*Join Tables*). | Relacionamentos nativos de primeira classe com ponteiros em disco. |
| **Custo de Junção** | $O(N \log M)$ no pior caso; JOINs degradam severamente à medida que o volume cresce. | $O(1)$ por travessia; navega apenas pelas arestas locais do nó (*Index-Free Adjacency*). |
| **Esquema** | Rígido (*Schema-on-write*); migrações de DDL necessárias. | Flexível (*Schema-optional*); evolução ágil de modelos de dados. |
| **Linguagem** | SQL (*Structured Query Language*). | Cypher (baseada em *ASCII Art*: nós em parênteses e arestas com setas). |

---

## 📝 4. Ciclo de Vida dos Dados com Cypher (CRUD)

A linguagem **Cypher** utiliza uma sintaxe declarativa visual intuitiva:
- Nós são delimitados por parênteses: `(p:Pessoa)`
- Relacionamentos são delimitados por colchetes e setas: `-[r:TRABALHA_NA]->`

> [!TIP]
> Execute os blocos abaixo na linha de comando do **Neo4j Browser** (topo da tela) ou dentro da sessão do **cypher-shell**.

---

### 4.1. Criar Dados (`CREATE`)

#### A. Criar um nó isolado com propriedades
Cria uma entidade do tipo `Pessoa` contendo propriedades primitivas:

```cypher
CREATE (p:Pessoa {nome: "Alice", idade: 30, cidade: "Ferraz de Vasconcelos"})
RETURN p;
```

#### B. Criar múltiplos nós com labels compostas
Um nó pode pertencer a mais de uma categoria ao mesmo tempo:

```cypher
CREATE (b:Pessoa:Estudante {nome: "Bob", idade: 22, matricula: 1001})
CREATE (c:Pessoa:Professor {nome: "Carlos", idade: 45, especialidade: "Grafos"})
RETURN b, c;
```

#### C. Criar nós e relacionamento simultaneamente
Conecta a pessoa à empresa, adicionando atributos descritivos na própria aresta:

```cypher
CREATE (p:Pessoa {nome: "Diana", idade: 28})-[:TRABALHA_NA {cargo: "Engenheira de Dados", desde: 2023}]->(e:Empresa {nome: "Google", setor: "Tecnologia"})
RETURN p, e;
```

---

### 4.2. Buscar e Filtrar Dados (`MATCH` / `RETURN` / `WHERE`)

#### A. Buscar todos os nós de um rótulo
Lista todas as pessoas cadastradas limitando o retorno:

```cypher
MATCH (p:Pessoa)
RETURN p
LIMIT 10;
```

#### B. Busca com filtro em linha (*Inline Pattern*)
Filtro direto na definição do nó:

```cypher
MATCH (p:Pessoa {nome: "Alice"})
RETURN p;
```

#### C. Busca avançada utilizando cláusula `WHERE`
Permite comparações matemáticas, lógicas e expressões regulares:

```cypher
MATCH (p:Pessoa)
WHERE p.idade >= 25 AND (p.cidade = "Ferraz de Vasconcelos" OR p.nome STARTS WITH "D")
RETURN p.nome AS Nome, p.idade AS Idade, p.cidade AS Cidade;
```

#### D. Navegação de conexões e projeção de colunas
Recupera quem trabalha em qual empresa e o respectivo cargo:

```cypher
MATCH (p:Pessoa)-[r:TRABALHA_NA]->(e:Empresa)
RETURN p.nome AS Funcionario, r.cargo AS Cargo, r.desde AS Desde, e.nome AS Empresa;
```

---

### 4.3. Atualizar Dados (`SET`)

#### A. Adicionar ou alterar propriedades de um nó
Localiza o nó pela chave e adiciona/modifica campos dinamicamente:

```cypher
MATCH (p:Pessoa {nome: "Alice"})
SET p.email = "alice@fatec.sp.gov.br",
    p.cidade = "São Paulo"
RETURN p;
```

#### B. Adicionar novo rótulo (Label) a um nó existente
Promove a Alice adicionando a etiqueta `Alumni`:

```cypher
MATCH (p:Pessoa {nome: "Alice"})
SET p:Alumni
RETURN p;
```

#### C. Atualizar propriedades em um relacionamento
Modifica os metadados de uma aresta:

```cypher
MATCH (p:Pessoa {nome: "Diana"})-[r:TRABALHA_NA]->(e:Empresa {nome: "Google"})
SET r.cargo = "Líder Técnica", r.salario_faixa = "Senior"
RETURN p, r, e;
```

---

### 4.4. Deletar Dados (`DELETE` / `REMOVE`)

#### A. Remover apenas uma propriedade ou rótulo (`REMOVE`)
O comando `REMOVE` apaga atributos ou etiquetas sem destruir o nó:

```cypher
-- Remove apenas o atributo idade
MATCH (p:Pessoa {nome: "Alice"})
REMOVE p.idade
RETURN p;

-- Remove a etiqueta Alumni
MATCH (p:Pessoa {nome: "Alice"})
REMOVE p:Alumni
RETURN p;
```

#### B. Deletar apenas o relacionamento entre dois nós
Desconecta a Diana da empresa Google, mantendo os nós de ambos intactos:

```cypher
MATCH (p:Pessoa {nome: "Diana"})-[r:TRABALHA_NA]->(e:Empresa {nome: "Google"})
DELETE r;
```

#### C. Deletar um nó isolado
O Neo4j impede a exclusão de nós que ainda possuam conexões ativas para garantir a integridade referencial do grafo:

```cypher
MATCH (p:Pessoa {nome: "Bob"})
DELETE p;
```

> [!WARNING]
> Se você tentar rodar `DELETE` em um nó que ainda possui arestas conectadas, receberá o erro:
> `Cannot delete node <id> because it still has relationships. To delete this node, you must first delete its relationships or use DETACH DELETE.`

#### D. Deletar nó em cascata com suas arestas (`DETACH DELETE`)
Remove o nó e todas as arestas conectadas a ele em uma única operação atômica:

```cypher
MATCH (p:Pessoa {nome: "Alice"})
DETACH DELETE p;
```

---

## ⚡ 5. Operações Avançadas e o Poder dos Grafos

### 5.1. Operação Idempotente com `MERGE` (Evitando Duplicações)

O comando `CREATE` sempre cria novos registros, mesmo que já exista outro nó idêntico. O comando `MERGE` atua como um **Get-or-Create / Upsert**: busca o padrão; se não existir, cria; se existir, reaproveita.

```cypher
MERGE (e:Empresa {nome: "Microsoft"})
ON CREATE SET e.criado_em = datetime(), e.origem = "Novo Registro"
ON MATCH SET e.ultimo_acesso = datetime()
RETURN e;
```

Conectar duas entidades de forma idempotente sem duplicar relacionamento:

```cypher
MERGE (p:Pessoa {nome: "Carlos"})
MERGE (inst:Instituicao {nome: "FATEC Ferraz de Vasconcelos"})
MERGE (p)-[r:LECIONA_EM]->(inst)
ON CREATE SET r.desde = 2020
RETURN p, r, inst;
```

---

### 5.2. Povoamento da Rede de Demonstração

Para os próximos testes de travessia e recomendação, vamos gerar uma pequena rede interconectada de tecnologia e estudos:

```cypher
// Criando alunos e interesses
MERGE (lucas:Aluno {nome: "Lucas"})
MERGE (mariana:Aluno {nome: "Mariana"})
MERGE (gabriel:Aluno {nome: "Gabriel"})
MERGE (beatriz:Aluno {nome: "Beatriz"})
MERGE (tiago:Aluno {nome: "Tiago"})

MERGE (neo:Tecnologia {nome: "Neo4j"})
MERGE (python:Tecnologia {nome: "Python"})
MERGE (docker:Tecnologia {nome: "Docker"})

// Relacionamentos de amizade
MERGE (lucas)-[:AMIGO_DE]->(mariana)
MERGE (mariana)-[:AMIGO_DE]->(gabriel)
MERGE (gabriel)-[:AMIGO_DE]->(beatriz)
MERGE (beatriz)-[:AMIGO_DE]->(tiago)

// Interesses em tecnologias
MERGE (lucas)-[:ESTUDA]->(neo)
MERGE (mariana)-[:ESTUDA]->(neo)
MERGE (mariana)-[:ESTUDA]->(python)
MERGE (gabriel)-[:ESTUDA]->(docker)
MERGE (beatriz)-[:ESTUDA]->(python);
```

---

### 5.3. Recomendação de Amigos ("Amigos de Amigos" - 2 Saltos)

Como sugerir novas amizades para o **Lucas** baseado nas conexões existentes?
Procuramos pessoas a 2 saltos de distância com quem Lucas ainda não possua conexão direta:

```cypher
MATCH (eu:Aluno {nome: "Lucas"})-[:AMIGO_DE]->(amigo:Aluno)-[:AMIGO_DE]->(sugestao:Aluno)
WHERE NOT (eu)-[:AMIGO_DE]->(sugestao) AND eu <> sugestao
RETURN sugestao.nome AS AmigoSugerido, amigo.nome AS AmigoEmComum;
```

---

### 5.4. Algoritmo de Menor Caminho (`shortestPath`)

Descubra a menor cadeia de conexões que liga **Lucas** até **Tiago**:

```cypher
MATCH (origem:Aluno {nome: "Lucas"}), (destino:Aluno {nome: "Tiago"})
MATCH caminho = shortestPath((origem)-[:AMIGO_DE*]-(destino))
RETURN caminho, length(caminho) AS Distancia;
```

> [!NOTE]
> No Neo4j Browser, esta consulta exibirá graficamente toda a trilha de nós intermediários conectando as duas extremidades.

---

### 5.5. Agregação e Recomendação Baseada em Interesses

Descubra quais tecnologias são mais populares entre os estudantes:

```cypher
MATCH (a:Aluno)-[:ESTUDA]->(t:Tecnologia)
RETURN t.nome AS Tecnologia, count(a) AS TotalAlunos, collect(a.nome) AS AlunosInteressados
ORDER BY TotalAlunos DESC;
```

---

## 🛡️ 6. Integridade e Otimização: Restrições e Índices

Em ambientes de produção com milhões de nós, a garantia de unicidade e a velocidade de indexação são críticas.

### 6.1. Restrição de Unicidade (*Unique Constraint*)
Impede a inserção de dois nós com o mesmo valor para a chave (cria automaticamente um índice subjacente):

```cypher
CREATE CONSTRAINT unique_aluno_nome IF NOT EXISTS
FOR (a:Aluno)
REQUIRE a.nome IS UNIQUE;
```

### 6.2. Índice de Propriedade (*Range Index*)
Acelera buscas que usam filtros de igualdade ou intervalos (`>`, `<`, `IN`):

```cypher
CREATE INDEX idx_tecnologia_nome IF NOT EXISTS
FOR (t:Tecnologia)
ON (t.nome);
```

### 6.3. Inspecionar Índices e Restrições Ativas

```cypher
SHOW CONSTRAINTS;
SHOW INDEXES;
```

---

## 🧩 7. Desafios Práticos (Para Fixação)

1. **Modelagem Acadêmica FATEC:**
   Crie uma estrutura contendo:
   - 3 nós `:Disciplina` (ex: "Tópicos de BD", "Engenharia de Software", "Estrutura de Dados").
   - 1 nó `:Professor` conectado às disciplinas via relacionamento `[:MINISTRA]`.
   - Conecte os alunos existentes às disciplinas com o relacionamento `[:MATRICULADO_EM {semestre: 20261}]`.

2. **Detecção de Colegas de Turma:**
   Escreva uma consulta Cypher que retorne quais alunos compartilham ao menos uma disciplina em comum, sem repetir pares invertidos (dica: utilize IDs ou comparação lexicográfica `a1.nome < a2.nome`).

3. **Recomendação de Estudos:**
   Escreva uma consulta que recomende uma `:Tecnologia` para o aluno **Lucas** com base no que seus amigos estudam, excluindo as tecnologias que Lucas já estuda.

4. **Teste de Persistência do Docker:**
   Saia do navegador, execute no terminal do host `docker compose down`, em seguida suba novamente com `docker compose up -d`. Conecte-se e verifique com `MATCH (n) RETURN count(n);` se todas as suas entidades e conexões foram preservadas intactas pelo volume persistido.

---

## 🛑 8. Encerramento e Limpeza

### 8.1. Sair do CLI `cypher-shell`
Se estiver conectado via terminal:
```cypher
:exit
```

### 8.2. Gerenciamento dos Containers no Host

```bash
# Pausar o container temporariamente
docker compose stop

# Parar e remover o container mantendo os volumes de dados salvos
docker compose down

# ATENÇÃO: Para remover completamente os dados e reiniciar o banco do zero
docker compose down -v
```

### 8.3. Reset Geral do Grafo (Via Cypher)
Caso queira esvaziar todos os nós e arestas sem derrubar o container:

```cypher
MATCH (n)
DETACH DELETE n;
```

---

## 🎁 9. Bônus: Procedimentos APOC e Utilitários

Como configuramos `NEO4J_PLUGINS=["apoc"]` no nosso `docker-compose.yml`, você tem acesso a centenas de procedimentos analíticos extras:

### 9.1. Verificar a versão do APOC instalada
```cypher
RETURN apoc.version();
```

### 9.2. Metadados do Grafo em Tempo Real
Gera um resumo estatístico das entidades sem varrer o banco todo:

```cypher
CALL apoc.meta.stats() YIELD nodeCount, relCount, labels, relTypes;
```

### 9.3. Dica Visual no Neo4j Browser
- Ao clicar em qualquer etiqueta ou tipo de relacionamento na barra superior do Neo4j Browser, você pode alterar no painel inferior a **cor**, o **tamanho** do círculo e a **propriedade exibida como legenda** (ex: exibir o campo `nome`).
- É possível exportar a visualização como **SVG**, **PNG** ou os dados brutos como **JSON** e **CSV**.
