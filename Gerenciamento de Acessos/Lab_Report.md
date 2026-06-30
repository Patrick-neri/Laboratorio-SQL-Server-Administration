# LAB_REPORT — Lab 01: Gerenciamento de Acessos SQL Server

**Data de execução:** Junho/2026  
**Ambiente:** Microsoft SQL Server — SSMS 22.7 

---

## 1. Contexto e objetivo

Gerenciamento de acessos é uma das responsabilidades mais cobradas em vagas de DBA júnior — e uma das que mais causam incidentes quando feita errada. Conceder permissão demais expõe dados sensíveis. Conceder de menos quebra sistemas em produção. Entender `GRANT`, `DENY` e `REVOKE` — e a diferença entre eles — é habilidade obrigatória.

Este lab cobre o ciclo completo no SQL Server: criação de credenciais, acesso a databases, permissões por objeto, roles, bloqueio por coluna e exclusão correta de usuários. Foi desenvolvido em paralelo ao estudo das aulas V18–V20 do Oracle, com o objetivo de documentar as diferenças e semelhanças entre os dois SGBDs.

**Por que importa na prática:**  
Em um CSC como a Rodobens, onde múltiplos sistemas acessam o mesmo servidor SQL Server com usuários de aplicação distintos, o DBA recebe chamados como "usuário não consegue acessar a procedure" ou "preciso bloquear acesso à tabela de salários" — exatamente os cenários deste lab.

---

## 2. Conceitos fundamentais

### Login vs Usuário: a distinção mais importante do SQL Server

| | Login | Usuário |
|---|---|---|
| **Nível** | Instância (servidor) | Database |
| **Criado em** | `master` | Database específica |
| **Função** | Autenticar no servidor | Autorizar acesso a objetos |
| **Comando** | `CREATE LOGIN` | `CREATE USER FOR LOGIN` |
| **Verificado em** | `sys.sql_logins` / `sys.server_principals` | `sys.database_principals` |

Um login sem usuário: conecta no servidor, mas não acessa nenhuma database.  
Um usuário sem login: "usuário órfão" — aparece no banco mas ninguém consegue usá-lo.

### Os três comandos de permissão

| Comando | O que faz | Pode ser sobreposto? |
|---|---|---|
| `GRANT` | Concede acesso | Sim — DENY substitui |
| `DENY` | Bloqueia explicitamente | Sim — GRANT substitui, mas DENY tem prioridade sobre roles |
| `REVOKE` | Remove a última permissão definida | — |

**Regra crítica:** `DENY` tem precedência sobre `GRANT` via role. Se o usuário está em `db_datareader` (que dá SELECT em tudo) e recebe um `DENY SELECT ON tabela`, ele não consegue acessar aquela tabela. Isso permite o padrão: "acesso geral + bloqueio específico".

---

## 3. Sequência de execução

### 3.1 Criando as databases de teste

```sql
-- Criar database 1 (principal dos testes)
IF EXISTS (SELECT name FROM sys.databases WHERE name = 'Treinamento_Modulo02_1')
    DROP DATABASE Treinamento_Modulo02_1

CREATE DATABASE Treinamento_Modulo02_1

-- Criar database 2 (para testar acesso cross-database)
IF EXISTS (SELECT name FROM sys.databases WHERE name = 'Treinamento_Modulo02_2')
    DROP DATABASE Treinamento_Modulo02_2

CREATE DATABASE Treinamento_Modulo02_2
```

---

### 3.2 Criando o Login Patrick

Via SSMS (interface gráfica), gerei o script de criação antes de executar — boa prática para revisar e documentar:

```sql
-- Nível de instância: criar o login
USE [master]
GO
CREATE LOGIN [Patrick] WITH PASSWORD = N'patrick@123',
    DEFAULT_DATABASE = Treinamento_Modulo02_1,
    CHECK_EXPIRATION = OFF,
    CHECK_POLICY = ON
GO

-- Nível de database: criar o usuário mapeado ao login
USE Treinamento_Modulo02_1
GO
CREATE USER [Patrick] FOR LOGIN [Patrick]
GO
```

> 💡 `CHECK_POLICY = ON` aplica as políticas de senha do Windows ao login SQL Server. `CHECK_EXPIRATION = OFF` desabilita a expiração — útil para contas de serviço, mas não recomendado para contas pessoais.

Validando o login criado:

```sql
SELECT
    name,
    create_date,
    modify_date,
    LOGINPROPERTY(name, 'DaysUntilExpiration') AS DaysUntilExpiration,
    LOGINPROPERTY(name, 'PasswordLastSetTime')  AS PasswordLastSetTime,
    LOGINPROPERTY(name, 'IsExpired')            AS IsExpired,
    LOGINPROPERTY(name, 'IsMustChange')         AS IsMustChange
FROM sys.sql_logins
WHERE name = 'Patrick'
```

> <img width="1574" height="128" alt="image" src="https://github.com/user-attachments/assets/71e3ae49-d103-40df-a60c-e2782eccf311" />

---

### 3.3 Testando o isolamento de databases

Com o login Patrick conectado em uma segunda janela do SSMS, tentei acessar a segunda database:

```sql
-- Conexão como Patrick
USE Treinamento_Modulo02_2
-- Erro: The server principal "Patrick" is not able to access the database
--       "Treinamento_Modulo02_2" under the current security context.
```

> <img width="990" height="129" alt="image" src="https://github.com/user-attachments/assets/88a5963e-c25a-43a1-b910-26602f8f3b40" />

---

### 3.4 Permissões em objetos individuais

Criei a tabela de testes como sysadmin:

```sql
USE Treinamento_Modulo02_1

CREATE TABLE Modulo02 (
    Id_Treinamento  INT IDENTITY,
    Dt_Referencia   DATETIME DEFAULT(GETDATE()),
    Ds_Observacao   VARCHAR(1000)
)

INSERT INTO Modulo02 (Ds_Observacao)
SELECT 'Teste Acesso Database'
GO 10  -- repete o INSERT 10 vezes
```

**Teste de SELECT sem permissão (conexão Patrick):**

```sql
SELECT * FROM Modulo02
-- Msg 229: The SELECT permission was denied on the object 'Modulo02'
```

> <img width="794" height="103" alt="image" src="https://github.com/user-attachments/assets/9af46dcd-03bd-42b4-b222-aef9d31cd811" />

**Conceder e verificar:**

```sql
-- Como sysadmin
GRANT SELECT ON Modulo02 TO Patrick

-- Como Patrick — agora funciona
SELECT * FROM Modulo02  -- ✅ 10 linhas retornadas
```

><img width="404" height="259" alt="image" src="https://github.com/user-attachments/assets/29cac1db-b5ae-4168-820e-383664cddcb5" />


**Permissão em function:**

```sql
-- Criar function (sysadmin)
CREATE FUNCTION fncModulo02 (@Id INT)
RETURNS TABLE AS
RETURN (SELECT * FROM Modulo02 WHERE Id_Treinamento = @Id)

-- Testar como Patrick → Msg 229: SELECT permission denied on 'fncModulo02'

GRANT SELECT ON fncModulo02 TO Patrick
-- Usuário Patrick executa com sucesso
SELECT * FROM fncModulo02(1)
```

> <img width="424" height="104" alt="image" src="https://github.com/user-attachments/assets/401a086b-d38a-45fa-93cf-aa23993a2025" />

**Permissão em view:**

```sql
CREATE VIEW vwModulo02 AS SELECT * FROM Modulo02

-- Patrick → Msg 229: SELECT permission denied on 'vwModulo02'

GRANT SELECT ON vwModulo02 TO Patrick
-- Usuário Patrick executa com sucesso
SELECT * FROM vwModulo02
```

> <img width="412" height="264" alt="image" src="https://github.com/user-attachments/assets/ef8b4f19-fe3e-4e56-aa07-446cfa73afe4" />

**Permissão em procedure:**

```sql
CREATE PROCEDURE stpModulo02 AS SELECT * FROM Modulo02

-- Patrick → Msg 229: The EXECUTE permission was denied on 'stpModulo02'
-- Nota: procedures usam EXECUTE, não SELECT

GRANT EXECUTE ON stpModulo02 TO Patrick
-- Patrick executa com sucesso ✅
EXEC stpModulo02
```

> <img width="412" height="178" alt="image" src="https://github.com/user-attachments/assets/e9604344-be62-42f9-90f1-df4954d2c73b" />

> 💡 Diferença importante: para procedures, o comando é `GRANT EXECUTE` — não `GRANT SELECT`. Para tables, views e functions inline, é `GRANT SELECT`.

---

### 3.5 Permissão para usuário acessar mais de uma procedure:

```sql
-- Criar mais duas procedures
CREATE PROCEDURE stpModulo02_2 AS SELECT * FROM Modulo02
GO
CREATE PROCEDURE stpModulo02_3 AS SELECT * FROM Modulo02

-- Usuário Patrick tenta executar → erro (sem permissão)
EXEC stpModulo02_2
EXEC stpModulo02_3

-- GRANT EXECUTE sem especificar objeto = acesso a TODAS as procedures da database
GRANT EXECUTE TO Patrick

-- Usuário Patrick executa ambas com sucesso 

-- Bloquear uma procedure específica com DENY
DENY EXECUTE ON stpModulo02_2 TO Patrick

-- Usuário Patrick tenta executar:
EXEC stpModulo02_2  -- ❌ Msg 229: EXECUTE permission denied
EXEC stpModulo02_3  -- ✅ Sucesso

```

> Resultado EXEC stpModulo02_2 após o GRANT
> ><img width="399" height="255" alt="image" src="https://github.com/user-attachments/assets/d596e691-b7c7-415d-bb66-3e855fb42224" />

> Resultado EXEC stpModulo02_2 após o DENY
> ><img width="835" height="117" alt="image" src="https://github.com/user-attachments/assets/0e02698a-ae44-426f-b47b-d9eaef853798" />



> 💡 Este é o padrão mais usado em produção: dar acesso amplo e bloquear exceções com `DENY`. O `DENY` sobrepõe qualquer `GRANT` — inclusive o concedido por role ou escopo de database.

---

### 3.6 Liberando acesso a uma procedure que acessa duas databases

Criando cenário realista — procedure na database 1 que acessa dado da database 2:

```sql
-- Criar tabela na database 2 (sysadmin)
USE Treinamento_Modulo02_2
CREATE TABLE Modulo02_Database2 (cod INT)
INSERT INTO Modulo02_Database2 SELECT 20

-- Criar procedure na database 1 que acessa as duas databases
USE Treinamento_Modulo02_1
CREATE PROCEDURE stpModulo02_4 AS
BEGIN
    SELECT * FROM Modulo02
    SELECT * FROM Treinamento_Modulo02_2..Modulo02_Database2
END

-- Usuário Patrick executa → primeiro SELECT retorna, segundo falha:
-- Msg 916: The server principal "Patrick" is not able to access
--          the database "Treinamento_Modulo02_2"
```

> <img width="997" height="117" alt="image" src="https://github.com/user-attachments/assets/eb5347d0-d07a-46f5-be8b-8ce1a5cd87fe" />

Solução: criar usuário e conceder permissão na database 2:

```sql
USE Treinamento_Modulo02_2
GO
CREATE USER [Patrick] FOR LOGIN [Patrick]
GO
GRANT SELECT ON Modulo02_Database2 TO [Patrick]

-- Usuário Patrick executa stpModulo02_4 → ambos os SELECTs retornam 
USE Treinamento_Modulo02_1
EXEC stpModulo02_4
```

> <img width="471" height="159" alt="image" src="https://github.com/user-attachments/assets/ccfd1c9a-da26-4477-b04a-195631f6aa62" />

> 💡 Cross-database access no SQL Server exige que o login tenha usuário mapeado em **cada** database que a procedure acessa — não basta a permissão de `EXECUTE` na procedure.

---

### 3.7 Comportamento de GRANT, DENY e REVOKE

```sql
-- Query de auditoria de permissões por objeto
SELECT
    prmssn.state_desc,
    prmssn.permission_name  AS [Permission],
    sp.type_desc,
    sp.name,
    grantor.name            AS [Grantor],
    grantee.name            AS [Grantee]
FROM sys.all_objects AS sp
JOIN sys.database_permissions AS prmssn
    ON prmssn.major_id = sp.object_id AND prmssn.minor_id = 0 AND prmssn.class = 1
JOIN sys.database_principals AS grantor
    ON grantor.principal_id = prmssn.grantor_principal_id
JOIN sys.database_principals AS grantee
    ON grantee.principal_id = prmssn.grantee_principal_id
WHERE grantee.name = 'Patrick'
```

> <img width="614" height="154" alt="image" src="https://github.com/user-attachments/assets/56ddcf33-9709-4ed3-98be-e1df07d3801f" />

Testando a sequência de GRANT → REVOKE → DENY → REVOKE:

```sql
-- Estado: Usuário Patrick tem GRANT SELECT na tabela Modulo02
SELECT * FROM Modulo02  -- ✅ Usuário Patrick acessa

-- REVOKE remove o GRANT
REVOKE SELECT ON Modulo02 TO Patrick
SELECT * FROM Modulo02  -- ❌ Sem permissão

-- GRANT concede novamente
GRANT SELECT ON Modulo02 TO Patrick
SELECT * FROM Modulo02  -- ✅

-- DENY bloqueia explicitamente (substitui o GRANT)
DENY SELECT ON Modulo02 TO Patrick
SELECT * FROM Modulo02  -- ❌ DENY em vigor

-- GRANT substitui o DENY
GRANT SELECT ON Modulo02 TO Patrick
SELECT * FROM Modulo02  -- ✅

-- REVOKE remove o GRANT (último estado)
REVOKE SELECT ON Modulo02 TO Patrick
SELECT * FROM Modulo02  -- ❌

-- DENY bloqueia
DENY SELECT ON Modulo02 TO Patrick
-- REVOKE remove o DENY (último estado)
REVOKE SELECT ON Modulo02 TO Patrick
-- Agora não há nem GRANT nem DENY → sem acesso, sem bloqueio explícito
```

**Resumo visual do comportamento:**

```
GRANT  → acesso liberado
DENY   → bloqueia (sobrepõe GRANT de role, mas GRANT direto substitui DENY)
REVOKE → remove o último estado definido (seja GRANT ou DENY)
         → não é o oposto de GRANT — é a remoção da última declaração
```

---

### 3.8 Caso de uso real: db_datareader + DENY por coluna

Este é o cenário mais próximo do dia a dia  — um usuário que precisa ler tudo, exceto dados sensíveis:

```sql
-- Adicionar usuário Patrick à role db_datareader (leitura de todos os objetos)
ALTER ROLE [db_datareader] ADD MEMBER [Patrick]

-- Criar tabela de salários
CREATE TABLE Salario (
    Id_Colaborador  INT IDENTITY,
    Nm_Colaborador  VARCHAR(100),
    Vl_Salario      NUMERIC(9,2)
)

INSERT INTO Salario (Nm_Colaborador, Vl_Salario)
VALUES ('Estagiario', 675.00),
       ('Gerente', 20000),
       ('DBA em São Paulo', 10000),
       ('DBA em Vitória', 3000)

-- Usuário Patrick lê a tabela inteira (via db_datareader) 
SELECT * FROM Salario

-- Bloquear acesso completo à tabela
DENY SELECT ON Salario TO [Patrick]
SELECT * FROM Salario
-- ❌ Msg 229: SELECT permission was denied on 'Salario'

-- Alternativa mais precisa: bloquear apenas a coluna de salário
REVOKE SELECT ON Salario TO [Patrick]  -- remove o DENY da tabela
DENY SELECT ON Salario(Vl_Salario) TO [Patrick]

-- Usuário Patrick acessa apenas as colunas permitidas:
SELECT Id_Colaborador, Nm_Colaborador FROM Salario  -- ✅
SELECT Vl_Salario FROM Salario                      -- ❌ coluna bloqueada
```

> Consulta antes de bloquear acesso à coluna
> ><img width="314" height="141" alt="image" src="https://github.com/user-attachments/assets/e72a2eb2-6087-44eb-8ee8-cbc48c16f4e1" />

> Erro após bloquear a coluna e fazer uma consulta
> ><img width="936" height="125" alt="image" src="https://github.com/user-attachments/assets/88819b92-fb79-4de7-bd9c-378a4105c6b7" />

> Acesso apenas as colunas permitidas
> ><img width="314" height="171" alt="image" src="https://github.com/user-attachments/assets/b7d83409-26b7-485f-b1d5-7d067041bf69" />




> 💡 `DENY` por coluna é o nível máximo de granularidade de permissão no SQL Server — permite proteger dados sensíveis sem bloquear o acesso à tabela inteira.

---

### 3.9 Excluindo o login Patrick corretamente

```sql
-- Verificar conexões abertas antes de excluir
SELECT loginame, 'kill ' + CAST(spid AS CHAR(5)), *
FROM sysprocesses
WHERE loginame = 'Patrick'

-- Encerrar conexões abertas (substituir pelo SPID retornado)
KILL <spid>
```

Excluir via SSMS: Object Explorer → Security → Logins → Patrick → Delete


> ⚠️ Excluir o login NÃO remove automaticamente os usuários mapeados nas databases. O usuário `Patrick` continua existindo em `Treinamento_Modulo02_1` e `Treinamento_Modulo02_2` como usuário órfão. É necessário excluir manualmente em cada database:

```sql
-- Excluir usuário órfão em cada database
USE Treinamento_Modulo02_1
DROP USER [Patrick]

USE Treinamento_Modulo02_2
DROP USER [Patrick]
```

---

## 4. Erros encontrados e como resolvi

### Erro 1 — Msg 229: SELECT permission denied (sem permissão em função/view)

**Causa:** função inline e view precisam de permissão própria — herdar `GRANT SELECT` na tabela base não é suficiente.  
**Solução:** `GRANT SELECT ON fncModulo02 TO Patrick` e `GRANT SELECT ON vwModulo02 TO Patrick`.  
**Aprendizado:** cada objeto tem sua ACL independente. Permissão na tabela não se propaga para views e functions que a referenciam.

---

### Erro 2 — Msg 229: EXECUTE permission denied (tentativa de GRANT SELECT em procedure)

**Causa:** procedures não usam `SELECT` — usam `EXECUTE`.  
**Solução:** `GRANT EXECUTE ON stpModulo02 TO Patrick`.  
**Aprendizado:** a permissão correta por tipo de objeto é: `SELECT` para tabelas/views/functions, `EXECUTE` para procedures e functions escalares.

---

### Erro 3 — Msg 916 ao executar procedure cross-database

```
The server principal "Patrick" is not able to access the database
"Treinamento_Modulo02_2" under the current security context.
```

**Causa:** login `Patrick` não tinha usuário mapeado na database 2.  
**Solução:** `CREATE USER [Patrick] FOR LOGIN [Patrick]` na database 2 + `GRANT SELECT` na tabela.  
**Aprendizado:** cross-database access requer que o login tenha usuário em **todas** as databases envolvidas — não basta ter permissão na procedure.

---

## 5. Conceitos consolidados

**Hierarquia de permissões SQL Server:**

```
Instância
 └── Login (CREATE LOGIN)
       └── Database
             └── Usuário (CREATE USER FOR LOGIN)
                   ├── Role (ALTER ROLE db_datareader ADD MEMBER)
                   └── Objeto (GRANT/DENY/REVOKE SELECT/EXECUTE ON objeto)
                         └── Coluna (DENY SELECT ON tabela(coluna))
```

**Precedência do DENY:**

```
DENY em objeto  >  GRANT via role  >  sem permissão
GRANT em objeto  >  DENY via role  (GRANT direto sobrepõe DENY de role)
```

**Padrão mais usado em produção:**

```sql
-- 1. Dar acesso geral via role
ALTER ROLE [db_datareader] ADD MEMBER [usuario]

-- 2. Bloquear exceções com DENY
DENY SELECT ON tabela_sensiveis TO [usuario]
-- ou
DENY SELECT ON tabela(coluna_sensiivel) TO [usuario]
```

---

