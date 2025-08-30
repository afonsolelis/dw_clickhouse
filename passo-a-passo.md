Segue o passo a passo do que você fez, com comandos, resultado esperado e correções que aplicou.

1. Verificou contêineres em execução

```
docker ps
```

Resultado: nenhum contêiner listado.

2. Tentou abrir o cliente sem contêiner existente

```
docker exec -it clickhouse clickhouse-client
```

Erro: `No such container: clickhouse`.

3. Baixou a imagem do ClickHouse Server

```
docker pull clickhouse/clickhouse-server:latest
```

Resultado: imagem baixada com digest `sha256:dd08d3...`.

4. Confirmou novamente que não havia contêiner

```
docker ps
```

Resultado: vazio.

5. Subiu o servidor ClickHouse com nome, portas e volume

```
docker run -d --name clickhouse \
  -p 8123:8123 -p 9000:9000 \
  -v clickhouse_data:/var/lib/clickhouse \
  clickhouse/clickhouse-server:latest
```

Resultado: contêiner `clickhouse` iniciado.

6. Listou contêineres para conferir o up

```
docker ps
```

Resultado: `clickhouse` Up, portas 8123 e 9000 mapeadas.

7. Abriu o cliente do ClickHouse dentro do contêiner

```
docker exec -it clickhouse clickhouse-client
```

Resultado: conectado à versão `25.8.1`. Warnings do sistema exibidos.

8. Digitou entrada inválida para testar

```
ddddd
```

Erro de sintaxe. Sem efeito.

9. Criou o database do DW

```
CREATE DATABASE data_warehouse;
```

Ok.

10. Listou databases

```
SHOW DATABASES;
```

Resultado: `INFORMATION_SCHEMA, data_warehouse, default, information_schema, system`.

11. Tentou `USE data` e viu sugestões do client. Depois selecionou o DB correto

```
USE data_warehouse;
```

Ok.

12. Tentou criar `dim_usuario` sem colunas e obteve erro de parênteses. Corrigiu criando a tabela com MergeTree e PK

```
CREATE TABLE dim_usuario (
  id_usuario UInt32,
  nome String,
  cidade String
) ENGINE = MergeTree
PRIMARY KEY id_usuario;
```

Ok.

13. Conferiu as tabelas

```
SHOW TABLES;
```

Resultado: `dim_usuario`.

14. Consultou a dimensão vazia

```
SELECT id_usuario, nome, cidade FROM dim_usuario;
```

Resultado: 0 linhas.

15. Inseriu linhas na `dim_usuario`

```
INSERT INTO dim_usuario VALUES
(1001,'Ana','São Paulo'),
(1002,'Bruno','Rio de Janeiro'),
(1003,'Jesse Ribeiro Silva','Cotia'),
(1004,'Kaio Cavalcante','Brasilia'),
(1005,'Pamela','São Paulo');
```

Ok.

16. Validou a carga

```
SELECT id_usuario, nome, cidade FROM dim_usuario;
```

Resultado: 5 linhas.

17. Criou `dim_pagina` com erro de vírgula e PK incorreta (`ig_pagina`). Corrigiu criando com PK certa
    Primeira tentativa: faltou vírgula antes de `url` e PK referenciou coluna inexistente → erro `UNKNOWN_IDENTIFIER`.
    Tentativa correta:

```
CREATE TABLE dim_pagina (
  id_pagina UInt32,
  descricao String,
  url String
) ENGINE = MergeTree
PRIMARY KEY id_pagina;
```

Ok.

18. Conferiu

```
SHOW TABLES;
```

Resultado: `dim_pagina, dim_usuario`.

19. Inseriu linhas na `dim_pagina`

```
INSERT INTO dim_pagina VALUES
(1,'Home','/home'),
(2,'Produto','/produto'),
(3,'Checkout','/checkout');
```

Ok.

20. Consultou com coluna errada `id` e recebeu erro. Consultou com coluna correta
    Errado:

```
SELECT id, descricao, url FROM dim_pagina;
```

Erro `UNKNOWN_IDENTIFIER id`.
Certo:

```
SELECT id_pagina, descricao, url FROM dim_pagina;
```

Resultado: 3 linhas.

21. Criou `dim_tempo`

```
CREATE TABLE dim_tempo (
  id_tempo UInt32,
  data Date,
  mes UInt32,
  ano UInt32,
  dia_semana String
) ENGINE = MergeTree
PRIMARY KEY id_tempo;
```

Ok.

22. Conferiu

```
SHOW TABLES;
```

Resultado: `dim_pagina, dim_tempo, dim_usuario`.

23. Inseriu datas na `dim_tempo`

```
INSERT INTO dim_tempo VALUES
(1,'2025-08-30',30,2025,'Sabado'),
(2,'2025-08-31',31,2025,'Domingo');
```

Ok.

24. Validou a carga

```
SELECT * FROM dim_tempo;
```

Resultado: 2 linhas.

25. Criou fato com erro de digitação no nome da coluna e ORDER BY em coluna inexistente. Corrigiu criando com coluna correta e ORDER BY correspondente
    Primeira tentativa:

```
CREATE TABLE fato_visita (
  id_visira UInt64, ...
) ENGINE = MergeTree() ORDER BY id_visita;
```

Erro: `Missing columns: 'id_visita'` por mismatch `id_visira`.
Tentativa correta:

```
CREATE TABLE fato_visita (
  id_visita UInt64,
  id_usuario UInt32,
  id_pagina UInt32,
  id_tempo UInt32,
  duracao UInt16
) ENGINE = MergeTree
ORDER BY id_visita;
```

Ok.

26. Conferiu

```
SHOW TABLES;
```

Resultado: `dim_pagina, dim_tempo, dim_usuario, fato_visita`.

27. Inseriu 20 linhas de fato

```
INSERT INTO fato_visita VALUES
(1,1001,1,1,120),
...,
(20,1005,2,2,45);
```

Ok.

28. Validou a carga

```
SELECT * FROM fato_visita;
```

Resultado: 20 linhas.

29. Fez consulta analítica com JOINs. Duas tentativas com erro por nome de tabela com espaço (`dim usuario` e `dim usuario_`). Corrigiu usando `dim_usuario`
    Consulta final correta:

```
SELECT
  u.nome AS usuario,
  u.cidade,
  p.descricao AS pagina,
  t.data,
  t.dia_semana,
  f.duracao
FROM fato_visita AS f
JOIN dim_usuario AS u ON f.id_usuario = u.id_usuario
JOIN dim_pagina  AS p ON f.id_pagina  = p.id_pagina
JOIN dim_tempo   AS t ON f.id_tempo   = t.id_tempo
ORDER BY t.data, usuario;
```

Resultado: 20 linhas ordenadas por data e usuário.

—
Observações rápidas
• Sempre referencie no `PRIMARY KEY` e no `ORDER BY` colunas que existem e escreva os nomes exatamente iguais.
• Em `MergeTree`, use `ORDER BY` obrigatório; `PRIMARY KEY` herda de `ORDER BY` se omitido.
• Evite espaços em nomes de tabelas. Use `snake_case`.
• Mensagens `UNKNOWN_IDENTIFIER` e `Unmatched parentheses` indicam vírgulas faltando, nomes errados ou parênteses desequilibrados.
