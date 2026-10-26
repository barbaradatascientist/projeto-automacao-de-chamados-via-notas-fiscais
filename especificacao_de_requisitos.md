# ESPECIFICAÇÃO DE REQUISITOS DE SOFTWARE (SRS)

**Projeto:** Automação de Chamados de Regularização via Ingestão de Notas Fiscais  
**Versão:** 1.1  

---

## 1. INTRODUÇÃO

### 1.1 Objetivo do Documento
Este documento especifica os requisitos funcionais, requisitos não funcionais, integrações e regras de negócio para o desenvolvimento do serviço de automação de ingestão, validação contratual de faturas e abertura automática de chamados de regularização de medição na plataforma Jira.

### 1.2 Escopo do Produto

#### O que o sistema faz:
- Monitora continuamente caixa postal corporativa dedicada via Microsoft Graph API / Exchange.
- Filtra e identifica mensagens contendo documentos fiscais (XML NF-e e DANFE em PDF).
- Extrai metadados dos arquivos XML (com fallback via OCR para PDF legível).
- Cruza os dados da NF com os contratos e Ordens de Compra (OC) ativos na base SQL Server (sistema Minerion).
- Executa a abertura parametrizada de chamados no Jira Service Management anexando os arquivos originais.
- Notifica fornecedores e moderadores em caso de divergências cadastrais.
- Classifica mensagens de cobrança de títulos vencidos e roteia notificações automáticas para Financeiro, Fiscal ou Fornecedor conforme o estágio do documento no ERP.

#### O que o sistema NÃO faz:
- Não efetua liquidação financeira, agendamento bancário ou pagamento de títulos.
- Não efetua validação fiscal e tributária.
- Não realiza a conferência física e recebimento de materiais em almoxarifado.
- Não processa requisições de compra avulsas sem contrato prévio vinculado (nesta fase do projeto).
- Não processa Conhecimentos de Transporte Eletrônico (CT-e) nem responde a e-mails promocionais/comunicações não financeiras.

### 1.3 Glossário e Abreviações
- **DANFE:** Documento Auxiliar da Nota Fiscal Eletrônica.
- **NF-e:** Nota Fiscal Eletrônica (formato XML padrão SEFAZ).
- **OC:** Ordem de Compra.
- **Jira:** Sistema de gerenciamento de chamados e fluxo de trabalho.
- **Minerion:** Sistema ERP/gestão de suprimentos e contratos.
- **SLA:** *Service Level Agreement* (Acordo de Nível de Serviço).

---

## 2. DESCRIÇÃO GERAL

### 2.1 Perspectiva do Produto
O sistema opera como um serviço de integração desacoplado (*worker* / daemon assíncrono), interligando:
1. **Canal de Entrada:** Servidor de E-mail (Microsoft Exchange / Microsoft Graph API).
2. **Motor de Extração:** Parser estruturado de XML (DOM/SAX) e OCR/Regex para documentos PDF.
3. **Base Corporativa:** Banco de dados SQL Server do ERP Minerion (validação de contratos e medições).
4. **Canal de Saída:** API REST do Jira para criação e encadeamento de chamados.

### 2.2 Perfis de Usuários (Atores)

| Perfil | Descrição | Nível de Acesso |
| :--- | :--- | :--- |
| **Robô / Serviço (Worker)** | Executa o pipeline de ingestão, validações e abertura de chamados. | Leitura/Escrita em e-mails; Leitura em SQL; Escrita no Jira. |
| **Analista de Suprimentos** | Dá andamento aos chamados gerados com sucesso e analisa divergências. | Leitura/Escrita no Jira. |
| **Moderador** | Atua sobre e-mails classificados em quarentena ou exceções de leitura. | Leitura na caixa de e-mail; Leitura/Escrita no Jira. |
| **Administrador do Sistema** | Configura credenciais, parâmetros de tolerância, conexões e monitora logs. | Acesso administrativo completo a configs e observabilidade. |

### 2.3 Premissas e Dependências
- A caixa postal dedicada deve dispor de credenciais com escopo de permissão adequado (`Mail.ReadWrite`).
- O fornecedor deve disponibilizar documento fiscal em formato XML válido (schema SEFAZ) ou PDF com camada de texto legível.
- A base SQL Server do Minerion deve estar operacional e acessível via rede com tempo de resposta compatível com os SLAs estabelecidos.
- A API REST do Jira deve possuir autenticação ativa via API Token ou OAuth 2.0 com permissões de criação de issues e anexação de arquivos.

---

## 3. REQUISITOS DE INTEGRAÇÃO E INTERFACE EXTERNA

### 3.1 Interface de Entrada (E-mail)
- Conexão segura obrigatória via OAuth 2.0 (Microsoft Graph API) ou TLS na porta 993 (IMAP seguro).
- Leitura de metadados: remetente, assunto, data/hora, corpo da mensagem e lista de anexos.

### 3.2 Interface de Saída (Jira Service Management)
- Envio de requisições RESTful (`POST /rest/api/3/issue`) em formato JSON.
- Mapeamento de campos obrigatórios: Projeto, Tipo de Item (Issue Type), Fornecedor (CNPJ/Razão Social), Número da NF, Valor Total, Número do Contrato/OC, Centro de Custo e Descrição da Medição.
- Upload multipart dos arquivos de entrada (XML e PDF) diretamente no ticket criado (`POST /rest/api/3/issue/{issueIdOrKey}/attachments`).

---

## 4. REQUISITOS FUNCIONAIS (RF)

| ID | Nome do Requisito | Descrição | Prioridade |
| :--- | :--- | :--- | :--- |
| **RF01** | Monitoramento de Mensagens | O sistema deve monitorar a caixa de entrada dedicada e identificar novas mensagens elegíveis a partir de palavras-chave no assunto ou corpo (NF, Nota Fiscal, Fatura, Cobrança, Boleto). | Essencial |
| **RF02** | Filtragem e Triagem Inicial | O sistema deve verificar a presença de anexos `.xml` ou `.pdf`. Se a mensagem contiver as palavras-chave mas não possuir anexos válidos, deve ser movida para a pasta `Pendencias`. Mensagens sem palavras-chave devem ser marcadas como lidas e mantidas fora do fluxo de erro. | Essencial |
| **RF03** | Extração de Metadados de XML | O sistema deve efetuar o parsing do XML da NF-e, extraindo no mínimo: chave de acesso (`<chNFe>`), número da nota (`<nNF>`), dados do emitente (`<emit>`), destinatário (`<dest>`), valor total (`<vNF>`), impostos e número do pedido/OC (`<xPed>`). | Essencial |
| **RF04** | Extração de Dados de PDF (Fallback) | Na ausência de XML, o sistema deve executar OCR/expressões regulares no PDF para capturar: CNPJ Emitente, CNPJ Destinatário, Número da NF, Valor Total e Referência da OC. | Importante |
| **RF05** | Validação de Contrato Ativo (Minerion) | O sistema deve consultar a base SQL Server do sistema Minerion para confirmar se o CNPJ emitente possui contrato ativo e saldo disponível associado à OC indicada. Caso não possua, o e-mail deve ser direcionado para a pasta `Pendencias` com registro de log. | Essencial |
| **RF06** | Abertura de Chamado no Jira | Cumpridas todas as validações, o sistema deve registrar a requisição no Jira com título padronizado: `[NF {nNF}] - {Razao_Social_Emitente} - OC: {Numero_OC}`. | Essencial |
| **RF07** | Vinculação de Anexos | O sistema deve realizar o upload do arquivo XML e do arquivo PDF original no chamado criado no Jira antes da conclusão do atendimento da fila. | Essencial |
| **RF08** | Gestão de Pastas e Estados do E-mail | Após a criação do chamado com sucesso, o sistema deve marcar a mensagem como lida e movê-la para a pasta `Processados`. Em caso de erro irrecuperável ou inconsistência de negócio, deve movê-la para `Pendencias` ou `Falha_Parse`. | Essencial |
| **RF09** | Notificação Automática ao Fornecedor | Em caso de divergência de dados passível de retificação (CNPJ incorreto do destinatário, valor do boleto maior que o valor total da nota), o sistema deve disparar e-mail automático ao remetente com os apontamentos técnicos solicitando a reemissão da nota. | Desejável |
| **RF10** | Notificação de Exceção para Moderador | Quando uma falha de sistema ocorrer ou a inconsistência não puder ser resolvida por regra automática, o sistema deve notificar o Moderador com os detalhes da falha. | Essencial |
| **RF11** | Classificação e Descarte de Ruído | O sistema deve classificar o e-mail recebido descartando comunicações gerais e propagandas (sem anexos fiscais e sem termos de cobrança), mantendo-as na caixa ou movendo para `/Descartados`. | Essencial |
| **RF12** | Filtro Exclusivo de NF-e (Ignorar CT-e) | O sistema deve validar a tipologia do documento; caso o arquivo XML seja identificado como CT-e (modelo 57) ou documento de frete, o sistema deve ignorar o processamento fiscal e mover para a pasta `/CTe_Nao_Processados`. | Essencial |
| **RF13** | Identificação de Cobrança de Título Atrasado | O sistema deve identificar e-mails com teor de cobrança de títulos/boletos pendentes, extrair o número da NF/boleto referenciado e consultar o pipeline financeiro/fiscal no Minerion. | Essencial |
| **RF14** | Roteamento de Cobrança por Causa-Raiz | Conforme a consulta da cobrança, o sistema deve agir autonomamente:<br>1. **Nota Inexistente:** Disparar e-mail ao fornecedor solicitando a NF.<br>2. **Nota não escriturada:** Disparar e-mail ao setor Fiscal alertando sobre o vencimento sem registro.<br>3. **Nota registrada pendente de liquidação:** Disparar e-mail ao setor Financeiro com o aviso de cobrança. | Essencial |

---

## 5. REGRAS DE NEGÓCIO (RN)

- **RN01 - Prevenção de Duplicidade (Unicidade da NF):**  
  O sistema não deve abrir chamados para notas fiscais já processadas. A unicidade deve ser validada por: Chave NF-e ou (`CNPJ Emitente` + `Número da NF` + `Série`). Identificada duplicidade, o e-mail deve ser movido para a subpasta `Duplicadas` e auditado no log, abortando o processamento.

- **RN02 - Validação de Destinatário Autorizado:**  
  O processamento somente é autorizado se o CNPJ destinatário (`<dest><CNPJ>`) pertencer à lista de filiais ativas do grupo econômico da contratante.

- **RN03 - Prioridade da Fonte de Dados:**  
  Havendo simultaneamente XML e PDF no mesmo e-mail, os dados devem ser extraídos obrigatoriamente do arquivo XML. O PDF será utilizado exclusivamente para visualização humana e anexo no ticket.

- **RN04 - Tolerância de Variação de Valor Contratual e Teto Mensal:**  
  O valor da nota fiscal não pode exceder o saldo remanescente da Ordem de Compra/medição no Minerion para o mês de competência. Caso a nota apresente valor divergente além da margem de tolerância configurada ou faturamento atípico não previsto em contrato, o lançamento deve ser suspenso e encaminhado para validação humana (`Pendencias`).

- **RN05 - Conformidade de Dados Cadastrais:**  
  A nota deve ser rejeitada para abertura automática caso o CNPJ destinatário divirja do estabelecimento indicado na Ordem de Compra vinculada.

- **RN06 - Segregação de Tipos de Documento Fiscal:**  
  Apenas documentos com schema de NF-e (modelo 55) ou NFS-e são elegíveis para abertura de chamados de regularização. Quaisquer documentos do tipo CT-e (modelo 57) devem ser desconsiderados pela esteira de ingestão.

- **RN07 - Diagnóstico de Cobrança Atrasada:**  
  Ao receber uma notificação de atraso/cobrança de fornecedor, a prioridade de resolução deve seguir a hierarquia:
  1. *Nota não encontrada no sistema:* Notificar Fornecedor.
  2. *Nota encontrada, status = "Aguardando Escrituração/Fiscal":* Alertar Fiscal.
  3. *Nota encontrada, status = "Escriturada / Aguardando Pagamento":* Alertar Financeiro.

---

## 6. REQUISITOS NÃO FUNCIONAIS (RNF)

| ID | Categoria | Descrição | Métrica / Critério de Aceite |
| :--- | :--- | :--- | :--- |
| **RNF01** | Desempenho | Tempo de processamento ponta a ponta (leitura, validações e criação do chamado) para notas com XML. | $\le 180$ segundos por documento. |
| **RNF02** | Segurança | Nenhuma credencial ou chave de API pode constar em código-fonte (*hardcoded*). Todas devem ser gerenciadas via cofre de segredos / variáveis de ambiente. | Auditoria de conformidade (100% *compliant*). |
| **RNF03** | Auditoria e Rastreabilidade | Todas as ações do ciclo de vida da mensagem devem produzir logs em formato estruturado (JSON), registrando timestamp, `message_id`, CNPJ, número da NF, status e stack trace em falhas. | Retenção mínima de 90 dias. |
| **RNF04** | Disponibilidade e Resiliência | O serviço deve implementar política de retentativas com recuo exponencial (*exponential backoff*) em caso de falhas transitórias em APIs externas. | Disponibilidade $\ge 99{,}5\%$ em dias úteis. |
| **RNF05** | Manutenibilidade | Arquitetura modular desacoplada (Camada de Ingestão, Parsers, Validadores de Negócio e Gateways de API). | Cobertura de testes unitários $\ge 80\%$ nos parsers e regras de validação. |

---

## 7. MATRIZ DE TRATAMENTO DE EXCEÇÕES

| Cenário de Exceção | Causa Raiz | Ação do Sistema | Destino do E-mail |
| :--- | :--- | :--- | :--- |
| E-mail com palavra-chave sem anexo | Falta de anexo ou formato não suportado | Registra log de aviso e notifica moderador | `Pendencias` |
| Arquivo XML corrompido ou fora do schema | Falha de integridade do arquivo | Registra erro no log e dispara notificação técnica | `Falha_Parse` |
| Fornecedor sem contrato ativo ou OC inexistente | Divergência cadastral no Minerion | Suspende abertura automática e gera log | `Pendencias` |
| NF já cadastrada anteriormente | Tentativa de reprocessamento ou duplicidade | Aborta processo sem abrir novo chamado | `Duplicadas` |
| Indisponibilidade da API do Jira | Falha de rede ou timeout de serviço externo | Executa 3 tentativas com intervalo exponencial; se persistir, mantém na caixa de entrada para novo ciclo | Mantém na Caixa de Entrada |
| Mensagem de propaganda / newsletter | Ausência de anexos fiscais e ausência de cobrança | Marca como lido e arquiva | `/Descartados` ou mantém lido |
| Anexo identificado como CT-e | Modelo 57 / Conhecimento de Transporte | Não processa; registra log informativo | `CTe_Ignorados` / `CTe_Nao_Processados` |
| Cobrança: NF não localizada | Falta de envio do documento pelo parceiro | Responde ao remetente solicitando o envio da NF | `/Cobrancas/Aguardando_Fornecedor` |
| Cobrança: NF parada no Fiscal | Atraso no registro/validação fiscal interna | Dispara e-mail de alerta à equipe Fiscal | `/Cobrancas/Encaminhado_Fiscal` |
| Cobrança: NF parada no Financeiro | Atraso no pagamento de NF já registrada | Dispara e-mail de cobrança à equipe Financeira | `/Cobrancas/Encaminhado_Financeiro` |