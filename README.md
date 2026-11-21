# Instrumentacao de Software e Analise de Desempenho com Oracle Database: Aplicacao Pratica do Metodo de Cary Millsap

Este repositorio documenta um laboratorio experimental de engenharia de desempenho de software fundamentado nos principios metodologicos descritos por Cary Millsap no artigo *Thinking Clearly About Performance* (publicado na *Communications of the ACM* e na *ACM Queue* em 2010).

O objetivo do projeto e demonstrar, por meio de dados empiricos de rastreamento e perfilamento em banco de dados relacional, a substituicao da abordagem tradicional de tentativa e erro (*trial-and-error*) por uma metodologia axiomatica baseada na decomposicao matematica do tempo de resposta. O laboratorio evoluiu de uma medicao preliminar (em que se diagnosticou a presenca de consultas recursivas de amostragem dinamica) para uma execucao rigorosamente limpa e isolada, em total conformidade com a literatura de engenharia de desempenho.

---

## 1. Fundamentacao Teorica: O Modelo de Cary Millsap

A engenharia de desempenho de sistemas computacionais frequentemente sofre com praticas empiricas nao estruturadas, nas quais alteracoes de configuracao ou indices sao introduzidas sem comprovacao causal previa. Conforme postulado por Millsap, otimizar sistemas por tentativa e erro e inviavel em arquiteturas multicamadas complexas. A analise formal exige medicao deterministica e auditavel do consumo de tempo.

### 1.1. Decomposicao do Tempo de Resposta

O tempo de resposta ($R$) percebido por uma transacao ou tarefa de usuario e decomposto no somatorio do tempo de servico ($S$), em que recursos como CPU estao efetivamente processando instrucoes, e do tempo de espera ($W$), no qual o processo aguarda a liberacao de recursos de infraestrutura, bloqueios ou transferencias de rede:

$$R = S + W$$

Em arquiteturas cliente-servidor ou sistemas multicamadas, a execucao de uma operacao de negocio gera multiplos intercambios de mensagens e chamadas procedurais entre a aplicacao e o subsistema de persistencia.

### 1.2. Limitacao dos Diagramas de Sequencia e Necessidade do Perfil

Embora diagramas de sequencia UML sejam uteis para conceituar fluxos de interacao em pequena escala, eles tornam-se ineficazes para diagnosticar tarefas reais que executam milhares de operacoes atomicas. A representacao visual nao escala diante de volumes macicos de chamadas repetitivas.

Para resolver essa limitacao, a metodologia preconiza a construcao do Perfil de Tempo de Resposta (*Resource Profile*): uma tabela ordenada de forma decrescente onde cada linha representa o consumo de tempo acumulado por categoria de chamada ou evento de espera. O perfil evidencia com clareza matematica onde o tempo foi consumido e onde nao foi consumido, orientando a priorizacao das intervencoes com base na Lei de Amdahl.

---

## 2. Arquitetura do Experimento e Ambiente Operacional

O laboratorio foi executado em ambiente controlado sob sistema operacional Microsoft Windows de 64 bits, utilizando uma instancia local do Oracle Database configurada para instrumentacao estendida de sessao.

| Componente | Especificacao |
| :--- | :--- |
| Sistema Operacional | Microsoft Windows x64 (Compilacao 10.0.26200.9974) |
| Sistema Gerenciador de Banco de Dados | Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 (64-bit Production) |
| Ferramentas de Linha de Comando | Oracle SQL*Plus Release 11.2.0.1.0, Oracle TKPROF Release 11.2.0.1.0 |
| Mecanismo de Instrumentacao | Evento 10046 Nivel 12 (Extended SQL Trace com binds e esperas) |
| Identificador da Instancia (SID) | oracle |
| Diretorio Padrao de Diagnostico | C:\oracle\diag\rdbms\oracle\oracle\trace |

---

## 3. Modelo de Dados e Carga de Teste

O cenario de testes simula um subsistema de logistica e rastreamento de encomendas internacionais, composto por duas entidades relacionais ligadas por restricao de integridade referencial:

1. **`entregas`**: Tabela mestre que armazena os dados cadastrais da encomenda, incluindo identificador primario, codigo de rastreamento com restricao de unicidade (`UNIQUE NOT NULL`), remetente e destinatario.
2. **`eventos_rastreio`**: Tabela de detalhe que registra o historico de movimentacao fisica da encomenda, referenciando a tabela mestre via chave estrangeira (`FOREIGN KEY`) e registrando status, localizacao geografica e carimbo de data/hora (`TIMESTAMP`).

```sql
-- DDL de criacao das estruturas relacionais
CREATE TABLE entregas (
    id_entrega NUMBER PRIMARY KEY,
    codigo_rastreio VARCHAR2(20) UNIQUE NOT NULL,
    remetente VARCHAR2(100),
    destinatario VARCHAR2(100),
    data_postagem DATE DEFAULT SYSDATE
);

CREATE TABLE eventos_rastreio (
    id_evento NUMBER PRIMARY KEY,
    id_entrega NUMBER,
    status VARCHAR2(50) NOT NULL,
    localizacao VARCHAR2(100),
    data_hora TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT fk_entrega FOREIGN KEY (id_entrega) REFERENCES entregas(id_entrega)
);
```

A carga de dados foi populada com eventos operacionais simulando o fluxo de envio internacional:

```sql
INSERT INTO entregas (id_entrega, codigo_rastreio, remetente, destinatario)
VALUES (1, 'BR123456789BR', 'Loja Central', 'Jose Roberto');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (1, 1, 'Postado', 'Agencia Sorocaba');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (2, 1, 'Em Transito', 'Centro de Distribuicao SP');

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

COMMIT;
```

---

## 4. Evolucao Metodologica: Da Medicao com Ruido a Execucao Limpa

A conducao do experimento ocorreu em dois momentos distintos, ilustrando a importancia do controle de variaveis de contorno em testes de software:

### 4.1. O Primeiro Teste e o Diagnostico da Amostragem Dinamica

No primeiro ensaio (`Profile_Rastreio_em_17112026.txt`), a consulta foi submetida imediatamente apos a insercao dos dados, sem a coleta previa de metadados estatisticos no dicionario de dados. O Otimizador Baseado em Custo (*Cost-Based Optimizer - CBO*) detectou ausencia de estatisticas e disparou uma consulta recursiva interna:

```sql
SELECT /* OPT_DYN_SAMP */ /*+ ALL_ROWS IGNORE_WHERE_CLAUSE ... */ 
  NVL(SUM(C1),:"SYS_B_0"), NVL(SUM(C2),:"SYS_B_1"), COUNT(DISTINCT C3) ... 
FROM (SELECT /*+ NO_PARALLEL("EV") FULL("EV") ... */ ... FROM "EVENTOS_RASTREIO" "EV") SAMPLESUB;
```

Essa operacao acessoria gerou 3 chamadas recursivas (`Parse`, `Execute`, `Fetch`) e consumiu 7 buffers de leitura em memoria, elevando a contagem de instrucoes do trace para 3 declaracoes (104 linhas) e onerando a fase de `Parse` da consulta principal em 8 buffers adicionais (`query=17` no total).

### 4.2. A Execucao Limpa e Determinada

Para eliminar o trabalho recursivo e obter um perfil 100% focado no codigo da aplicacao, adotou-se o seguinte protocolo formal:

1. **Alimentacao Previa do Dicionario:** Execucao de `DBMS_STATS.gather_table_stats` para as tabelas `ENTREGAS` e `EVENTOS_RASTREIO` logo apos o `COMMIT`.
2. **Identificador Exclusivo de Rastreamento:** Configuracao de `tracefile_identifier = 'Consulta_limpa'`, facilitando a localizacao do arquivo sem colisao de nomes.
3. **Ativacao Estrita do Evento 10046 Nivel 12:** Instrumentacao direta no nivel do kernel do processo servidor.
4. **Desativacao e Encerramento Formal:** Desligamento do evento 10046 seguido de `EXIT` da sessao SQL*Plus, garantindo o descarregamento (*flush*) completo dos buffers do trace no disco.
5. **Compilacao Padronizada com TKPROF:** Processamento direto no diretorio oficial de trace com os parametros `sys=no waits=yes sort=prsela,exeela,fchela`.

```sql
-- Protocolo de execucao limpa
EXEC DBMS_STATS.gather_table_stats('HR', 'ENTREGAS');
EXEC DBMS_STATS.gather_table_stats('HR', 'EVENTOS_RASTREIO');

ALTER SESSION SET tracefile_identifier = 'Consulta_limpa';
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

SELECT e.codigo_rastreio, e.destinatario, ev.status, ev.localizacao,
       TO_CHAR(ev.data_hora, 'DD/MM/YYYY HH24:MI') as data_atualizacao
FROM entregas e
JOIN eventos_rastreio ev ON e.id_entrega = ev.id_entrega
WHERE e.codigo_rastreio = 'BR987654321US'
ORDER BY ev.data_hora DESC;

ALTER SESSION SET EVENTS '10046 trace name context off';
SELECT value FROM v$diag_info WHERE name = 'Default Trace File';
EXIT;
```

---

## 5. Analise Comparativa dos Perfis de Rastreamento (TKPROF)

A comparacao entre o perfil com amostragem dinamica (`Profile_Rastreio_em_17112026.txt`) e o perfil refinado (`Profile_Limpo_20112026.txt`) comprova o isolamento da medicao:

| Metrica de Desempenho | Teste 1 (Com Dynamic Sampling) | Teste 2 (Execucao Limpa) | Diferenca / Impacto |
| :--- | :--- | :--- | :--- |
| Instrucoes no Trace File | 3 instrucoes (104 linhas) | 2 instrucoes (51 linhas) | Reducao de 51% no volume do trace |
| Instrucoes SQL Recursivas | 1 instrucao (`OPT_DYN_SAMP`) | 0 instrucoes | Eliminacao total de codigo acessorio |
| Chamadas Recursivas (Total) | 3 chamadas / 7 buffers | 0 chamadas / 0 buffers | Zero interferencia do CBO em runtime |
| Leituras Logicas no Parse | 8 buffers consistentes (`query=8`) | 0 buffers consistentes (`query=0`) | Otimizacao imediata via metadados |
| Leituras Logicas no Fetch | 9 buffers consistentes (`query=9`) | 9 buffers consistentes (`query=9`) | Consumo identico e deterministico |
| Leituras Logicas Totais (LIO) | 17 buffers consistentes (`query=17`) | 9 buffers consistentes (`query=9`) | Reducao de 47% nas consultas a memoria |
| Leituras Fisicas em Disco (PIO)| 0 blocos (`disk=0`) | 0 blocos (`disk=0`) | Operacao 100% em buffer cache |

### 5.1. Decomposicao Detalhada do Perfil Limpo

A saida do TKPROF no teste limpo reflete estritamente o trabalho executado pela consulta analitica:

```text
call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch        2      0.00       0.00          0          9          0           4
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        4      0.00       0.00          0          9          0           4

Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 85
```

* **Parse com Custo Zero em Memoria:** Ao dispor previamente de estatisticas atualizadas, o CBO compilou a consulta consumindo `query=0` buffers, registrando apenas a insercao no Library Cache (`Misses in library cache during parse: 1`).
* **Fetch e Matriz de Leitura (*Array Fetch*):** Foram necessarias 2 chamadas de `Fetch` para retornar 4 linhas. A primeira recuperou o conjunto de dados e a segunda confirmou o final dos registros (`End of Fetch`), em conformidade com o comportamento padrao do cliente SQL*Plus.

### 5.2. Plano de Execucao Fisico Otimizado

```text
Rows     Row Source Operation
-------  ---------------------------------------------------
      4  SORT ORDER BY (cr=9 pr=0 pw=0 time=0 us cost=5 size=282 card=3)
      4   NESTED LOOPS  (cr=9 pr=0 pw=0 time=3 us cost=4 size=282 card=3)
      1    TABLE ACCESS BY INDEX ROWID ENTREGAS (cr=2 pr=0 pw=0 time=0 us cost=1 size=35 card=1)
      1     INDEX UNIQUE SCAN SYS_C0011097 (cr=1 pr=0 pw=0 time=0 us cost=0 size=0 card=1)(object id 74581)
      4    TABLE ACCESS FULL EVENTOS_RASTREIO (cr=7 pr=0 pw=0 time=3 us cost=3 size=177 card=3)
```

1. **`INDEX UNIQUE SCAN SYS_C0011097`:** Validacao da chave de unicidade da entrega em apenas 1 buffer consistente (`cr=1`) com custo zero (`cost=0`).
2. **`TABLE ACCESS BY INDEX ROWID ENTREGAS`:** Resgate da linha mestre consumindo 1 buffer adicional, totalizando 2 buffers (`cr=2`).
3. **`TABLE ACCESS FULL EVENTOS_RASTREIO`:** Varredura completa da tabela dependente restrita a 7 buffers (`cr=7`), associando os 4 eventos vinculados a chave estrangeira.
4. **`NESTED LOOPS` e `SORT ORDER BY`:** A juncao e a ordenacao temporal (`ORDER BY ev.data_hora DESC`) operaram inteiramente no espaco de memoria da PGA, acumulando rigorosamente 9 buffers de leitura consistente (`cr=9`) e zero leituras fisicas (`pr=0`, `pw=0`).

### 5.3. Principio LIO versus PIO (Cary Millsap)

A preservacao de `disk=0` em ambas as execucoes comprova que a carga operou inteiramente em memoria RAM (*Logical I/O*). No entanto, o teste limpo evidencia que a reducao de leituras logicas (de 17 para 9 buffers) elimina ciclos de CPU desnecessarios e mitiga a concorrencia em travas de memoria (*latches/mutexes* do cache de blocos), comprovando a tese de Millsap de que a otimizacao de software deve focar primariamente na contagem de chamadas e em leituras logicas, e nao exclusivamente no monitoramento de discos fisicos.

---

## 6. Estrutura do Repositorio

A organizacao do projeto segue padrao modular para facil navegacao e auditoria tecnica:

```text
├── sql/
│   ├── 01_criacao_tabelas.sql            # DDL das tabelas entregas e eventos_rastreio
│   ├── 02_carga_dados.sql                # Insercoes de dados ficticios de teste
│   └── 03_consulta_rastreada_limpa.sql   # Coleta de estatisticas, evento 10046 e query
├── traces/
│   ├── oracle_ora_4124_teste_rastreio.trc # Trace bruto do Teste 1 (com Dynamic Sampling)
│   ├── Profile_Rastreio_em_17112026.txt   # Relatorio TKPROF do Teste 1
│   ├── oracle_ora_1000_Consulta_limpa.trc # Trace bruto do Teste 2 (Execucao Limpa)
│   └── Profile_Limpo_20112026.txt         # Relatorio TKPROF do Teste 2 Limpo
├── docs/
│   └── comandos-sql-cenario1.md          # Roteiro formal de execucao e catalogo de comandos
├── scripts/
│   └── gerar_profile_limpo.bat           # Script de automacao da compilacao com TKPROF
└── README.md                             # Documentacao metodologica e analise cientifica
```

---

## 7. Guia de Reprodutibilidade Passo a Passo

Para reproduzir integralmente o ensaio limpo em qualquer ambiente Oracle Database:

### Passo 1: Execucao da Carga e Rastreamento no SQL*Plus
Conecte-se com a conta de aplicacao e execute o ciclo completo:

```sql
sqlplus hr/SenhaBancoOracle@localhost:1521/oracle

-- Coleta de estatisticas
EXEC DBMS_STATS.gather_table_stats('HR', 'ENTREGAS');
EXEC DBMS_STATS.gather_table_stats('HR', 'EVENTOS_RASTREIO');

-- Habilitacao do rastreamento estendido
ALTER SESSION SET tracefile_identifier = 'Consulta_limpa';
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

-- Consulta monitorada
SELECT e.codigo_rastreio, e.destinatario, ev.status, ev.localizacao,
       TO_CHAR(ev.data_hora, 'DD/MM/YYYY HH24:MI') as data_atualizacao
FROM entregas e
JOIN eventos_rastreio ev ON e.id_entrega = ev.id_entrega
WHERE e.codigo_rastreio = 'BR987654321US'
ORDER BY ev.data_hora DESC;

-- Desligamento e saida
ALTER SESSION SET EVENTS '10046 trace name context off';
EXIT;
```

### Passo 2: Processamento com TKPROF no Diretorio Oficial
Abra o Prompt de Comando do Windows (CMD) e navegue diretamente ate o diretorio oficial de trace para evitar dispersao de arquivos:

```cmd
cd C:\oracle\diag\rdbms\oracle\oracle\trace

tkprof oracle_ora_1000_Consulta_limpa.trc Profile_Limpo_20112026.txt sys=no waits=yes sort=prsela,exeela,fchela
```

---

## 8. Conclusoes e Proximos Passos (Cenario 2)

O refinamento experimental demonstrou que a eliminacao de ruídos internos no SGBD e requisito fundamental para a validade de perfis de desempenho:

1. **Determinismo nas Metricas:** Sem amostragem dinamica, cada fracao de tempo e cada buffer contabilizado refletem exatamente o custo da logica de negocio.
2. **Impacto da Modelagem:** O uso de indice unico na chave de busca e integridade referencial viabilizou um plano de execucao estavel com apenas 9 leituras em memoria para recuperar a cadeia de eventos.
3. **Avanco para a Fase 2:** Com a linha de base limpa homologada, o proximo estagio do projeto consiste na construcao de um experimento com loop de repeticoes uniformes utilizando variaveis de ligacao (*bind variables*), permitindo avaliar a distribuicao estatistica da latencia, dispersao temporal e percentis de tempo de resposta conforme formulado na segunda parte do trabalho de Cary Millsap.