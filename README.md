# MySQL Shell: JSON, Document Store e X DevAPI na Prática

**Público:** DBAs e profissionais com experiência em MySQL  
**Versão de referência:** MySQL 8.0+ / MySQL Shell 8.0+
**Autor:** Marcio Costa
**Ambiente de simulação:** VM Oracle linux 8.10 - MySQL 8.4.8

---

## Introdução

O MySQL Shell (`mysqlsh`) vai muito além de um simples cliente de linha de comando. Ele é o ponto de entrada para uma série de funcionalidades modernas do MySQL: suporte nativo a JSON como tipo de dado de primeira classe, uma API NoSQL completa via X DevAPI, e o Document Store — que permite trabalhar com coleções de documentos sem abrir mão da robustez de um banco relacional.

Neste laboratorio você vai colocar a mão na massa em três frentes:

1. **JSON no SQL** — colunas `JSON`, funções de extração e manipulação, índices funcionais
2. **X DevAPI e Document Store** — CRUD com coleções, sem SQL
3. **Índices em campos JSON** — gerados como colunas virtuais para performance real

Todos os exemplos são executáveis diretamente no MySQL Shell.

---

## Ambiente

### Instalação do MySQL Shell

```bash
# Ubuntu/Debian
sudo apt install mysql-shell

# RHEL/CentOS
sudo yum install mysql-shell

# macOS (Homebrew)
brew install mysql-shell
```

### Conectando ao servidor

O MySQL Shell suporta dois protocolos:

| Protocolo | Porta padrão | Uso                        |
|-----------|-------------|----------------------------|
| Classic   | 3306        | SQL tradicional             |
| X Protocol| 33060       | X DevAPI, Document Store   |

```bash
# Conexão clássica (SQL)
mysqlsh root@localhost:3306 --sql

# Conexão X Protocol (NoSQL / Document Store)
mysqlsh root@localhost:33060 --js
```

> **Nota:** O X Protocol requer que o plugin `mysqlx` esteja ativo. Verifique com `SHOW PLUGINS;` e confirme `mysqlx | ACTIVE`.

### Alternando modos dentro do Shell

```
\sql   -- modo SQL
\js    -- modo JavaScript
\py    -- modo Python
\help  -- ajuda contextual
```

---

## Parte 1 — JSON no SQL

### 1.1 Criando tabelas com colunas JSON

```sql
CREATE DATABASE loja;
USE loja;

CREATE TABLE pedidos (
  id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  cliente_id  INT UNSIGNED NOT NULL,
  dados       JSON NOT NULL,
  criado_em   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

O tipo `JSON` valida o documento na inserção e armazena em formato binário otimizado — não é um `TEXT` disfarçado.

### 1.2 Inserindo documentos

```sql
INSERT INTO pedidos (cliente_id, dados) VALUES
(1, '{
  "status": "aprovado",
  "total": 349.90,
  "itens": [
    {"sku": "A001", "nome": "Teclado Mecânico", "qty": 1, "preco": 299.90},
    {"sku": "B012", "nome": "Mousepad XL",      "qty": 1, "preco": 50.00}
  ],
  "entrega": {
    "tipo": "expressa",
    "prazo_dias": 2,
    "endereco": {"cidade": "Ribeirão Preto", "uf": "SP"}
  }
}'),
(2, '{
  "status": "pendente",
  "total": 1200.00,
  "itens": [
    {"sku": "C099", "nome": "Monitor 27 polegadas", "qty": 1, "preco": 1200.00}
  ],
  "entrega": {
    "tipo": "normal",
    "prazo_dias": 7,
    "endereco": {"cidade": "São Paulo", "uf": "SP"}
  }
}');
```

### 1.3 Extraindo valores com `->` e `->>`

O operador `->` retorna o valor como JSON; `->>`  retorna como texto (unquoted).

```sql
-- Retorna com aspas: "aprovado"
SELECT dados->'$.status' AS status FROM pedidos WHERE id = 1;

-- Retorna sem aspas: aprovado
SELECT dados->>'$.status' AS status FROM pedidos WHERE id = 1;

-- Acessando objetos aninhados
SELECT
  id,
  dados->>'$.entrega.tipo'             AS tipo_entrega,
  dados->>'$.entrega.endereco.cidade'  AS cidade
FROM pedidos;
```

### 1.4 Funções JSON essenciais

```sql
-- JSON_EXTRACT: equivalente ao ->
SELECT JSON_EXTRACT(dados, '$.total') AS total FROM pedidos;

-- JSON_UNQUOTE: remove aspas do resultado de JSON_EXTRACT
SELECT JSON_UNQUOTE(JSON_EXTRACT(dados, '$.status')) AS status FROM pedidos;

-- JSON_CONTAINS: verifica presença de um valor
SELECT id FROM pedidos
WHERE JSON_CONTAINS(dados->'$.itens[*].sku', '"A001"');

-- JSON_SEARCH: encontra o caminho de um valor
SELECT JSON_SEARCH(dados, 'one', 'A001', NULL, '$.itens[*].sku') AS path
FROM pedidos WHERE id = 1;

-- JSON_ARRAYAGG: agrega linhas em array JSON
SELECT cliente_id, JSON_ARRAYAGG(dados->>'$.status') AS statuses
FROM pedidos GROUP BY cliente_id;

-- JSON_ARRAY e JSON_OBJECT: constroem JSON inline
SELECT JSON_OBJECT(
  'pedido_id', id,
  'cliente',   cliente_id,
  'total',     dados->>'$.total'
) AS resumo
FROM pedidos;
```

### 1.5 Modificando documentos

```sql
-- JSON_SET: insere ou substitui
UPDATE pedidos
SET dados = JSON_SET(dados, '$.status', 'enviado', '$.entrega.prazo_dias', 1)
WHERE id = 1;

-- JSON_INSERT: insere apenas se o caminho não existir
UPDATE pedidos
SET dados = JSON_INSERT(dados, '$.nota_fiscal', 'NF-2024-001')
WHERE id = 1;

-- JSON_REMOVE: remove um campo
UPDATE pedidos
SET dados = JSON_REMOVE(dados, '$.nota_fiscal')
WHERE id = 1;

-- JSON_MERGE_PATCH: merge superficial (RFC 7396)
UPDATE pedidos
SET dados = JSON_MERGE_PATCH(dados, '{"prioridade": "alta"}')
WHERE id = 2;
```

### 1.6 Expandindo arrays com `JSON_TABLE`

`JSON_TABLE` é uma das funções mais poderosas: transforma JSON em linhas relacionais.

```sql
SELECT
  p.id AS pedido_id,
  item.sku,
  item.nome,
  item.qty,
  item.preco,
  item.qty * item.preco AS subtotal
FROM pedidos p,
JSON_TABLE(
  p.dados,
  '$.itens[*]' COLUMNS (
    sku   VARCHAR(20) PATH '$.sku',
    nome  VARCHAR(100) PATH '$.nome',
    qty   INT          PATH '$.qty',
    preco DECIMAL(10,2) PATH '$.preco'
  )
) AS item;
```

---

## Parte 2 — X DevAPI e Document Store

O Document Store do MySQL expõe uma API NoSQL sobre tabelas especiais chamadas **coleções** (`Collections`). Internamente são tabelas com coluna `doc JSON` e `_id` gerado automaticamente — mas você interage com elas via objetos, não SQL.

### 2.1 Criando um schema e uma coleção (modo JS)

```bash
mysqlsh root@localhost:33060 --js
```

```javascript
// Obtém referência ao schema
var db = session.getSchema('loja');

// Cria a coleção (se não existir)
db.createCollection('produtos');

// Ou acessa uma já existente
var col = db.getCollection('produtos');
```

### 2.2 CRUD com coleções

#### Inserção — `add()`

```javascript
col.add(
  { nome: "SSD NVMe 1TB", categoria: "armazenamento", preco: 459.90,
    specs: { leitura_mb: 3500, gravacao_mb: 3000, interface: "PCIe 4.0" },
    tags: ["nvme", "ssd", "storage"], estoque: 42 },

  { nome: "RAM DDR5 32GB", categoria: "memoria", preco: 379.00,
    specs: { velocidade_mhz: 5600, latencia: "CL36", tipo: "DDR5" },
    tags: ["ram", "memoria", "ddr5"], estoque: 18 },

  { nome: "CPU Ryzen 9 7950X", categoria: "processador", preco: 3299.00,
    specs: { nucleos: 16, threads: 32, tdp_w: 170, socket: "AM5" },
    tags: ["cpu", "amd", "ryzen"], estoque: 5 }
).execute();
```

O `_id` é gerado automaticamente como UUID v4 se não fornecido.

#### Consulta — `find()`

```javascript
// Todos os documentos
col.find().execute();

// Com filtro
col.find('categoria = "armazenamento"').execute();

// Projeção de campos
col.find('preco > 400')
   .fields('nome', 'preco', 'estoque')
   .execute();

// Ordenação e limite
col.find()
   .sort('preco DESC')
   .limit(2)
   .execute();

// Acesso a campos aninhados com notação de ponto
col.find('specs.nucleos >= 8')
   .fields('nome', 'specs.nucleos', 'specs.threads')
   .execute();

// Operador IN
col.find(':tag IN tags')
   .bind('tag', 'nvme')
   .execute();
```

#### Atualização — `modify()`

```javascript
// Atualiza campo simples
col.modify('nome = "RAM DDR5 32GB"')
   .set('estoque', 25)
   .execute();

// Incremento com expressão
col.modify('estoque < 10')
   .set('alerta', 'estoque_baixo')
   .execute();

// Remove campo
col.modify('alerta = "estoque_baixo"')
   .unset('alerta')
   .execute();

// Atualiza campo aninhado
col.modify('nome = "SSD NVMe 1TB"')
   .set('specs.leitura_mb', 3600)
   .execute();
```

#### Remoção — `remove()`

```javascript
// Por filtro
col.remove('estoque = 0').execute();

// Por _id específico (substitua pelo id real retornado no find)
col.remove('_id = :id').bind('id', '<uuid-aqui>').execute();
```

### 2.3 Transações com Document Store

```javascript
session.startTransaction();

try {
  col.modify('nome = "CPU Ryzen 9 7950X"')
     .set('estoque', 4)
     .execute();

  // outras operações...

  session.commit();
  print('Transação confirmada.');
} catch (e) {
  session.rollback();
  print('Rollback: ' + e.message);
}
```

### 2.4 Inspecionando a coleção como tabela (SQL)

Por baixo dos panos, a coleção é uma tabela normal:

```sql
\sql
DESCRIBE loja.produtos;
SHOW CREATE TABLE loja.produtos\G
```

Você verá uma coluna `doc JSON NOT NULL` e um índice gerado sobre `_id`.

---

## Parte 3 — Índices em Campos JSON

Sem índices, qualquer filtro em campo JSON faz full scan. A solução é criar **colunas virtuais geradas** e indexá-las.

### 3.1 Índice em coluna da tabela `pedidos`

```sql
\sql
USE loja;

-- Coluna virtual gerada a partir do JSON
ALTER TABLE pedidos
  ADD COLUMN status_pedido VARCHAR(20)
    GENERATED ALWAYS AS (dados->>'$.status') VIRTUAL;

-- Índice sobre a coluna virtual
CREATE INDEX idx_status ON pedidos (status_pedido);

-- Confirmação
EXPLAIN SELECT * FROM pedidos WHERE dados->>'$.status' = 'aprovado'\G
-- key: idx_status ✓
```

### 3.2 Índice funcional (MySQL 8.0+)

Alternativa mais direta sem criar a coluna explicitamente:

```sql
-- Adiciona a coluna virtual
ALTER TABLE pedidos
  ADD COLUMN cidade_entrega VARCHAR(100)
    GENERATED ALWAYS AS (dados->>'$.entrega.endereco.cidade') VIRTUAL;

-- Cria o índice na coluna virtual
CREATE INDEX idx_cidade ON pedidos (cidade_entrega);

-- Verifica o uso do índice
EXPLAIN SELECT id FROM pedidos
WHERE cidade_entrega = 'São Paulo'\G
```

### 3.3 Índice em coleção via X DevAPI

```javascript
\js
var col = session.getSchema('loja').getCollection('produtos');

// Índice simples em campo escalar
col.createIndex('idx_categoria', {
  fields: [{ field: '$.categoria', type: 'TEXT(50)' }]
});

// Índice em campo numérico aninhado
col.createIndex('idx_preco', {
  fields: [{ field: '$.preco', type: 'DECIMAL(10,2)' }]
});

// Índice composto
col.createIndex('idx_cat_preco', {
  fields: [
    { field: '$.categoria', type: 'TEXT(50)' },
    { field: '$.preco',     type: 'DECIMAL(10,2)' }
  ]
});

// Listar índices existentes
session.sql('SHOW INDEX FROM loja.produtos').execute();
```

### 3.4 Verificando uso dos índices

```javascript
// Explain via X DevAPI
col.find('categoria = "armazenamento" AND preco < 500')
   .execute()
   .getWarnings();

// Ou via SQL para análise completa
session.sql(`
  EXPLAIN SELECT * FROM loja.produtos
  WHERE (doc->>'$.categoria') = 'armazenamento'
    AND (doc->>'$.preco') < 500
`).execute();
```

---

## Boas Práticas e Considerações para Produção

### Schema validation (MySQL 8.0.19+)

Você pode impor um JSON Schema às coleções:

```javascript
db.createCollection('pedidos_v2', {
  validation: {
    level: 'STRICT',
    schema: {
      type: 'object',
      properties: {
        status: { type: 'string', enum: ['pendente', 'aprovado', 'enviado', 'cancelado'] },
        total:  { type: 'number', minimum: 0 }
      },
      required: ['status', 'total']
    }
  }
});
```

### Quando usar JSON vs colunas relacionais

| Cenário | Recomendação |
|---------|-------------|
| Campos fixos e bem definidos | Colunas relacionais |
| Atributos variáveis por entidade | Coluna JSON + índice funcional |
| Estrutura completamente dinâmica | Document Store |
| Relatórios analíticos pesados | Materialize via `JSON_TABLE` em tabela real |

### Performance: STORED vs VIRTUAL

```sql
-- VIRTUAL: calculado na leitura (menos espaço em disco)
ADD COLUMN status_v VARCHAR(20) GENERATED ALWAYS AS (dados->>'$.status') VIRTUAL;

-- STORED: calculado na escrita (mais rápido na leitura, ocupa espaço)
ADD COLUMN status_s VARCHAR(20) GENERATED ALWAYS AS (dados->>'$.status') STORED;
```

Para campos muito consultados com alta cardinalidade, prefira `STORED`.

---

## Referências

- [MySQL 8.0 — JSON Data Type](https://dev.mysql.com/doc/refman/8.0/en/json.html)
- [MySQL Shell — X DevAPI Guide](https://dev.mysql.com/doc/x-devapi-userguide/en/)
- [MySQL Document Store](https://dev.mysql.com/doc/refman/8.0/en/document-store.html)
- [JSON_TABLE Reference](https://dev.mysql.com/doc/refman/8.0/en/json-table-functions.html)
- [Functional Indexes](https://dev.mysql.com/doc/refman/8.0/en/create-index.html#create-index-functional-key-parts)
