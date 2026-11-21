# Instrumentação de Software e Análise de Desempenho com Oracle Database: Aplicação Prática do Método de Cary Millsap

Este repositório documenta um laboratório experimental de engenharia de desempenho de software fundamentado nos princípios metodológicos descritos por Cary Millsap no artigo *Thinking Clearly About Performance* (publicado na *Communications of the ACM* e na *ACM Queue* em 2010).

O objetivo do projeto é demonstrar, por meio de dados empíricos de rastreamento e perfilamento em banco de dados relacional, a substituição da abordagem tradicional de tentativa e erro (*trial-and-error*) por uma metodologia axiomática baseada na decomposição matemática do tempo de resposta. O laboratório evoluiu de uma medição preliminar (na qual se diagnosticou a presença de consultas recursivas de amostragem dinâmica) para uma execução rigorosamente limpa e isolada, em total conformidade com a literatura de engenharia de desempenho.

---

## 1. Fundamentação Teórica: O Modelo de Cary Millsap

A engenharia de desempenho de sistemas computacionais frequentemente sofre com práticas empíricas não estruturadas, nas quais alterações de configuração ou índices são introduzidas sem comprovação causal prévia. Conforme postulado por Millsap, otimizar sistemas por tentativa e erro é inviável em arquiteturas multicamadas complexas. A análise formal exige medição determinística e auditável do consumo de tempo.

### 1.1. Decomposição do Tempo de Resposta

O tempo de resposta ($R$) percebido por uma transação ou tarefa de usuário é decomposto no somatório do tempo de serviço ($S$), no qual recursos computacionais (como CPU) estão efetivamente processando instruções, e do tempo de espera ($W$), no qual o processo aguarda a liberação de recursos de infraestrutura, bloqueios de concorrência ou transferências de rede:

$$R = S + W$$

Em arquiteturas cliente-servidor ou sistemas multicamadas, a execução de uma operação de negócio gera múltiplos intercâmbios de mensagens e chamadas procedurais entre a aplicação e o subsistema de persistência.

### 1.2. Limitação dos Diagramas de Sequência e Necessidade do Perfil

Embora diagramas de sequência UML sejam úteis para conceituar fluxos de interação em pequena escala, eles tornam-se ineficazes para diagnosticar tarefas reais que executam milhares de operações atômicas. A representação puramente visual não escala diante de volumes massivos de chamadas repetitivas.

Para resolver essa limitação, a metodologia preconiza a construção do Perfil de Tempo de Resposta (*Resource Profile*): uma tabela ordenada de forma decrescente na qual cada linha representa o consumo de tempo acumulado por categoria de chamada ou evento de espera. O perfil evidencia com clareza matemática onde o tempo foi consumido e onde não foi consumido, orientando a priorização das intervenções com base na Lei de Amdahl.

---

## 2. Arquitetura do Experimento e Ambiente Operacional

O laboratório foi executado em ambiente controlado sob sistema operacional Microsoft Windows de 64 bits, utilizando uma instância local do Oracle Database configurada para instrumentação estendida de sessão.

| Componente | Especificação |
| :--- | :--- |
| Sistema Operacional | Microsoft Windows x64 (Compilação 10.0.26200.9974) |
| Sistema Gerenciador de Banco de Dados | Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 (64-bit Production) |
| Ferramentas de Linha de Comando | Oracle SQL*Plus Release 11.2.0.1.0, Oracle TKPROF Release 11.2.0.1.0 |
| Mecanismo de Instrumentação | Evento 10046 Nível 12 (Extended SQL Trace com variáveis de ligação e esperas) |
| Identificador da Instância (SID) | oracle |
| Diretório Padrão de Diagnóstico | C:\oracle\diag\rdbms\oracle\oracle\trace |

---

## 3. Modelo de Dados e Carga de Teste

O cenário de testes simula um subsistema de logística e rastreamento de encomendas internacionais, composto por duas entidades relacionais ligadas por restrição de integridade referencial:

1. **`entregas`**: Tabela mestre que armazena os dados cadastrais da encomenda, incluindo identificador primário, código de rastreamento com restrição de unicidade (`UNIQUE NOT NULL`), remetente e destinatário.
2. **`eventos_rastreio`**: Tabela de detalhe que registra o histórico de movimentação física da encomenda, referenciando a tabela mestre via chave estrangeira (`FOREIGN KEY`) e registrando status, localização geográfica e carimbo de data e hora (`TIMESTAMP`).

```sql
-- DDL de criação das estruturas relacionais
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
VALUES (1, 'BR123456789BR', 'Loja Central', 'José Roberto');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (1, 1, 'Postado', 'Agência Sorocaba');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (2, 1, 'Em Trânsito', 'Centro de Distribuição SP');

INSERT INTO entregas (id_entrega, codigo_rastreio, remetente, destinatario)
VALUES (2, 'BR987654321US', 'José Roberto', 'Filial California, USA');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (3, 2, 'Postado (Envio Internacional)', 'Agência Central - Sorocaba');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (4, 2, 'Chegado ao Aeroporto', 'Aeroporto de Guarulhos - SP');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (5, 2, 'Liberado na Alfândega', 'Alfândega Americana - CA');

INSERT INTO eventos_rastreio (id_evento, id_entrega, status, localizacao)
VALUES (6, 2, 'Entregue ao Destinatário', 'Filial California, USA');

COMMIT;
```

---

## 4. Evolução Metodológica: Da Medição com Ruído à Execução Limpa

A condução do experimento ocorreu em dois momentos distintos, ilustrando a importância do controle de variáveis de contorno em testes de software:

### 4.1. O Primeiro Teste e o Diagnóstico da Amostragem Dinâmica

No primeiro ensaio (`Profile_Rastreio_em_17112026.txt`), a consulta foi submetida imediatamente após a inserção dos dados, sem a coleta prévia de metadados estatísticos no dicionário de dados. O Otimizador Baseado em Custo (*Cost-Based Optimizer - CBO*) detectou ausência de estatísticas e disparou uma consulta recursiva interna:

```sql
SELECT /* OPT_DYN_SAMP */ /*+ ALL_ROWS IGNORE_WHERE_CLAUSE ... */ 
  NVL(SUM(C1),:"SYS_B_0"), NVL(SUM(C2),:"SYS_B_1"), COUNT(DISTINCT C3) ... 
FROM (SELECT /*+ NO_PARALLEL("EV") FULL("EV") ... */ ... FROM "EVENTOS_RASTREIO" "EV") SAMPLESUB;
```

Essa operação acessória gerou 3 chamadas recursivas (`Parse`, `Execute`, `Fetch`) e consumiu 7 buffers de leitura em memória, elevando a contagem de instruções do trace para 3 declarações (104 linhas) e onerando a fase de `Parse` da consulta principal em 8 buffers adicionais (`query=17` no total).

### 4.2. O Protocolo da Execução Limpa e Determinística

Para eliminar o trabalho recursivo e obter um perfil estritamente focado no código da aplicação, adotou-se o seguinte protocolo formal:

1. **Alimentação Prévia do Dicionário de Dados:** Execução de `DBMS_STATS.gather_table_stats` para as tabelas `ENTREGAS` e `EVENTOS_RASTREIO` logo após o `COMMIT`.
2. **Identificador Exclusivo de Rastreamento:** Configuração de `tracefile_identifier = 'Consulta_limpa'`, facilitando a localização do arquivo sem colisão de nomes.
3. **Ativação Estrita do Evento 10046 Nível 12:** Instrumentação direta no nível do kernel do processo servidor.
4. **Desativação e Encerramento Formal:** Desligamento do evento 10046 seguido do comando `EXIT` na sessão do SQL*Plus, garantindo o descarregamento (*flush*) completo dos buffers de rastreamento no disco.
5. **Compilação Padronizada com TKPROF:** Processamento direto no diretório oficial de rastreamento com os parâmetros `sys=no waits=yes sort=prsela,exeela,fchela`.

```sql
-- Protocolo de execução limpa
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

## 5. Análise Comparativa dos Perfis de Rastreamento (TKPROF)

A comparação entre o perfil com amostragem dinâmica (`Profile_Rastreio_em_17112026.txt`) e o perfil refinado (`Profile_Limpo_20112026.txt`) comprova o isolamento da medição:

| Métrica de Desempenho | Teste 1 (Com Dynamic Sampling) | Teste 2 (Execução Limpa) | Diferença e Impacto |
| :--- | :--- | :--- | :--- |
| Instruções no Arquivo de Trace | 3 instruções (104 linhas) | 2 instruções (51 linhas) | Redução de 51% no volume do trace |
| Instruções SQL Recursivas | 1 instrução (`OPT_DYN_SAMP`) | 0 instruções | Eliminação total de código acessório |
| Chamadas Recursivas (Total) | 3 chamadas / 7 buffers | 0 chamadas / 0 buffers | Zero interferência do CBO em tempo de execução |
| Leituras Lógicas no Parse | 8 buffers consistentes (`query=8`) | 0 buffers consistentes (`query=0`) | Otimização imediata via metadados |
| Leituras Lógicas no Fetch | 9 buffers consistentes (`query=9`) | 9 buffers consistentes (`query=9`) | Consumo idêntico e determinístico |
| Leituras Lógicas Totais (LIO) | 17 buffers consistentes (`query=17`) | 9 buffers consistentes (`query=9`) | Redução de 47% nas consultas à memória RAM |
| Leituras Físicas em Disco (PIO)| 0 blocos (`disk=0`) | 0 blocos (`disk=0`) | Operação 100% no Buffer Cache |

### 5.1. Decomposição Detalhada do Perfil Limpo

A saída do TKPROF no teste limpo reflete estritamente o trabalho executado pela consulta analítica:

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

* **Parse com Custo Zero em Memória:** Ao dispor previamente de estatísticas atualizadas, o CBO compilou a consulta consumindo `query=0` buffers, registrando apenas a inserção inicial no Library Cache (`Misses in library cache during parse: 1`).
* **Fetch e Matriz de Leitura (*Array Fetch*):** Foram necessárias 2 chamadas de `Fetch` para retornar 4 linhas. A primeira recuperou o conjunto de dados e a segunda confirmou o encerramento dos registros (`End of Fetch`), em conformidade com o comportamento padrão do cliente SQL*Plus.

### 5.2. Plano de Execução Físico Otimizado

```text
Rows     Row Source Operation
-------  ---------------------------------------------------
      4  SORT ORDER BY (cr=9 pr=0 pw=0 time=0 us cost=5 size=282 card=3)
      4   NESTED LOOPS  (cr=9 pr=0 pw=0 time=3 us cost=4 size=282 card=3)
      1    TABLE ACCESS BY INDEX ROWID ENTREGAS (cr=2 pr=0 pw=0 time=0 us cost=1 size=35 card=1)
      1     INDEX UNIQUE SCAN SYS_C0011097 (cr=1 pr=0 pw=0 time=0 us cost=0 size=0 card=1)(object id 74581)
      4    TABLE ACCESS FULL EVENTOS_RASTREIO (cr=7 pr=0 pw=0 time=3 us cost=3 size=177 card=3)
```

1. **`INDEX UNIQUE SCAN SYS_C0011097`:** Validação da chave de unicidade da entrega em apenas 1 buffer consistente (`cr=1`) com custo zero (`cost=0`).
2. **`TABLE ACCESS BY INDEX ROWID ENTREGAS`:** Resgate da linha mestre consumindo 1 buffer adicional, totalizando 2 buffers (`cr=2`).
3. **`TABLE ACCESS FULL EVENTOS_RASTREIO`:** Varredura completa da tabela dependente restrita a 7 buffers (`cr=7`), associando os 4 eventos vinculados à chave estrangeira.
4. **`NESTED LOOPS` e `SORT ORDER BY`:** A junção e a ordenação temporal (`ORDER BY ev.data_hora DESC`) operaram inteiramente no espaço de memória da PGA, acumulando rigorosamente 9 buffers de leitura consistente (`cr=9`) e zero leituras físicas (`pr=0`, `pw=0`).

### 5.3. Princípio LIO versus PIO (Cary Millsap)

A preservação de `disk=0` em ambas as execuções comprova que a carga operou inteiramente em memória RAM (*Logical I/O*). No entanto, o teste limpo evidencia que a redução de leituras lógicas (de 17 para 9 buffers) elimina ciclos de CPU desnecessários e mitiga a concorrência em travas de controle de memória (*latches* e *mutexes* do cache de blocos). Esse resultado comprova a tese central de Millsap de que a otimização de software deve priorizar a contagem de chamadas e as leituras lógicas, e não focar exclusivamente em gargalos de disco físico.

---

## 6. Estrutura do Repositório

A organização do projeto segue padrão modular para facilitar a navegação e a auditoria técnica:

```text
├── sql/
│   ├── 01_criacao_tabelas.sql            # DDL das tabelas entregas e eventos_rastreio
│   ├── 02_carga_dados.sql                # Inserções de dados fictícios de teste
│   └── 03_consulta_rastreada_limpa.sql   # Coleta de estatísticas, evento 10046 e query
├── traces/
│   ├── oracle_ora_4124_teste_rastreio.trc # Trace bruto do Teste 1 (com Dynamic Sampling)
│   ├── Profile_Rastreio_em_17112026.txt   # Relatório TKPROF do Teste 1
│   ├── oracle_ora_1000_Consulta_limpa.trc # Trace bruto do Teste 2 (Execução Limpa)
│   └── Profile_Limpo_20112026.txt         # Relatório TKPROF do Teste 2 Limpo
├── docs/
│   ├── comandos-sql-cenario1.md          # Roteiro formal de execução e catálogo de comandos
│   └── observacoes-versao-oracle-2010.md # Análise histórica e técnica da release 11.2.0.1.0
├── scripts/
│   └── gerar_profile_limpo.bat           # Script de automação da compilação com TKPROF
└── README.md                             # Documentação metodológica e análise científica
```

---

## 7. Guia de Reprodutibilidade Passo a Passo

Para reproduzir integralmente o ensaio limpo em qualquer ambiente Oracle Database:

### Passo 1: Execução da Carga e Rastreamento no SQL*Plus
Conecte-se com a conta de aplicação e execute o ciclo completo:

```sql
sqlplus hr/SenhaBancoOracle@localhost:1521/oracle

-- Coleta prévia de estatísticas
EXEC DBMS_STATS.gather_table_stats('HR', 'ENTREGAS');
EXEC DBMS_STATS.gather_table_stats('HR', 'EVENTOS_RASTREIO');

-- Habilitação do rastreamento estendido
ALTER SESSION SET tracefile_identifier = 'Consulta_limpa';
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

-- Consulta monitorada
SELECT e.codigo_rastreio, e.destinatario, ev.status, ev.localizacao,
       TO_CHAR(ev.data_hora, 'DD/MM/YYYY HH24:MI') as data_atualizacao
FROM entregas e
JOIN eventos_rastreio ev ON e.id_entrega = ev.id_entrega
WHERE e.codigo_rastreio = 'BR987654321US'
ORDER BY ev.data_hora DESC;

-- Desligamento e saída formal
ALTER SESSION SET EVENTS '10046 trace name context off';
EXIT;
```

### Passo 2: Processamento com TKPROF no Diretório Oficial
Abra o Prompt de Comando do Windows (CMD) e navegue diretamente até o diretório oficial de trace para evitar dispersão de arquivos:

```cmd
cd C:\oracle\diag\rdbms\oracle\oracle\trace

tkprof oracle_ora_1000_Consulta_limpa.trc Profile_Limpo_20112026.txt sys=no waits=yes sort=prsela,exeela,fchela
```

---

## 8. Conclusões e Próximos Passos (Cenário 2)

O refinamento experimental demonstrou que a eliminação de ruídos internos no SGBD é requisito fundamental para a validade de perfis de desempenho:

1. **Determinismo nas Métricas:** Sem amostragem dinâmica, cada fração de tempo e cada buffer contabilizado refletem rigorosamente o custo da lógica de negócio.
2. **Impacto da Modelagem:** O uso de índice único na chave de busca e integridade referencial viabilizou um plano de execução estável com apenas 9 leituras em memória para recuperar a cadeia de eventos.
3. **Avanço para a Fase 2:** Com a linha de base limpa homologada, o próximo estágio do projeto consiste na construção de um experimento com loop de repetições uniformes utilizando variáveis de ligação (*bind variables*), permitindo avaliar a distribuição estatística da latência, dispersão temporal e percentis de tempo de resposta conforme formulado na segunda parte do trabalho de Cary Millsap.
