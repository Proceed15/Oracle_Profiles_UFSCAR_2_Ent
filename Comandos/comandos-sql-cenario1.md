# Roteiro de Execucao e Comandos SQL - Cenario 1: Rastreamento Isolado e Perfil Limpo no Oracle

Este documento organiza formalmente todos os scripts e comandos SQL, PL/SQL e de terminal executados para a criacao do esquema, carga de dados, coleta estatistica e geracao do perfil de desempenho limpo (*Profile_Limpo_20112026.txt*), de acordo com a metodologia de Cary Millsap em *Thinking Clearly About Performance*.

---

## 1. Preparacao do Ambiente e Concessao de Acessos

Comandos administrativos executados no SQL*Plus autenticado como SYSDBA para inicializacao da instancia, desbloqueio da conta de aplicacao e liberacao de privilegios.

```sql
-- Conectar a instancia como administrador do sistema
CONNECT sys/SenhaBancoOracle AS SYSDBA;

-- Inicializar a instancia e abrir o banco de dados (caso esteja inativa)
STARTUP;

-- Desbloquear a conta hr e definir credencial de acesso
ALTER USER hr IDENTIFIED BY SenhaBancoOracle ACCOUNT UNLOCK;

-- Conceder privilegios para manipulacao de esquema e sessoes
GRANT DBA TO hr;

-- Conectar com a conta de trabalho da aplicacao
CONNECT hr/SenhaBancoOracle;
```

---

## 2. Definicao de Esquema (DDL)

Criacao da tabela mestre de encomendas e da tabela dependente de historico de eventos logisticos, garantindo integridade referencial e indice unico.

```sql
-- Criacao da tabela principal de entregas
CREATE TABLE entregas (
    id_entrega NUMBER PRIMARY KEY,
    codigo_rastreio VARCHAR2(20) UNIQUE NOT NULL,
    remetente VARCHAR2(100),
    destinatario VARCHAR2(100),
    data_postagem DATE DEFAULT SYSDATE
);

-- Criacao da tabela de eventos de rastreamento com chave estrangeira
CREATE TABLE eventos_rastreio (
    id_evento NUMBER PRIMARY KEY,
    id_entrega NUMBER,
    status VARCHAR2(50) NOT NULL,
    localizacao VARCHAR2(100),
    data_hora TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT fk_entrega FOREIGN KEY (id_entrega) REFERENCES entregas(id_entrega)
);
```

---

## 3. Carga Inicial de Dados de Teste (DML)

Insercao dos registros simulando postagens e eventos logisticos internacionais com confirmacao transacional.

```sql
-- Encomenda 1: Rastreamento nacional
INSERT INTO entregas (id_entrega, codigo_rastreio, remetente, destinatario)
VALUES (1, 'BR123456789BR', 'Loja Central', 'Jose Roberto');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (1, 1, 'Postado', 'Agencia Sorocaba');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (2, 1, 'Em Transito', 'Centro de Distribuicao SP');

-- Encomenda 2: Rastreamento internacional
INSERT INTO entregas (id_entrega, codigo_rastreio, remetente, destinatario)
VALUES (2, 'BR987654321US', 'Jose Roberto', 'Filial California, USA');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (3, 2, 'Postado (Envio Internacional)', 'Agencia Central - Sorocaba');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (4, 2, 'Chegado ao Aeroporto', 'Aeroporto de Guarulhos - SP');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (5, 2, 'Liberado na Alfandega', 'Alfandega Americana - CA');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (6, 2, 'Entregue ao Destinatario', 'Filial California, USA');

-- Confirmacao definitiva das transacoes no banco
COMMIT;
```

---

## 4. Coleta Previa de Estatisticas e Formatacao do Ambiente

Execucao do pacote DBMS_STATS para alimentar o dicionario de dados do otimizador CBO, eliminando a ocorrencia de amostragem dinamica (*Dynamic Sampling*) durante o rastreamento posterior.

```sql
-- Analisar e popular estatisticas das tabelas antes de ligar o trace
EXEC DBMS_STATS.gather_table_stats('HR', 'ENTREGAS');
EXEC DBMS_STATS.gather_table_stats('HR', 'EVENTOS_RASTREIO');

-- Formatacao visual do SQL*Plus para organizacao da saida tabular
SET LINESIZE 200;
SET PAGESIZE 100;
COLUMN codigo_rastreio FORMAT A15;
COLUMN remetente FORMAT A15;
COLUMN destinatario FORMAT A25;
COLUMN status FORMAT A30;
COLUMN localizacao FORMAT A30;
COLUMN data_atualizacao FORMAT A16;
```

---

## 5. Rastreamento Estendido e Execucao da Consulta Analitica

Ciclo de instrumentacao da sessao com ativacao do evento 10046 no Nivel 12 (Extended SQL Trace com captura de valores de bind e eventos de espera), execucao da consulta analitica e desligamento formal.

```sql
-- Definir identificador exclusivo no nome do arquivo de saida no disco
ALTER SESSION SET tracefile_identifier = 'Consulta_limpa';

-- Ativacao do evento 10046 nivel 12
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

-- Consulta analitica com juncao e ordenacao temporal decrescente
SELECT e.codigo_rastreio, e.destinatario, ev.status, ev.localizacao,
       TO_CHAR(ev.data_hora, 'DD/MM/YYYY HH24:MI') AS data_atualizacao
FROM entregas e
JOIN eventos_rastreio ev ON e.id_entrega = ev.id_entrega
WHERE e.codigo_rastreio = 'BR987654321US'
ORDER BY ev.data_hora DESC;

-- Desativacao imediata do rastreamento
ALTER SESSION SET EVENTS '10046 trace name context off';

-- Identificacao do caminho absoluto do arquivo .trc gerado
SELECT value FROM v$diag_info WHERE name = 'Default Trace File';

-- Encerrar a sessao para descarregar todos os buffers do trace no arquivo
EXIT;
```

---

## 6. Processamento do Perfil de Desempenho com TKPROF

Comandos executados no Prompt de Comando do Windows (CMD), padronizados para navegacao direta no diretorio oficial de rastreamento do Oracle para gravacao local do arquivo de perfil tabular.

```cmd
:: Navegar ate o diretorio oficial de trace da instancia Oracle
cd C:\oracle\diag\rdbms\oracle\oracle\trace

:: Processar o trace gerando o perfil limpo ordenado pelo tempo de resposta
tkprof oracle_ora_1000_Consulta_limpa.trc Profile_Limpo_20112026.txt sys=no waits=yes sort=prsela,exeela,fchela
```

---

## 7. Sumario Comparativo dos Perfis Gerados

A tabela sintetiza as metricas apuradas entre o primeiro teste (com presenca de amostragem dinamica) e o teste uniforme refinado (apos a execucao de DBMS_STATS).

| Metrica Analisada | Teste 1 (Profile_Rastreio_em_17112026.txt) | Teste 2 Limpo (Profile_Limpo_20112026.txt) |
| :--- | :--- | :--- |
| Ocorrencia de Amostragem Dinamica | Presente (`SELECT /* OPT_DYN_SAMP */`) | Ausente (Eliminada na origem) |
| Total de Instrucoes SQL no Trace | 3 instrucoes (104 linhas) | 2 instrucoes (51 linhas) |
| Leituras Logicas no Parse da Query | 8 buffers consistentes (`query=8`) | 0 buffers consistentes (`query=0`) |
| Leituras Logicas no Fetch da Query | 9 buffers consistentes (`query=9`) | 9 buffers consistentes (`query=9`) |
| Leituras Logicas Totais da Query | 17 buffers consistentes (`query=17`) | 9 buffers consistentes (`query=9`) |
| Leituras Fisicas em Disco (Disk) | 0 blocos | 0 blocos |
| Total de Chamadas Recursivas | 3 chamadas / 7 buffers | 0 chamadas / 0 buffers |
