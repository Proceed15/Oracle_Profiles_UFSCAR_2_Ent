# Observacoes Tecnicas e Historicas sobre o Oracle Database Release 11.2.0.1.0 (2010) em Ambiente Windows 11

Este documento reune uma analise tecnica, arquitetural e epistemologica a respeito da utilizacao do Oracle Database 11g Release 2 (versao 11.2.0.1.0 de 64 bits) no laboratorio experimental de avaliacao de desempenho de software. O foco central destas observacoes reside no resgate do contexto historico da publicacao seminal de Cary Millsap em 2010, na comparacao entre as premissas conceituais da epoca e a execucao fisica contemporanea, bem como nos desafios de portabilidade e engenharia superados ao operar um sistema gerenciador de 2010 sobre a infraestrutura moderna do Microsoft Windows 11.

---

## 1. Contextualizacao Historica: A Concomitancia de 2010

A compreensao aprofundada dos dados coletados nos arquivos de rastreamento exige situar temporalmente a ferramenta utilizada e o referencial teorico adotado.

### 1.1. O Marco Editorial da Publicacao de Cary Millsap

O artigo *Thinking Clearly About Performance (Part 1)*, de autoria de Cary Millsap, foi publicado oficialmente em setembro de 2010 na revista *Communications of the ACM* (Volume 53, Numero 9, paginas 55 a 60), com desenvolvimento editorial coordenado pela revista *ACM Queue* (Volume 8, Numero 9), sendo complementado pela Parte 2 em outubro de 2010 (Volume 53, Numero 10, paginas 39 a 45). 

A publicacao integrou a secao *Practice* da ACM, concebida para aproximar a pesquisa em ciencia da computacao dos desafios reais da engenharia de software industrial. Millsap propôs uma formulacao axiomatica para a analise de desempenho, demonstrando que praticas baseadas em tentativa e erro (*trial-and-error*) falham sistematicamente em sistemas distribuidos complexos e devem ser substituidas pela decomposicao estrita do tempo de resposta.

### 1.2. A Versao Contemporanea do SGBD

No exato bimestre de publicacao do artigo (setembro e outubro de 2010), a versao mais recente, madura e amplamente distribuida do Oracle Database para plataformas Microsoft Windows (32 e 64 bits) era rigorosamente o **Oracle Database 11g Release 2, versao 11.2.0.1.0**. 

Segundo registros historicos da equipe de desenvolvimento e upgrade da Oracle Corporation, o pacote 11.2.0.1.0 para Windows foi disponibilizado publicamente em 6 de abril de 2010. O primeiro pacote cumulativo de correcoes da serie, o Patch Set 11.2.0.2 (distribuido sob o numero de patch 10098816 via My Oracle Support), embora lançado em setembro de 2010 para distribuicoes Linux, so alcancou a disponibilidade geral para sistemas Windows em 16 de dezembro de 2010.

Dessa forma, a instancia executada no experimento — identificada no cabecalho do utilitario TKPROF e no arquivo de alerta como *Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production* — representa uma reconstrucao fidedigna do estado da arte tecnologico existente no momento exato em que a metodologia de Millsap foi apresentada a comunidade academica internacional.

---

## 2. A Trajetoria do Autor e a Natureza do Artigo na ACM

Um ponto critico de esclarecimento metodologico refere-se ao vinculo institucional de Cary Millsap e a neutralidade tecnologica de sua formulacao teorica.

### 2.1. Desvinculacao da Oracle Corporation

Embora tenha iniciado sua trajetoria com o ecossistema Oracle em 1989 e liderado o *System Performance Group* da Oracle Corporation ao longo da decada de 1990, Cary Millsap desligou-se da corporacao em 1999. Na sequencia, cofundou a Hotsos (1999) e posteriormente fundou a Method R Corporation em 2008, empresa da qual era presidente quando redigiu os artigos da ACM em 2010.

Portanto, o artigo nao foi um comunicado de produto ou documento institucional da Oracle Corporation, mas sim uma producao cientifico-metodologica independente financiada por decadas de pesquisa em engenharia de sistemas.

### 2.2. Agnolicismo Conceitual versus Instrumentacao Fisica

O texto publicado na *Communications of the ACM* e intencionalmente generico e agnostico em relacao a fornecedores de banco de dados. Os diagramas de interacao utilizam nomenclaturas conceituais abstratas como `DB:prepare()`, `DB:execute()`, `DB:fetch()` e `App:await_db_netIO()`. 

Contudo, a comprovacao empirica e a instrumentacao mais granular do metodo derivam historicamente da arquitetura de instrumentacao do Oracle Database, especificamente do *Extended SQL Trace* (acessado via evento diagnostico 10046) e dos analisadores de perfil como o TKPROF e os softwares da Method R. Ao implementar o laboratorio utilizando o proprio Oracle Database 11.2.0.1.0, o experimento conecta os conceitos abstratos da ACM com a sua implementacao fisica primordial.

---

## 3. Especificacoes Tecnicas e Arquiteturais da Release 11.2.0.1.0

A versao 11.2.0.1.0 operou sob paradigmas arquiteturais que a distinguem de geracoes anteriores e posteriores do SGBD, refletindo o padrao computacional de 2010.

| Dimensao Arquitetural | Caracteristica na Release 11.2.0.1.0 (2010) | Contraste com Releases Modernas (12c a 26ai) |
| :--- | :--- | :--- |
| Modelo de Banco de Dados | Instancia Tradicional Autonoma (Instancia unica / Nao-Multitenant) | Arquitetura Multitenant obrigatoria (CDB e PDBs isolados) |
| Gerenciamento de Memoria | Gerenciamento Automatico Unificado de Memoria (`MEMORY_TARGET = 6432M`) | Suporte avancado a In-Memory Column Store e vetores SIMD |
| Otimizador de Consultas (CBO) | Otimizacao baseada em custo com Dynamic Sampling Nivel 2 padrao | SQL Plan Management avancado e Adaptive Query Optimization |
| Dicionario e Metadados | Estrutura relacional classica (`cdef$`, `hist_head$`, `obj$`) | Dicionarios particionados e metadados com suporte a Inteligencia Artificial |
| Protocolo de Trace | Extended SQL Trace Nivel 12 gravado diretamente em arquivo texto do SO | Rastreamento unificado compativel com Trace File Analyzer e Cloud Telemetry |

### 3.1. Ausencia de Multitenancy

A versao 11.2.0.1.0 e anterior a introducao da arquitetura Multitenant (que surgiu apenas em julho de 2013 com o Oracle Database 12c Release 1). Toda a execucao do experimento ocorre diretamente na instancia raiz (identificada pelo SID `oracle`), sem camadas intermediarias de encapsulamento como *Pluggable Databases* (PDBs). Esse modelo simplifica a analise do processo servidor no sistema operacional e garante que o consumo de buffers e tempo de CPU mensurado pelo TKPROF pertença exclusivamente a sessao ativa do usuario da aplicacao.

### 3.2. Gerenciamento Unificado de Memoria (AMM)

Conforme registrado nos arquivos de inicializacao e no log da instancia (`alert_oracle.log`), o banco operou com o parametro `memory_target = 6432M`. Essa configuracao habilita o *Automatic Memory Management* (AMM), recurso consolidado na linha 11g em que o proprio kernel do Oracle redistribui blocos dinamicamente entre a *System Global Area* (SGA - incluindo Buffer Cache e Shared Pool) e a *Program Global Area* (PGA) alocada para operacoes de ordenacao da sessao.

---

## 4. O Paradoxo Operacional: Executando um Software de 2010 no Windows 11

Um dos pontos mais notaveis do laboratorio experimental consiste no desacoplamento entre a data de compilacao dos binarios e o sistema hospedeiro: a execucao nativa de um software compilado em 2009/2010 sobre o Microsoft Windows 11 (compilacao de 64 bits 10.0.26200.9974).

### 4.1. Incompatibilidade Formal versus Suporte Operacional

Segundo a documentacao oficial da Oracle Corporation (*Operating System Checklist for Oracle Database Installation on Microsoft Windows*), o suporte comercial e a certificacao de conformidade para o Windows 11 x64 (restrito as edicoes Pro, Enterprise e Education) iniciaram-se exclusivamente a partir do **Oracle Database 19c Release Update 19.13** e do **Oracle Database 21c (versao 21.4)**. 

A familia Oracle 11gR2 (11.2.0.1 e 11.2.0.2) encerrou seu ciclo de vida util ha anos (*Unsupported / End-of-Life*) e teve sua certificacao formal limitada historicamente ao Windows Server 2003, Windows Server 2008, Windows Server 2008 R2 e edicoes desktop contemporaneas como Windows XP, Windows Vista e Windows 7. 

Nao obstante a total ausencia de suporte comercial, o experimento comprovou que os binarios de 64 bits do SGBD mantem compatibilidade operacional direta com as chamadas de sistema da API Win32/Win64 moderna, viabilizando a execucao estável do servico e a captura de traces.

### 4.2. Barreiras de Instalacao e Contornos Tecnicos Superados

A realizacao do teste exigiu a superacao de barreiras tecnicas documentadas pela comunidade especializada durante a instalacao nativa de versões legadas do *Oracle Universal Installer* (OUI):

1. **Falha de Reconhecimento de Versao do SO (`INS-13001`):** O utilitario de verificacao de ambiente do instalador (*Cluster Verification Utility - CVU*) nao reconhece identificadores de versao do kernel superiores ao Windows 6.1 (Windows 7), exigindo a execucao com o parametro de contorno `setup.exe -debug` em modo administrativo para contornar a checagem preliminar.
2. **Conflito de JVM e Fechamento Silencioso:** O descompactador temporario do instalador baseava-se em versoes legadas do Java SDK que sofrem bloqueios por mecanismos de seguranca preventiva do sistema hospedeiro.
3. **Bloqueios de Seguranca Ativa:** O servico de protecao em tempo real do Windows Defender e o Windows Firewall bloqueiam frequentemente as portas de escuta (`1521`) e o registro de servicos Windows (`OracleServiceORACLE` e `OracleJobSchedulerORACLE`).

A superacao desses entraves no ambiente experimental atesta rigor pratico de infraestrutura e viabilizou um ambiente de pesquisa reproduzivel sem custo de licenciamento adicional.

---

## 5. Analise do Motor Interno na Release 11.2.0.1.0

A observacao minuciosa dos perfis gerados pelo TKPROF (`Profile_Rastreio_em_17112026.txt` e `Profile_Limpo_20112026.txt`) fornece informacoes precisas sobre o comportamento algoritmico do SGBD de 2010.

### 5.1. Otimizador Baseado em Custo e Amostragem Dinamica

O fenomeno da amostragem dinamica observado no primeiro ensaio decorre da implementacao padrao do CBO no Oracle 11gR2. 

Quando tabelas sao criadas e povoadas sem estatisticas no dicionario, o parametro `optimizer_dynamic_sampling` (ativo no Nivel 2 por padrao na versao 11.2) instrui o otimizador a disparar uma consulta recursiva (`SELECT /* OPT_DYN_SAMP */ ...`) sobre 64 blocos aleatorios da tabela `EVENTOS_RASTREIO` para estimar sua cardinalidade. Na versao 11.2.0.1.0, esse calculo resultou em uma estimativa de cardinalidade (`card=82`) para a tabela de eventos, gerando o plano com `HASH GROUP BY` e elevando a contagem de instrucoes do trace para 3 declaracoes e 104 linhas.

No teste limpo, ao executar previamente o pacote `DBMS_STATS.gather_table_stats` na versao 11g, o dicionario de dados armazenou a cardinalidade real com precisao (`card=3`), eliminando o processo de amostragem em tempo de execucao, reduzindo o trace para 2 declaracoes e 51 linhas, e diminuindo o consumo de buffers lógicos no `Parse` de 8 buffers para zero.

### 5.2. O Utilitario TKPROF Release 11.2.0.1.0

O processamento do trace bruto pelo binario nativo do TKPROF (`Release 11.2.0.1.0 - Development on Sex Nov 20 21:54:30 2026`) gerou o perfil decomposto segundo o padrao de ordenacao da Lei de Amdahl (`sort=prsela,exeela,fchela`). 

A decomposicao confirma:
* **Matriz de Execucao Estável:** Parse (1 chamada), Execute (1 chamada), Fetch (2 chamadas para 4 linhas).
* **Consumo Focado em Memoria:** 9 buffers consistentes (`cr=9`) e zero leituras fisicas de disco (`disk=0`).
* **Temporizacao em Microssegundos:** Na secao de plano fisico (*Row Source Operation*), o kernel do Oracle 11gR2 registrou os tempos operacionais com resolucao de microssegundos (`time=0 us` e `time=3 us`), demonstrando que o subsistema de contadores de alta precisão do Windows 11 funcionou em harmonia com as chamadas de kernel da release de 2010.

---

## 6. Implicacoes Metodologicas para Pesquisa e Bancas Academicas

A articulacao entre o referencial teorico de Cary Millsap e a adocao estrita do Oracle Database 11.2.0.1.0 agrega valor cientifico ao projeto nos seguintes eixos:

1. **Rigor e Fidelidade Epistemologica:** Em pesquisas de ciencia da computacao, avaliar um modelo teorico sobre a mesma infraestrutura de software contemporanea a sua concepcao isola variaveis dependentes e confere autoridade metodologica a analise, impedindo que otimizacoes modernas introduzidas em releases posteriores mascarem os fenomenos fundamentais da computacao.
2. **Universalidade dos Axiomas de Desempenho:** Ao comprovar que a equacao fundamental $R = S + W$, a ordenacao decrescente do perfil tabular e a regra de reducao de leituras logicas funcionam com precisao matematica na versao de 2010 e se mantem consistentes na engenharia de sistemas contemporanea, o experimento atesta a perenidade dos principios de Millsap.
3. **Dominio do Ciclo de Vida da Infraestrutura:** A capacidade de diagnosticar problemas de kernel, instalar e operar versoes legadas sem suporte em sistemas operacionais modernos e depurar o comportamento interno do otimizador de consultas (como a diferenciacao entre chamadas recursivas e consultas de usuario) demonstra maturidade tecnica indispensavel para o nivel de pos-graduacao (*stricto sensu*).

---

## 7. Sintese Conclusiva

A utilizacao do **Oracle Database 11g Release 2 (11.2.0.1.0)** no presente laboratorio experimental nao se tratou de uma escolha aleatoria, mas de uma convergencia historica e tecnica com o marco de publicacao do artigo de Cary Millsap na *Communications of the ACM* em 2010. 

A execucao revelou a estabilidade dos mecanismos de rastreamento de baixo nivel do kernel (evento 10046 e TKPROF), a ausencia de complexidades distributivas de arquiteturas multitenant posteriores e a plena aplicabilidade das leis de desempenho em sistemas de software.
