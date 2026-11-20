# Instrumentacao de Software e Analise de Desempenho com Oracle Database: Aplicacao Pratica do Metodo de Cary Millsap

Este repositorio documenta um laboratorio experimental de engenharia de desempenho de software fundamentado nos principios metodologicos descritos por Cary Millsap no artigo *Thinking Clearly About Performance* (publicado na *Communications of the ACM* e na *ACM Queue* em 2010).

O objetivo do projeto e demonstrar, por meio de dados empiricos de rastreamento e perfilamento em banco de dados relacional, a substituicao da abordagem tradicional de tentativa e erro (*trial-and-error*) por uma metodologia axiomatica baseada na decomposicao matematica do tempo de resposta.
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

---

## 3. Modelo de Dados e Carga de Teste

O cenario de testes simula um subsistema de logistica e rastreamento de encomendas internacionais, composto por duas entidades relacionais ligadas por restricao de integridade referencial:

1. **`entregas`**: Tabela mestre que armazena os dados cadastrais da encomenda, incluindo identificador primario, codigo de rastreamento com restricao de unicidade (`UNIQUE NOT NULL`), remetente e destinatario.
2. **`eventos_rastreio`**: Tabela de detalhe que registra o historico de movimentacao fisica da encomenda, referenciando a tabela mestre via chave estrangeira (`FOREIGN KEY`) e registrando status, localizacao geografica e carimbo de data/hora (`TIMESTAMP`).

```sql
-- DDL de criacao das estruturas
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

Os dados inseridos no experimento representam postagens ficticias e eventos operacionais simulando o fluxo de envio entre agencias locais e alfandega internacional.

---

## 4. Metodologia de Instrumentacao (Extended SQL Trace)

Para capturar o tempo de execucao sem interferencia de agregacoes estatisticas externas, a sessao do usuario foi instrumentada no nivel do processo servidor do Oracle utilizando a tecnologia de rastreamento estendido.

A ativacao ocorreu por meio do evento diagnostico `10046` no **Nivel 12**, que habilita a coleta de:
* Instrucoes SQL individuais emitidas pela sessao;
* Decomposicao detalhada dos estagios de processamento (`Parse`, `Execute` e `Fetch`);
* Valores reais das variaveis de ligacao (*bind variables*);
* Tempos e duracoes exatas de cada evento de espera do sistema operacional e de rede.

```sql
-- Marcacao do arquivo de saida no diretorio de diagnostico
ALTER SESSION SET tracefile_identifier = 'teste_rastreio';

-- Ativacao da instrumentacao detalhada (Nivel 12)
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

-- Execucao da carga de trabalho analitica monitorada
SELECT e.codigo_rastreio, e.destinatario, ev.status, ev.localizacao,
       TO_CHAR(ev.data_hora, 'DD/MM/YYYY HH24:MI') as data_atualizacao
FROM entregas e
JOIN eventos_rastreio ev ON e.id_entrega = ev.id_entrega
WHERE e.codigo_rastreio = 'BR987654321US'
ORDER BY ev.data_hora DESC;

-- Desativacao imediata do rastreamento
ALTER SESSION SET EVENTS '10046 trace name context off';
```

A localizacao do arquivo bruto gerado (`.trc`) foi identificada dinamicamente consultando a visao de diagnostico:

```sql
SELECT value FROM v$diag_info WHERE name = 'Default Trace File';
```

---

## 5. Analise Empirica dos Perfis de Rastreamento (TKPROF)

O arquivo bruto de rastreamento (`oracle_ora_4124_teste_rastreio.trc`) foi processado pelo utilitario **TKPROF**, gerando o perfil tabular consolidado (`Profile_Rastreio_em_17112026.txt`). A analise das metricas coletadas revela aspectos cruciais sobre a execucao fisica e o comportamento do otimizador de consultas.

### 5.1. Decomposicao da Consulta Analitica Principal

A consulta rastreada executou a juncao entre a entrega identificada pelo codigo `BR987654321US` e seus respectivos eventos de rastreamento, retornando quatro registros ordenados de forma decrescente:

```text
call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.01       0.01          0          8          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch        2      0.00       0.00          0          9          0           4
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        4      0.01       0.01          0         17          0           4
```

A analise da decomposicao evidencia:

* **Ciclo de Chamadas (*Call Count*):** A execucao exigiu 1 fase de analise sintatica e semantica (`Parse`), 1 disparo de execucao (`Execute`) e 2 ciclos de recuperacao de blocos (`Fetch`). A necessidade de 2 chamadas de `Fetch` para recuperar 4 linhas decorre do tamanho padrao da matriz de leitura (*fetch array size*) do cliente SQL*Plus, no qual a ultima chamada sinaliza o termino dos registros (`End of Fetch`).
* **Consumo de CPU vs. Tempo Decorrido:** O tempo de CPU totalizou 0,01 segundos e coincidiu com o tempo decorrido total (0,01s), demonstrando ausencia de gargalos de concorrencia ou contencao por recursos compartilhados durante o processamento da instrucao.

### 5.2. Analise do Plano de Execucao Fisico

O perfil gerado pelo TKPROF documenta as operacoes internas executadas pelo mecanismo de processamento de dados:

```text
Rows     Row Source Operation
-------  ---------------------------------------------------
      4  SORT ORDER BY (cr=9 pr=0 pw=0 time=0 us cost=5 size=546 card=3)
      4   NESTED LOOPS  (cr=9 pr=0 pw=0 time=3 us cost=4 size=546 card=3)
      1    TABLE ACCESS BY INDEX ROWID ENTREGAS (cr=2 pr=0 pw=0 time=0 us cost=1 size=77 card=1)
      1     INDEX UNIQUE SCAN SYS_C0011097 (cr=1 pr=0 pw=0 time=0 us cost=1 size=0 card=1)(object id 74581)
      4    TABLE ACCESS FULL EVENTOS_RASTREIO (cr=7 pr=0 pw=0 time=3 us cost=3 size=315 card=3)
```

1. **Acesso por Indice Unico (`INDEX UNIQUE SCAN`):** O predicado `WHERE e.codigo_rastreio = 'BR987654321US'` utilizou o indice unico `SYS_C0011097` gerado pela restricao de chave unica da tabela `entregas`. Essa operacao custou exatamente 1 leitura em memoria (`cr=1`) para localizar a chave e o respectivo endereco fisico de linha (`ROWID`).
2. **Acesso a Tabela Mestre (`TABLE ACCESS BY INDEX ROWID`):** Com o `ROWID` em maos, o banco recuperou a linha correspondente consumindo 1 buffer adicional, totalizando 2 leituras consistentes (`cr=2`).
3. **Juncao em Laco Aninhado (`NESTED LOOPS`):** Para a unica entrega identificada, o banco realizou a varredura completa da tabela secundaria (`TABLE ACCESS FULL EVENTOS_RASTREIO`) consumindo 7 buffers em memoria (`cr=7`), associando as 4 linhas filhas.
4. **Ordenacao em Memoria (`SORT ORDER BY`):** A ordenacao final por data e hora decrescente (`ORDER BY ev.data_hora DESC`) foi concluida sem alocacao de espaco em disco temporario.

### 5.3. Eficiencia de Memoria: Leituras Logicas (LIO) vs. Fisicas (PIO)

Em todas as chamadas analisadas, a coluna `disk` (leituras fisicas em subsistema de armazenamento secundario) manteve-se estritamente em **0**.

Todo o trabalho computacional foi realizado via leituras logicas em memoria RAM (*Logical I/O - LIO*), contabilizadas nas colunas `query=17` e `current=0`. Esse comportamento ilustra o principio defendido por Cary Millsap de que, mesmo quando os blocos residem no *Buffer Cache*, cada leitura logica consome ciclos de CPU e requer o estabelecimento de travas em estruturas de controle (*latches* e *mutexes*). Portanto, a metrica prioritaria de reducao em otimizacao de software deve ser o numero absoluto de blocos examinados, e nao apenas o monitoramento de I/O de disco.

### 5.4. Amostragem Dinamica (*Dynamic Sampling*) Detectada

O rastreamento revelou a execucao automatica de uma instrucao SQL recursiva gerada internamente pelo motor do banco:

```sql
SELECT /* OPT_DYN_SAMP */ /*+ ALL_ROWS IGNORE_WHERE_CLAUSE ... */ 
  NVL(SUM(C1),:"SYS_B_0"), NVL(SUM(C2),:"SYS_B_1"), COUNT(DISTINCT C3) ...
FROM (SELECT /*+ NO_PARALLEL("EV") FULL("EV") ... */ ... FROM "EVENTOS_RASTREIO" "EV") SAMPLESUB;
```

**Diagnostico Tecnico:**
Como as tabelas haviam acabado de ser criadas e populadas sem a execucao previa do pacote de coleta de estatisticas (`DBMS_STATS.GATHER_TABLE_STATS`), o Otimizador Baseado em Custo (*Cost-Based Optimizer - CBO*) detectou ausencia de metadados no dicionario de dados.

Para estimar a cardinalidade e selecionar o plano de acesso mais adequado, o CBO realizou uma amostragem dinamica em tempo de execucao, consumindo 1 chamada de `Parse`, 1 de `Execute`, 1 de `Fetch` e 7 leituras logicas (`query=7`).

Em ambientes de producao, a coleta tempestiva de estatisticas evita essa sobrecarga recursiva e garante planos de execucao estaveis.

---

## 6. Estrutura do Repositorio

Para garantir organizacao e auditabilidade, os artefatos do projeto estao dispostos na seguinte estrutura:

```text
├── sql/
│   ├── 01_criacao_tabelas.sql        # DDL das tabelas entregas e eventos_rastreio
│   ├── 02_carga_dados.sql            # Scripts de carga com dados ficticios
│   └── 03_consulta_rastreada.sql     # Habilitacao do evento 10046 e query de teste
├── traces/
│   ├── oracle_ora_4124_teste_rastreio.trc # Arquivo bruto de rastreamento gerado pelo Oracle
│   └── Profile_Rastreio_em_17112026.txt   # Relatorio formatado e decomposto pelo TKPROF
├── scripts/
│   └── gerar_profile.bat             # Automacao do comando TKPROF via terminal
└── README.md                         # Documentacao metodologica e analise dos resultados
```

---

## 7. Guia de Reprodutibilidade Passo a Passo

Para reproduzir este experimento em qualquer ambiente compativel com Oracle Database (instalacao local no Windows, maquina virtual Linux ou conteiner Docker):

### Passo 1: Criacao e Carga do Esquema
Conecte-se ao banco de dados com um usuario com privilegios de criacao de objetos e execute os scripts de estrutura e dados:

```cmd
sqlplus usuario/senha@localhost:1521/XE @sql/01_criacao_tabelas.sql
sqlplus usuario/senha@localhost:1521/XE @sql/02_carga_dados.sql
```

### Passo 2: Execucao com Rastreamento Ativo
Execute o script instrumentado que liga o evento 10046, processa a consulta analitica e desativa a captura:

```cmd
sqlplus usuario/senha@localhost:1521/XE @sql/03_consulta_rastreada.sql
```

O script reportara o caminho absoluto do arquivo `.trc` gerado no sistema de arquivos da maquina servidora.

### Passo 3: Geracao do Perfil com TKPROF
Abra o Prompt de Comando (com privilegios de leitura no diretorio de rastreamento do Oracle) e execute o utilitario TKPROF para compilar o arquivo de trace bruto em formato legivel:

```cmd
tkprof "c:\oracle\diag\rdbms\oracle\oracle\trace\oracle_ora_4124_teste_rastreio.trc" "Profile_Rastreio.txt" sys=no waits=yes sort=prsela,exeela,fchela
```

Os parametros utilizados garantem:
* `sys=no`: Ocultacao de chamadas internas recursivas da administracao do sistema (mantendo o foco na carga da aplicacao);
* `waits=yes`: Inclusao detalhada da tabela de eventos de espera do sistema e da rede;
* `sort=prsela,exeela,fchela`: Ordenacao decrescente baseada no tempo decorrido total das instrucoes, em total conformidade com a aplicacao da Lei de Amdahl.

---

## 8. Conclusoes e Consideracoes Metodologicas

A realizacao pratica deste laboratorio confirma as premissas estabelecidas na literatura de engenharia de desempenho de software:

1. **Rastreamento Deterministico supera Palpites:** A captura via evento 10046 isolou o comportamento real da sessao sem interferencia de medias globais do sistema, demonstrando onde cada microssegundo foi gasto.
2. **Eficiencia no Acesso aos Dados:** A presenca do indice unico na chave de rastreamento restringiu o acesso inicial a apenas 2 blocos logicos (`cr=2`), evitando que o banco percorresse a tabela completa para identificar o remetente e o destinatario.
3. **Auditabilidade e Reprodutibilidade:** O perfil tabular gerado pelo TKPROF fornece uma base de evidencias objetiva e auditavel para orientar intervencoes de engenharia de software e arquitetura de banco de dados, atendendo aos padroes de rigor cientifico exigidos em pesquisas de pos-graduacao.
