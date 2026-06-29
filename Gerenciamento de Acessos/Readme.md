# Lab 01 — Gerenciamento de Acessos SQL Server

**Área:** Usuários e Permissões  
**Banco:** Microsoft SQL Server  

---

## Objetivo

Demonstrar o ciclo completo de gerenciamento de acessos no SQL Server: criação de logins e usuários, concessão e revogação de permissões em objetos individuais (tabelas, views, functions, procedures), uso de database roles, e o comportamento de `GRANT`, `REVOKE` e `DENY` — incluindo o caso de uso real de bloqueio por coluna.

---

## Contexto: Login vs Usuário no SQL Server

Essa distinção é fundamental e uma das mais cobradas em entrevistas de DBA SQL Server:

| Conceito | Nível | O que é |
|---|---|---|
| **Login** | Instância (servidor) | Credencial para se conectar ao SQL Server |
| **Usuário** | Database | Mapeamento do login dentro de uma database específica |

Um login pode existir sem usuário — mas não consegue acessar nenhuma database. Um usuário sem login é um "usuário órfão" — problema comum após restore de backup em outro servidor.

```
Instância SQL Server
│
└── Login: Patrick (nível servidor)
      │
      ├── Usuário: Patrick → Database: Treinamento_Modulo02_1
      └── Usuário: Patrick → Database: Treinamento_Modulo02_2
```

---

## O que este lab demonstra

| Habilidade | Relevância para o DBA |
|---|---|
| Criar login com SQL Server Authentication | Acesso sem dependência do Active Directory |
| Mapear login como usuário em databases específicas | Controle granular de quais bancos o usuário acessa |
| `GRANT SELECT` em tabela, view, function e procedure | Permissão mínima necessária por objeto |
| `GRANT EXECUTE` em procedure | Distinção entre permissão de leitura e execução |
| `GRANT EXECUTE TO` (escopo de database) | Conceder acesso a todas as procedures de uma vez |
| `DENY` em objeto específico | Bloqueio explícito — sobrepõe qualquer GRANT |
| `DENY` em coluna específica | Granularidade máxima — proteger campo sem bloquear a tabela |
| `REVOKE` — diferença de DENY | Remove a última permissão; não impede acesso futuro |
| `db_datareader` role | Acesso de leitura a todos os objetos da database |
| Verificar permissões via `sys.database_permissions` | Auditoria e diagnóstico de acessos |
| Excluir login e usuário corretamente | Evitar usuários órfãos — problema operacional comum |

---

## Paralelo com Oracle

Este lab foi desenvolvido em paralelo ao estudo de V18–V20 do Oracle (usuários, privilégios e roles). A tabela abaixo mostra o equivalente entre os dois bancos:

| Conceito | SQL Server | Oracle |
|---|---|---|
| Credencial de acesso | `CREATE LOGIN` | `CREATE USER ... IDENTIFIED BY` |
| Acesso a um banco | `CREATE USER FOR LOGIN` | `GRANT CREATE SESSION` |
| Permissão de leitura | `GRANT SELECT ON tabela TO usuario` | `GRANT SELECT ON tabela TO usuario` |
| Permissão de execução | `GRANT EXECUTE ON proc TO usuario` | `GRANT EXECUTE ON proc TO usuario` |
| Bloqueio explícito | `DENY` | Não existe equivalente direto — usa REVOKE ou VPD |
| Acesso a todos os objetos | `ALTER ROLE db_datareader ADD MEMBER` | `GRANT SELECT ANY TABLE` |
| Grupo de permissões | Database Role | Role Oracle |
| Remover permissão | `REVOKE` | `REVOKE` |

---

## Arquivos

```
Gerenciamento de Acessos/
├── README.md                     ← este arquivo
├── LAB_REPORT.md                 ← documentação completa com prints

```

---

## Como reproduzir

```sql
-- Conectar como sysadmin no SQL Server Management Studio (SSMS)
-- Executar o script na ordem das seções
-- Para os testes de permissão: abrir segunda conexão com login Patrick
```

> ℹ️ Partes do lab exigem duas conexões simultâneas: uma como `sysadmin` (para criar objetos e conceder permissões) e outra como `Patrick` (para testar os acessos).

---

## Referências

- Power Tuning — Módulo 02: Tarefas do Dia a Dia de um DBA (Fabrício Lima)
- [Microsoft Docs — CREATE LOGIN](https://learn.microsoft.com/pt-br/sql/t-sql/statements/create-login-transact-sql)
- [Microsoft Docs — GRANT / DENY / REVOKE](https://learn.microsoft.com/pt-br/sql/t-sql/statements/grant-transact-sql)
- [Microsoft Docs — sys.database_permissions](https://learn.microsoft.com/pt-br/sql/relational-databases/system-catalog-views/sys-database-permissions-transact-sql)
