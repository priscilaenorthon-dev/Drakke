# Prompt de desenvolvimento: clone do sistema Drake (Sapiensia) em Laravel/MySQL

**O Drake, da Sapiensia Tecnologia, é o sistema líder de mercado no Brasil para gestão de pessoal e operações offshore na indústria de petróleo e gás.** Com mais de 500 empresas, 10.000 usuários e 45.000 trabalhadores cadastrados controlando 600+ unidades operacionais, o Drake cobre todo o ciclo — do planejamento de escalas à execução logística, da conformidade documental ao processamento de folha. Este documento detalha cada módulo, fluxo de trabalho e regra de negócio necessários para construir um sistema equivalente usando Laravel (PHP) com MySQL no XAMPP, seguindo a metodologia obra/superpowers para desenvolvimento ágil com IA.

---

## Visão geral da arquitetura e escopo do sistema

O Drake opera como uma **aplicação web (SaaS multi-tenant)** acessível via navegador em `drake.bz`, com três camadas fundamentais que se repetem em cada módulo: **Cadastros** (alimentação de dados primários), **Tarefas** (comandos operacionais que processam dados) e **Consultas** (extração de relatórios e indicadores). A infraestrutura original utiliza Google Cloud Platform, AWS (Elasticsearch), SendGrid para e-mails, New Relic para monitoramento e Elastic Stack para busca e logs. O ecossistema inclui quatro produtos interconectados: o **Drake** (plataforma core para a empresa operadora), o **MyDrake** (app mobile/web para trabalhadores), o **iDrake Suppliers** (portal para fornecedores de serviços terceirizados) e o **iDrake Logistics** (portal para agências de viagem e transportadoras).

Para o clone em Laravel/MySQL no XAMPP, a arquitetura proposta deve ser **monolítica modular** com possibilidade futura de separação em microserviços. Cada módulo do Drake será um módulo Laravel com seus próprios Models, Controllers, Views, Services, Requests e Policies. O banco MySQL será organizado com prefixos de tabela por módulo. Autenticação via Laravel Breeze/Jetstream com **RBAC granular** (role-based access control) utilizando Spatie Laravel-Permission. O front-end pode usar Blade + Livewire para interatividade reativa, ou Inertia.js com Vue.js para uma experiência SPA. APIs RESTful para integração com os portais iDrake e o app MyDrake.

---

## Os 14 módulos principais e suas especificações detalhadas

O Drake possui **14 áreas funcionais** principais. Abaixo, cada módulo é descrito com suas entidades de banco de dados, telas, fluxos de trabalho e regras de negócio.

### Módulo 1 — Pessoas (cadastro central)

Este é o módulo fundacional que alimenta todos os demais. Contém dois cadastros essenciais:

**Entidade `trabalhadores` (workers)**: Nome completo, CPF, RG, data de nascimento, nacionalidade, sexo, estado civil, endereço completo, telefones, e-mail pessoal, e-mail corporativo, foto, dados bancários (banco, agência, conta, tipo), carteira de trabalho (CTPS número, série, UF, data emissão), PIS/PASEP, título de eleitor, certificado de reservista, CNH, passaporte (número, emissão, validade, país), vistos (tipo, protocolo, validade), RNE (para estrangeiros), situação (ativo/inativo/afastado/desligado), data de admissão, data de desligamento, categoria funcional, função/cargo, departamento, centro de custo, regime de trabalho, contrato vinculado, restrições de embarque (lista de restrições com motivo e validade), e campos customizáveis. A tela de cadastro do trabalhador funciona como um **dashboard completo do perfil** mostrando em abas: dados pessoais, documentos, qualificações/treinamentos, histórico de embarques, escala atual, compromissos futuros, situação de conformidade (compliant/não-compliant) e eventos.

**Entidade `entidades` (entities/companies)**: CNPJ, razão social, nome fantasia, inscrição estadual, endereço, contatos, tipo (operadora, fornecedora, agência de viagem, transportadora, centro de treinamento), status de credenciamento, documentos da empresa, contratos ativos e histórico. Cada entidade pode ter múltiplos **usuários** com diferentes perfis de acesso.

**Tabelas MySQL sugeridas**: `workers`, `worker_documents`, `worker_bank_accounts`, `worker_passports`, `worker_visas`, `worker_restrictions`, `entities`, `entity_contacts`, `entity_documents`, `entity_users`.

### Módulo 2 — RH (recursos humanos)

Gerencia categorias funcionais, funções, mapeamento de funções, nacionalidades, expatriados, carteiras de trabalho, restrições de embarque e toda a documentação do trabalhador.

**Cadastros**: Categorias (`categories` — CLT, PJ, estagiário, expatriado etc.), Funções (`job_functions` — cargo, CBO, descrição, requisitos mínimos), Mapeamento de funções (`function_mappings` — equivalência entre funções de diferentes contratos), Nacionalidades, Tipos de visto, Tipos de conta bancária, Tipos de protocolo de visto, Expatriados (mudanças pessoais, realocações internacionais).

**Tarefas de documentos**: O módulo RH é responsável pela **Gestão Eletrônica de Documentos (GED)**. Telas para: manter documentos (`worker_documents` — upload de arquivos PDF/imagem, tipo, validade, status de validação, quem validou, quando), manter tipos de documentos (`document_types` — nome, categoria, obrigatoriedade por função, prazo de validade padrão, alertas), manter pastas de documentos (`document_folders` — organização hierárquica). Cada documento tem estados: pendente, em análise, aprovado, recusado (com motivo), expirado. O sistema deve gerar **alertas automáticos 60 dias antes do vencimento** de qualquer documento obrigatório, e **bloquear embarque** de trabalhador com documentação vencida ou incompleta.

**Dashboard do trabalhador**: Tela consolidada mostrando foto, dados pessoais, situação contratual, conformidade documental (ícones verde/amarelo/vermelho), próximos vencimentos, histórico de embarques, escala vigente. Funciona como o "cartão de visita" do trabalhador no sistema.

### Módulo 3 — Conformidade e treinamento

O motor de conformidade é o coração regulatório do sistema. Impede que trabalhadores não qualificados embarquem, gerencia treinamentos e emite certificados.

**Matriz de qualificação** (`qualification_matrices`): Define quais qualificações são exigidas para cada função em cada unidade operacional/contrato. Estrutura: função × qualificação × obrigatoriedade (obrigatória/desejável) × validade. Qualificações incluem: CBSP (5 anos), HUET/T-HUET (5 anos), ASO (periódico), NR-37 (5 anos), NR-33, NR-35, NR-10, NR-13, BOSIET, cursos específicos por operadora. Cada qualificação (`qualifications`) tem: nome, código, validade padrão, tipo (segurança, técnica, médica, regulatória), requisitos para obtenção, requisitos para renovação.

**Necessidades de qualificação** (`qualification_needs`): Sistema calcula automaticamente, com base na matriz e nas qualificações atuais do trabalhador, quais treinamentos ele precisa realizar e quando. Gera lista priorizada por urgência (vencendo em 30/60/90 dias, já vencido, nunca obtido).

**Treinamentos presenciais** (`trainings`): Data, local, curso, instrutor, lista de treinandos, status (agendado, em andamento, concluído, cancelado), tipo de cancelamento, custo, fornecedor/centro de treinamento. Cada treinando tem status individual (inscrito, presente, ausente, aprovado, reprovado). Ao ser aprovado, sistema gera automaticamente a qualificação no perfil do trabalhador com data de validade.

**Treinamentos online** (`online_trainings`): Integração com plataforma de e-learning. Permite envio em lote. Trabalhador acessa via MyDrake, realiza módulos com quiz, sistema registra conclusão e emite certificado digital com assinatura digital (conformidade NR-1). Status: enviado, em andamento, concluído, reprovado.

**Consultas críticas**: Previsão de treinamentos (forecast — quantos trabalhadores precisarão de qual curso nos próximos X meses, permitindo planejamento de turmas e negociação com fornecedores), Conformidade do trabalhador (semáforo verde/amarelo/vermelho), Matriz de qualificação (view cruzada função × qualificação com indicadores).

**Validadores de treinamento**: Regras configuráveis que verificam consistência (ex: detectar evidências de qualificação obtidas em treinamentos presenciais realizados no mesmo período — possível fraude, trabalhador não poderia estar em dois treinamentos simultâneos). Parâmetros eSocial para reporte ao governo.

**Tabelas**: `qualifications`, `qualification_matrices`, `qualification_matrix_items`, `qualification_needs`, `courses`, `course_providers`, `training_locations`, `trainings`, `training_participants`, `online_trainings`, `online_training_attempts`, `certificates`, `training_validators`.

### Módulo 4 — Troca de turmas (crew rotation)

O módulo mais complexo e central do sistema. Gerencia todo o ciclo de escalas, desde o planejamento de vagas até a confirmação de embarque/desembarque.

**Cadastros fundamentais**:

- **Unidades Operacionais — UOP** (`operational_units`): Plataformas, FPSOs, navios, sondas. Campos: nome, código, tipo (plataforma fixa, semissubmersível, FPSO, navio-sonda, navio de apoio, flotel), operadora, localização (bacia, campo, coordenadas), capacidade total de POB, número de leitos, número de baleeiras com capacidade, helidecks, status (ativa, em manutenção, descomissionada). Cada UOP é um "mundo" isolado no sistema.

- **Contratos** (`contracts`): Vincula empresa fornecedora a operadora para determinada UOP. Campos: número do contrato, empresa contratante, empresa contratada, UOP, escopo de serviço, data início, data fim, valor, status. Cada contrato tem **projetos** (`contract_projects`) e **pessoas vinculadas** com funções específicas.

- **Plano de vagas** (`vacancy_plans`): Define o headcount necessário por função em cada UOP/contrato. Campos: UOP, contrato, função, quantidade de vagas por turma, regime de trabalho. É o "organograma operacional" da plataforma.

- **Regimes** (`work_regimes`): Define padrões de escala. Campos: nome (14×14, 14×21, 21×21, 28×28), dias embarcado, dias de folga, máximo de dias consecutivos (limite legal 15 dias pela Lei 5.811/72, exceção até 21 dias), tolerância para dobras. O regime mais comum é **14×14** (14 dias embarcado, 14 dias de folga).

- **Turmas** (`crews`): Grupos de trabalhadores que se revezam. Tipicamente Turma A e Turma B para cada vaga. Campos: nome/código, UOP, contrato, lista de membros, data de início da escala.

- **Tipos de ausência** (`absence_types`): Férias, licença médica, falta, treinamento, afastamento INSS, licença-paternidade/maternidade, motivos pessoais.

- **Tipos de dobra** (`extension_types`): Motivos para extensão além do período — falta de substituto, condições climáticas, emergência operacional, treinamento, férias do substituto.

- **SISPAT** (`sispat_situations`): Sistema de prevenção de acidentes de trabalho. Registra situações de segurança.

**A tela principal — Crew Rotation**: É um **calendário/timeline visual** (similar a um Gantt chart) que mostra por UOP e contrato: cada trabalhador com barras coloridas indicando períodos embarcado (azul), folga (verde), treinamento (amarelo), ausência (vermelho), dobra (laranja). Permite arrastar e soltar para realocar, clicar para editar eventos. Filtros por: UOP, contrato, turma, função, período, tipo de evento. Funcionalidades na tela:

- **Agendamento de embarque**: Selecionar trabalhador → definir data de embarque → sistema valida conformidade (documentos, treinamentos, ASO, restrições) → se conforme, gera necessidades logísticas automaticamente → se não conforme, bloqueia com indicador vermelho mostrando o que está faltando.
- **Agendamento de desembarque**: Selecionar trabalhador → definir data → gerar logística de retorno.
- **TBA (To Be Announced)**: Reservar vaga em voo/transporte para um trabalhador ainda não definido. Útil quando se sabe que haverá troca mas não se definiu quem substituirá. Suporta recorrência (TBA recorrente para trocas programadas).
- **Alteração de eventos em lote**: Mover múltiplos embarques/desembarques de uma vez (ex: quando um voo é cancelado e toda a turma precisa ser reagendada).
- **Indicadores RST**: Mostra status de validação de evidências de Requisição de Serviço Terceirizado.
- **Filtro rápido**: Busca por nome, CPF, matrícula, função.
- **Solicitação de embarque**: Cria pedido formal de embarque que passa por fluxo de aprovação.

**Gestão de voos** (`flights`): Cada troca de turma tipicamente envolve voos de helicóptero. O sistema permite: criar templates de voo (rotas recorrentes com origem, destino, horário, capacidade), gerenciar voos específicos (data, aeronave, piloto, passageiros, manifesto), vincular passageiros ao voo, gerar MTA (Manifesto de Transporte Aéreo — documento obrigatório com lista nominal de passageiros, empresa, função, destino).

**Gestão de ausências** (`absences`): Registra quando trabalhador não pode cumprir escala. Campos: trabalhador, tipo de ausência, data início, data fim, justificativa, documento comprobatório, aprovação do gestor. Ausência dispara automaticamente a **necessidade de cobertura**.

**Gestão de dobras** (`extensions`): Quando trabalhador precisa ficar além do período. Campos: trabalhador, data prevista de desembarque original, nova data, motivo, justificativa (obrigatória), aprovação. A lei limita a 21 dias no máximo.

**Gestão de adiamentos** (`deferrals`): Embarques/desembarques adiados por clima, logística, etc. Campos: evento original, nova data, motivo, status (pendente, confirmado, cancelado).

**Programações** (`schedules`): Sub-módulo que permite criar programações logísticas vinculadas às trocas de turma. Workflow: criar programação → editar participantes → incluir requisições de voo → finalizar programação logística → filtrar e pesquisar programações. Possui validadores de programação (regras que verificam consistência da programação).

**Consultas**: Agenda (calendário de todos os eventos), emitir notificação de embarque (documento formal enviado ao trabalhador informando data, hora, local de apresentação, itinerário completo), funções do contrato (vagas × preenchimento), participantes de programação.

**Tabelas**: `operational_units`, `contracts`, `contract_projects`, `contract_people`, `vacancy_plans`, `vacancy_plan_items`, `work_regimes`, `crews`, `crew_members`, `crew_rotation_events`, `flights`, `flight_templates`, `flight_passengers`, `mta_manifests`, `absences`, `absence_types`, `extensions`, `extension_types`, `deferrals`, `schedules`, `schedule_participants`, `sispat_situations`, `sispat_records`.

### Módulo 5 — A Bordo / POB (people on board)

Gerencia tudo que acontece **dentro da unidade operacional** após o embarque. É o módulo mais intuitivo — funciona de forma isolada por UOP, como se fosse um sistema próprio da plataforma. Suporta **modo offline** para operação quando a plataforma perde conectividade.

**Cadastros de infraestrutura**:
- **Baleeiras** (`lifeboats`): Cada baleeira tem código, localização na UOP, capacidade máxima de pessoas, tipo. O total de vagas em baleeiras define o POB máximo da UOP — **nunca pode haver mais pessoas a bordo do que vagas em baleeiras** (requisito de segurança obrigatório).
- **Camarotes** (`cabins`): Número/código, localização (deck, ala), quantidade de leitos, tipo (individual, duplo, triplo), amenidades, status (disponível, em manutenção).
- **Permissões de leito** (`berth_permissions`): Define quais funções/empresas podem usar quais leitos. Ex: camarotes do convés superior reservados para gestores, camarotes específicos para empresa X.
- **Grupos** (`pob_groups`): Agrupamento de pessoas a bordo (por empresa, por contrato, por disciplina).
- **Pessoas a bordo** (`pob_people`): Registro de cada pessoa embarcada com: trabalhador, data/hora embarque, camarote alocado, baleeira designada, turno (dia/noite), função a bordo, contrato, empresa.

**Tarefas operacionais a bordo**:
- **Definir horário de trabalho**: Atribuir turno (dia: 06h-18h ou noite: 18h-06h) a cada pessoa a bordo.
- **Definir local de operação**: Onde na plataforma o trabalhador está alocado.
- **Evento na unidade** (`unit_events`): Registrar ocorrências — acidentes, incidentes, simulados, inspeções, paradas, etc. Campos: tipo, data/hora, descrição, envolvidos, gravidade, ações tomadas.
- **Informar horas extras** (`overtime_records`): Registrar horas extras trabalhadas, com justificativa.
- **Realocar contrato a bordo**: Mover trabalhador de um contrato para outro sem desembarcar.
- **Realocar função a bordo**: Mudar a função do trabalhador temporariamente (ex: mecânico assumindo como eletricista).
- **Solicitação de diferença de função**: Quando trabalhador executa função diferente da contratual, gerando direito a adicional salarial.

**Operações de embarque/desembarque**: O fluxo tem duas etapas para cada: **Agendar** (planejar) e **Confirmar** (executar). Agendar embarque → pessoa aparece como "embarcando" → helicóptero chega → rádio operador confirma embarque → pessoa muda status para "a bordo" com data/hora exata. Mesmo para desembarque: agendar → confirmar quando realmente sai. Isso permite rastrear diferenças entre planejado e executado.

**MTA — Manifesto de Transporte Aéreo**: Tela que gera o documento oficial listando todos os passageiros de um voo de helicóptero. Inclui: nome, empresa, função, CPF, peso (para cálculo de carga), destino. É obrigatório por regulamentação.

**POB Planning**: Tela de planejamento visual mostrando ocupação da UOP ao longo do tempo — leitos ocupados vs disponíveis, vagas em baleeiras, projeção de overbooking. O sistema deve **alertar e bloquear** quando uma confirmação de embarque excederia a capacidade de baleeiras.

**Alterações a bordo**: Ajustes (correções de dados) e realocações (mudanças de camarote, baleeira, grupo).

**Relatórios**: Relatório de baleeiras (ocupação por baleeira), consultas customizadas, relatório de pessoas a bordo (POB list — lista nominal de todos a bordo com empresa, função, camarote, baleeira, data de embarque).

**Modo offline**: O módulo POB deve funcionar mesmo sem internet. Sincroniza quando conexão é restabelecida. Isso é crítico porque plataformas frequentemente perdem conectividade.

**Tabelas**: `lifeboats`, `cabins`, `berth_permissions`, `pob_groups`, `pob_records`, `pob_work_schedules`, `pob_locations`, `unit_events`, `unit_event_participants`, `overtime_records`, `contract_reallocations`, `function_reallocations`, `function_difference_requests`, `mta_manifests`, `mta_passengers`, `pob_adjustments`.

### Módulo 6 — Logística

Gerencia toda a cadeia logística de transporte de trabalhadores entre sua residência e a unidade operacional. Uma troca de turma típica envolve múltiplos trechos: ônibus da cidade de origem → aeroporto → voo comercial → hotel (pernoite) → transfer → heliponto → helicóptero → plataforma.

**Cadastros logísticos**:
- **Endereços** (`addresses`): Cidades, aeroportos, helipontos, hotéis, rodoviárias, bases de apoio.
- **Trajetos** (`routes`): Rotas completas de A a B, compostas por múltiplos trechos.
- **Trechos** (`route_segments`): Cada perna da viagem. Campos: origem, destino, tipo de transporte (aéreo comercial, aéreo executivo, helicóptero, rodoviário executivo, rodoviário regular, marítimo, transfer/van), fornecedor, custo estimado, duração estimada.
- **Trechos previstos** (`planned_segments`): Templates de trechos associados a UOPs específicas.
- **Custos dos trechos** (`segment_costs`): Tabela de preços por fornecedor, trecho e período.
- **Direitos logísticos** (`logistic_rights`): Define a quais tipos de transporte cada trabalhador/função tem direito. Ex: gerente tem direito a voo comercial executivo; técnico tem direito a econômico.
- **Limite mensal fornecedor** (`supplier_monthly_limits`): Teto de gasto por fornecedor logístico por mês.
- **Tabela de reembolso por distância** (`distance_reimbursement`): Para trabalhadores que usam veículo próprio.

**Tipos cadastrados**: Tipos de transporte aéreo (comercial, executivo, charter, helicóptero), tipos de transporte rodoviário (ônibus executivo, van, táxi, carro próprio), tipos de transporte marítimo (supply boat, lancha, balsa), tipos de hospedagem (hotel 3★, 4★, 5★, pousada), tipos de refeição em aeronave, posições de assento, localizações de assento, preferências de hotel (fumante/não-fumante), programas de milhagem, tipos de aluguel de carro (categorias e câmbio), categorias de cobrança.

**Fluxo de trabalho logístico**: Quando um embarque é agendado no módulo Troca de Turmas, o sistema **automaticamente gera necessidades logísticas** baseadas no trajeto padrão do trabalhador (residência → UOP). Cada necessidade logística tem: trabalhador, tipo de transporte, origem, destino, data/hora, status. O fluxo é:

1. **Geração automática**: Sistema cria necessidades baseado na escala e nos trechos previstos.
2. **Controle de necessidades**: Tela central onde o logístico visualiza todas as necessidades pendentes. Pode: ajustar, clonar, associar/dissociar necessidades, informar não-atendimento.
3. **Otimização**: Algoritmo que agrupa trabalhadores em mesmo voo/ônibus/hotel para reduzir custos.
4. **Requisição logística**: Formalização do pedido de compra de serviço.
5. **Ordem logística**: Envio ao fornecedor. Sistema calcula custo com base na tabela de preços. Se dentro do limite estimado, **aprovação automática**; se acima, exige aprovação gerencial (fluxo de aprovação com impressão e assinatura).
6. **Atendimento**: Fornecedor confirma (via iDrake Logistics ou e-mail) que o serviço será prestado.
7. **Reuso**: Atendimentos logísticos podem ser reutilizados (templates para trocas recorrentes).

**Compromissos** (`commitments`): Eventos logísticos genéricos vinculados ao trabalhador (reunião, exame médico, treinamento presencial, comparecimento em escritório). Cada compromisso pode gerar necessidades logísticas próprias.

**Gestão de conflitos**: Sistema detecta automaticamente conflitos logísticos — ex: trabalhador com embarque agendado mas sem passagem comprada, hotel reservado mas voo cancelado, dois embarques no mesmo dia, treinamento agendado durante período de embarque.

**Exportação de adiantamentos**: Gera lotes de adiantamento de viagem para trabalhadores que receberão diárias/valores antecipados.

**Gestão de cobranças RT**: Controla cobranças de serviços logísticos já prestados.

**Tabelas**: `addresses`, `routes`, `route_segments`, `planned_segments`, `segment_costs`, `logistic_rights`, `supplier_limits`, `distance_reimbursements`, `transport_types`, `accommodation_types`, `logistic_needs`, `logistic_requisitions`, `logistic_orders`, `logistic_order_approvals`, `logistic_attendances`, `commitments`, `commitment_logistics`, `logistic_conflicts`, `advance_batches`.

### Módulo 7 — Operações e serviços terceirizados

Gerencia operações diárias, equipes de trabalho, planejamento de tarefas e todo o fluxo de contratação e gestão de mão de obra terceirizada.

**Gestão de times** (`teams`): Criação e manutenção de equipes operacionais. Cada time tem: nome, UOP, contrato, líder, lista de participantes (com função de cada um), status. Permite ajustar participantes — adicionar, remover, trocar função de membros do time. Times podem ser montados por função ou por trabalhadores do "Talent Pool" (banco de talentos).

**Planejamento de tarefas**: Sub-módulo para gestão de atividades operacionais na plataforma.
- **Plano de tarefa** (`task_plans`): Plano macro com conjunto de tarefas. Campos: nome, UOP, contrato, período, status.
- **Tarefas** (`tasks`): Atividades individuais. Campos: classificação (tipo, categoria, característica), TAG (identificador do equipamento/local), criticidade da TAG, disciplina (mecânica, elétrica, instrumentação, civil, etc.), critérios de atividade, equipe necessária, duração estimada, status.
- **Planejador de tarefas** (`task_planner`): Tela visual (tipo Kanban ou Gantt) para programar e acompanhar tarefas, com indicadores de compromissos criados.

**Serviços terceirizados — RST (Requisição de Serviço Terceirizado)**: Este é o fluxo principal de interação com fornecedores via iDrake Suppliers.

**Fluxo completo RST**:
1. **Configuração prévia**: Empresa operadora configura no Drake: áreas requisitantes, grupos de validação, motivos de recusa, prioridades, tipos de serviço, tipos de autorização de terceiros.
2. **Criação da solicitação**: Área requisitante cria RST. Campos: descrição do serviço, motivo, local de execução, escopo de trabalho (atividades, número de trabalhadores necessários, funções, período), prioridade, documentos/qualificações exigidos (carregados da matriz de qualificação).
3. **Publicação para fornecedores**: RST é enviada automaticamente aos fornecedores credenciados via iDrake Suppliers.
4. **Atendimento do fornecedor**: Fornecedor recebe a solicitação, seleciona trabalhadores qualificados de seu banco de dados, e oferece como "membros" para atender a RST.
5. **Validação automática**: Sistema verifica automaticamente se os trabalhadores oferecidos atendem todos os requisitos da matriz de qualificação (treinamentos, ASO, certificados, documentos). Indicadores visuais mostram conformidade.
6. **Validação manual**: Empresa operadora pode validar/recusar membros, evidências de qualificação, documentos.
7. **Aprovação**: Quando todos os requisitos estão atendidos, o atendimento é aprovado.
8. **Autorização de trabalho**: Emissão de PT (Permissão de Trabalho) vinculada ao RST.
9. **Vinculação com embarque**: Trabalhadores aprovados são vinculados à escala de embarque.
10. **Atendimento incremental**: Possibilidade de atender parcialmente (ex: solicitar 5 eletricistas e receber 3 agora e 2 depois).

**Solicitação de cobertura**: Quando um trabalhador do quadro próprio fica ausente, sistema permite criar solicitação de cobertura com motivo, período, função necessária. Dispara busca de substituto no quadro próprio ou aciona RST para terceiro.

**Tela de inconsistências**: Identifica discrepâncias entre embarques/desembarques registrados no Drake e os reportados pelo fornecedor, permitindo conciliação.

**Tabelas**: `teams`, `team_members`, `task_plans`, `tasks`, `task_classifications`, `task_disciplines`, `task_criteria`, `tag_criticalities`, `service_requests`, `service_request_items`, `service_request_members`, `service_request_validations`, `service_request_attendances`, `work_authorizations`, `coverage_requests`, `coverage_request_reasons`, `requesting_areas`, `validation_groups`, `refusal_reasons`, `service_priorities`, `service_types`, `third_party_authorization_types`.

### Módulo 8 — SMS (sistema de gestão de segurança)

Focado em saúde ocupacional e segurança conforme NR-37 e SGSO (ANP).

**ASO — Atestado de Saúde Ocupacional**: Controle rigoroso de validade. Campos: trabalhador, data de emissão, data de validade, tipo de exame (admissional, periódico, retorno ao trabalho, mudança de função, demissional), médico responsável, CRM, resultado (apto, inapto, apto com restrições), restrições. O sistema deve **bloquear embarque se o ASO estiver vencido ou for vencer durante o período de embarque**. Configuração de parâmetros de validade de ASO por tipo de exame e função.

**Licenças médicas** (`medical_licenses`): CID, data início, data previsão fim, data efetiva fim, médico, tipo (INSS, empresa, particular), status (ativa, encerrada, prorrogada). Trabalhador com licença médica ativa é automaticamente impedido de embarcar.

**Afastamentos** (`leaves`): Qualquer tipo de afastamento — médico, disciplinar, pessoal, treinamento prolongado. Campos: tipo, motivo, data início, data fim, impacto na escala (substituto necessário?), documento comprobatório, aprovação.

**Matriz SMS**: Semelhante à matriz de qualificação, mas focada em requisitos de saúde — ASO, vacinação, exames complementares, atestados específicos. Configura quais documentos de saúde são obrigatórios por função/UOP.

**Tabelas**: `medical_certificates` (ASO), `aso_validity_rules`, `medical_licenses`, `leaves`, `leave_types`, `sms_matrix`, `sms_matrix_items`, `vaccination_records`.

### Módulo 9 — Folha de pagamento (payroll)

Processa folha de pagamento com regras específicas do regime offshore.

**Cadastros**: Classificação de eventos (tipos de verbas — salário base, hora extra, adicional noturno, adicional de periculosidade, adicional de confinamento, diárias, sobreaviso, dobra, diferença de função), Departamentos, Horários/Jornadas (definição de turnos com entrada, saída, intervalo), Ocorrências (tipos de eventos que geram impacto na folha — falta, atraso, hora extra, dobra, etc.).

**Fluxo de processamento**:
1. **Coleta de dados**: Sistema automaticamente coleta do módulo A Bordo: horas trabalhadas, horas extras, turnos, período embarcado, dobras, diferenças de função, eventos.
2. **Aprovação de eventos**: Workflow em que gestor/OIM aprova eventos que geram verbas (horas extras, dobras etc.). Tela de solicitação → tela de aprovação com filtros configuráveis.
3. **Cálculo de ocorrências**: Motor de regras configurável que transforma eventos em verbas. Estratégia de cálculo parametrizável por empresa/contrato.
4. **Cálculo de folha**: Processamento que aplica regras, gera verbas de provento e desconto, calcula INSS, IRRF, FGTS, adicional de periculosidade (30%), adicional noturno (20% mínimo), adicional de confinamento, sobreaviso.
5. **Emissão de TimeSheet**: Documento formal de registro de jornada, com definição de OIM (Offshore Installation Manager) responsável pela assinatura.
6. **Férias**: Cadastro de períodos aquisitivos, solicitação de férias (pode ser feita pelo trabalhador via MyDrake), aprovação, cálculo de férias com 1/3 constitucional.
7. **Eventos adicionais**: Gerenciamento de eventos fora do padrão — bônus, gratificações, descontos específicos.
8. **Lançamento manual de posição**: Para ajustes e correções.
9. **Ficha anual de posição**: Relatório anual consolidado por trabalhador com todas as verbas do período, para fins trabalhistas.

**Tabelas**: `payroll_events`, `event_classifications`, `departments`, `work_schedules`, `occurrence_types`, `occurrence_calculations`, `payroll_periods`, `payroll_records`, `payroll_items`, `timesheet_records`, `vacation_acquisitive_periods`, `vacation_requests`, `vacation_records`, `additional_events`, `manual_entries`.

### Módulo 10 — Financeiro

Centraliza controle financeiro de operações logísticas e serviços terceirizados.

**Cadastros**: Centros de custo (`cost_centers`), Contas contábeis (`accounting_accounts`), Moedas (`currencies`).

**Fluxo de pagamentos**: Fornecedores enviam faturas/cobranças pelo iDrake (Suppliers ou Logistics). Gestor financeiro acessa tela para **aprovar ou reprovar** cada pagamento. Permite importação em lote de planilhas de fatura (Excel/CSV → sistema). Controle de **reembolsos** para trabalhadores (despesas de viagem, alimentação, etc.).

**Tabelas**: `cost_centers`, `accounting_accounts`, `currencies`, `payment_requests`, `payment_approvals`, `invoice_imports`, `reimbursement_requests`, `reimbursement_items`.

### Módulo 11 — Notificações

Motor de notificações automatizadas configurável.

**Notificadores** (`notifiers`): Cada notificador é uma regra que dispara comunicações automáticas. Campos: nome, tipo (e-mail, push notification, SMS, in-app), trigger (evento que dispara — vencimento de documento em X dias, embarque agendado, RST criada, pagamento pendente, etc.), template de mensagem (com variáveis dinâmicas — nome do trabalhador, data, UOP etc.), destinatários (internos — usuários/grupos do sistema; externos — e-mails avulsos), filtros (condições adicionais para disparo), status (ativo/inativo), agendamento (imediato, diário, semanal). Permite clonar modelos existentes, personalizar templates, configurar permissões de quem pode criar/editar notificadores por grupo de usuários.

**Agendamento de consultas**: Permite agendar execução periódica de consultas customizadas e enviar resultados por e-mail automaticamente.

**Tabelas**: `notifiers`, `notifier_templates`, `notifier_recipients`, `notifier_filters`, `notifier_schedules`, `notification_logs`, `scheduled_queries`.

### Módulo 12 — TI / Administração

Configuração técnica, segurança e integrações.

**Gestão de acesso (RBAC)**: Sistema de permissões granular baseado em **papéis (roles)** e **recursos (resources)**. Cada recurso é uma funcionalidade do sistema (ex: "Manter Ordens Logísticas", "Emitir Relatório de Notificação de Embarque", "Aprovar Ordem Logística", "Alterar Ordem Logística Aprovada"). Recursos são agrupados por contexto (módulo). Tela de configuração permite: criar perfis, associar recursos a perfis, associar usuários a perfis, filtrar por papel para visualizar todos os recursos acessíveis. Controle de usuários ativos com desconexão automática de sessões inativas quando limite de licenças é atingido.

**Gestão de fluxos** (`workflows`): Configuração visual de fluxos de aprovação — quem aprova o quê, em que ordem, com que condições.

**Gestão de templates**: Templates de documentos, relatórios, notificações.

**Segurança**: Contas de usuário, log de auditoria completo (quem fez o quê, quando), procedimentos de autenticação, eliminação de dados (conformidade LGPD), monitoramento de usuários ativos.

**Integração — API de sincronização de dados**: Endpoints REST para sincronizar com sistemas externos (SAP, TOTVS, Oracle, sistemas legados). Entidades sincronizáveis: `SyncWorker`, `SyncJob`, `SyncCategory`, `SyncDepartment`, `SyncDocument`, `SyncEntity`, `SyncOperationalUnit`, `SyncNationality`, `SyncRegime`, `SyncCostCenter`, `SyncActivity`, `SyncAdditionalEvent`, `SyncMedicalLeave`, `SyncAbsence`, `SyncVacation`, `SyncVacationAcquisitivePeriod`, `SyncOccurrenceType`, `SyncContract`, `SyncCountry`, `SyncQualification`, `SyncTask`, `SyncTaskPlan`, `SyncTaskDiscipline`, `SyncTaskCriteria`, `SyncTaskTagCriticality`, `SyncVisaType`, `SaveSyncLog`. Cada entidade tem mapeamento de campos configurável via **Integration Maps** com campo de código e origem para mapeamento bidirecional com ERPs.

**WebHooks**: Configuração de webhooks que notificam sistemas externos quando eventos ocorrem no Drake (ex: embarque confirmado → webhook para SAP registrar entrada na UOP).

**Consultas customizadas (DQL)**: Linguagem própria de consulta que permite usuários avançados criarem relatórios customizados sem programação.

**Downloads e Inbox**: Central de downloads para relatórios gerados em background, Inbox para comunicação interna.

**Tabelas**: `roles`, `permissions`, `role_permissions`, `user_roles`, `users`, `user_sessions`, `audit_logs`, `workflow_definitions`, `workflow_steps`, `workflow_instances`, `document_templates`, `integrations`, `integration_maps`, `integration_map_fields`, `webhooks`, `webhook_logs`, `api_sync_logs`, `custom_queries`, `download_center`, `inbox_messages`.

### Módulo 13 — MyDrake (portal do trabalhador)

Aplicação mobile (iOS/Android) e web (`drake.bz/my`) para auto-atendimento do trabalhador.

**Funcionalidades**: Inbox (mensagens da empresa), central de solicitações (abrir chamados), compromissos (ver agenda — embarques, desembarques, treinamentos, exames), contatos corporativos, documentos (visualizar e fazer upload), férias (solicitar e acompanhar), necessidades de qualificação (ver o que precisa treinar), treinamento online (realizar cursos com quiz e download de certificado), consultar escala, ver logística planejada (hotel, voo, transfer), atualizar dados pessoais. **Registro via e-mail corporativo** previamente cadastrado no Drake — trabalhador recebe e-mail de ativação. Suporta login com conta Google.

**Benefícios para a empresa**: Reduz falhas de comunicação sobre escalas, cria transparência, controla visualização de mensagens, reduz absenteísmo (no-show), reduz custos de comunicação.

**Tabelas adicionais**: `mydrake_access`, `mydrake_messages`, `mydrake_requests`, `mydrake_training_sessions`, `mydrake_document_uploads`.

### Módulo 14 — iDrake Suppliers (portal de fornecedores)

Portal web separado (`suppliers.drake.bz`) para empresas fornecedoras de serviços terceirizados.

**Cadastros do lado fornecedor**: Clientes (lista de empresas que usam Drake onde o fornecedor pode se credenciar), Trabalhadores (cadastro completo com documentos e qualificações dos trabalhadores do fornecedor), Usuários (gestão de usuários do fornecedor com roles admin/operador), Atividades, Documentos, Tipos de documentos.

**Fluxo do fornecedor**:
1. Solicitar acesso ao Drake → receber credenciais.
2. Cadastrar dados da empresa, usuários e trabalhadores.
3. Solicitar credenciamento em cada empresa-cliente que deseja atender.
4. Receber solicitações de serviço terceirizado (RST) dos clientes.
5. Oferecer trabalhadores qualificados como membros para atender cada RST.
6. Sistema valida automaticamente qualificações dos trabalhadores oferecidos.
7. Se recusa de documento/trabalhador/serviço, tratar a recusa (corrigir ou substituir).
8. Solicitar embarque dos trabalhadores aprovados.
9. Enviar faturas/pagamentos via módulo financeiro integrado.
10. Solicitar isenção de documentos quando aplicável.

**Tabelas adicionais**: `supplier_credentials`, `supplier_workers`, `supplier_worker_documents`, `supplier_service_responses`, `supplier_member_offers`, `document_exemptions`.

---

## Fluxo de trabalho completo: do planejamento à execução

O ciclo operacional completo no Drake segue uma sequência lógica que integra todos os módulos:

**Fase 1 — Configuração inicial**: Cadastrar UOPs (plataformas/embarcações) com capacidades (leitos, baleeiras). Cadastrar contratos com escopo, período e funções. Criar plano de vagas (headcount por função por UOP). Definir regimes de trabalho (14×14, etc.). Criar turmas (A, B). Configurar matriz de qualificação (quais treinamentos/documentos são obrigatórios por função). Cadastrar trabalhadores com todos os dados, documentos e qualificações.

**Fase 2 — Planejamento de escalas**: No Crew Rotation, montar a escala de cada UOP. Vincular trabalhadores a vagas nas turmas. Sistema calcula automaticamente as datas de embarque/desembarque baseado no regime. Validar conformidade de cada trabalhador contra a matriz de qualificação — identificar pendências (treinamento vencendo, ASO a renovar). Agendar treinamentos necessários antes do próximo embarque. Reservar vagas TBA para posições ainda sem trabalhador definido. Gerar programação logística da troca de turma.

**Fase 3 — Preparação (7-30 dias antes)**: Emitir notificações de embarque para trabalhadores (via MyDrake). Gerar necessidades logísticas automaticamente. Logístico revisa, otimiza e emite requisições/ordens logísticas. Fornecedores logísticos confirmam atendimento (via iDrake Logistics). Verificação final de conformidade (documentos, ASO, treinamentos — se algum venceu neste intervalo, bloquear e providenciar solução).

**Fase 4 — Execução da troca de turma**: Trabalhador segue itinerário logístico (ônibus → avião → hotel → transfer → helicóptero). No heliponto, gerar MTA com lista de passageiros. Na plataforma, rádio operador confirma embarque no POB. Sistema atribui camarote e baleeira. Turma que sai tem desembarque confirmado. POB é atualizado em tempo real.

**Fase 5 — Operação a bordo (14-21 dias)**: Gestão de POB diária — turnos, horas extras, eventos, realocações. Planejamento e execução de tarefas operacionais. Gestão de times de trabalho. Se trabalhador adoece → registrar evento/licença médica → solicitar MEDEVAC se necessário → criar solicitação de cobertura → buscar substituto (quadro próprio ou RST).

**Fase 6 — Pós-operação**: Emissão de TimeSheets. Processamento de folha de pagamento. Aprovação de eventos (horas extras, dobras, diferenças de função). Conciliação financeira com fornecedores. Relatórios gerenciais.

---

## Estrutura técnica recomendada para o clone em Laravel

**Stack tecnológico**: Laravel 11+ (PHP 8.2+), MySQL 8.0+ (XAMPP), Blade + Livewire 3 (ou Inertia.js + Vue 3), Tailwind CSS, Spatie Laravel-Permission (RBAC), Laravel Sanctum (API auth), Laravel Queue (jobs assíncronos), Laravel Notifications (multi-canal), Laravel Scout + Meilisearch (busca), Mailtrap/Mailhog (e-mails em dev).

**Organização em módulos Laravel**: Cada módulo como um Service Provider registrado no `AppServiceProvider`. Estrutura de diretórios: `app/Modules/Pessoas/`, `app/Modules/RH/`, `app/Modules/Conformidade/`, `app/Modules/TrocaDeTurmas/`, `app/Modules/POB/`, `app/Modules/Logistica/`, `app/Modules/Operacoes/`, `app/Modules/SMS/`, `app/Modules/Folha/`, `app/Modules/Financeiro/`, `app/Modules/Notificacoes/`, `app/Modules/TI/`, `app/Modules/MyDrake/`, `app/Modules/IDrakeSuppliers/`.

**Estimativa de tabelas MySQL**: Aproximadamente **120-150 tabelas** para cobrir todas as entidades descritas acima.

**Prioridade de desenvolvimento (superpowers — desenvolvimento ágil com IA)**:

- **Sprint 1 — Fundação** (2-3 semanas): Auth, RBAC, cadastro de usuários, entidades, trabalhadores, UOPs. Dashboard básico.
- **Sprint 2 — Compliance Core** (2-3 semanas): Qualificações, matriz de qualificação, documentos do trabalhador, validação de conformidade, alertas de vencimento.
- **Sprint 3 — Crew Rotation** (3-4 semanas): Escalas, turmas, regimes, crew rotation visual (timeline/Gantt), agendamento de embarque/desembarque, gestão de ausências e dobras.
- **Sprint 4 — POB** (2-3 semanas): Baleeiras, camarotes, gestão de pessoas a bordo, confirmação de embarque/desembarque, MTA, relatórios POB. Modo offline via Service Workers.
- **Sprint 5 — Logística** (2-3 semanas): Rotas, trechos, direitos logísticos, geração automática de necessidades, requisições, ordens, fluxo de aprovação.
- **Sprint 6 — Operações e RST** (2-3 semanas): Times, tarefas, serviços terceirizados, fluxo completo RST.
- **Sprint 7 — Folha e Financeiro** (2-3 semanas): Folha de pagamento, TimeSheet, férias, controle financeiro.
- **Sprint 8 — SMS e Notificações** (1-2 semanas): ASO, licenças médicas, afastamentos, motor de notificações.
- **Sprint 9 — MyDrake e iDrake** (2-3 semanas): Portal do trabalhador (API + web responsivo), portal de fornecedores.
- **Sprint 10 — Integrações e Polish** (2 semanas): API de sincronização, webhooks, consultas customizadas, relatórios avançados, testes, otimização.

---

## Regulamentações brasileiras que o sistema deve contemplar

O sistema deve estar em conformidade com as seguintes regulamentações:

A **Lei 5.811/1972** rege o trabalho no setor petrolífero e define o limite máximo de **15 dias consecutivos** de confinamento (permitindo extensão até 21 dias em casos excepcionais). Os regimes de trabalho mais utilizados são o **14×14** (padrão Petrobras para empregados próprios), 14×21, e 21×21 (empresas terceirizadas com acordo coletivo).

A **NR-37** é a norma regulamentadora específica para plataformas de petróleo e estabelece requisitos de PGR (Programa de Gerenciamento de Riscos), PCMSO (Programa de Controle Médico), PRE (Plano de Resposta a Emergências), CIPLAT (Comissão Interna de Prevenção de Acidentes em Plataformas) e SESMT. Todo trabalhador precisa de ASO válido, e nenhum trabalhador pode acessar a plataforma sem ASO em dia. Registros de acesso, permanência e saída devem ser mantidos por no mínimo **12 meses**. Trabalhadores ausentes por mais de 180 dias precisam de reciclagem de treinamento.

O **SGSO da ANP** (Resolução 43/2007) define 17 práticas de gerenciamento de segurança operacional que o sistema deve suportar documentalmente, especialmente: qualificação e treinamento de pessoal (Prática 3), gestão de contratados (Prática 6), documentação e registros (Prática 5), e procedimentos de trabalho seguro (Prática 17).

Certificações obrigatórias incluem **CBSP** (5 anos de validade), **HUET/T-HUET** (5 anos), **ASO** (periódico), treinamento básico NR-37 (5 anos, reciclagem com 4h). Certificações adicionais por função: NR-33 (espaço confinado), NR-35 (altura), NR-10 (eletricidade), NR-13 (caldeiras e vasos de pressão), BOSIET (para operadoras internacionais). O sistema deve rastrear todas essas validades e bloquear automaticamente embarques de trabalhadores não conformes.

---

## Glossário de termos offshore para modelagem de dados

| Termo | Significado | Entidade no sistema |
|-------|-------------|-------------------|
| UOP | Unidade Operacional (plataforma, FPSO, navio) | `operational_units` |
| POB | People on Board / Pessoal a bordo | `pob_records` |
| MTA | Manifesto de Transporte Aéreo | `mta_manifests` |
| RST | Requisição de Serviço Terceirizado | `service_requests` |
| ASO | Atestado de Saúde Ocupacional | `medical_certificates` |
| CBSP | Curso Básico de Segurança de Plataforma | `qualifications` |
| HUET | Helicopter Underwater Escape Training | `qualifications` |
| Baleeira | Bote salva-vidas da plataforma | `lifeboats` |
| Camarote | Cabine/dormitório na plataforma | `cabins` |
| Leito | Cama individual no camarote | Campo em `cabins` |
| Dobra | Extensão além do período de embarque | `extensions` |
| Troca de turma | Crew change / rotação de equipe | `crew_rotation_events` |
| Embarque | Chegada à plataforma | Evento em `crew_rotation_events` |
| Desembarque | Saída da plataforma | Evento em `crew_rotation_events` |
| TBA | To Be Announced (vaga sem trabalhador definido) | Flag em `crew_rotation_events` |
| Regime | Padrão de escala (14×14 etc.) | `work_regimes` |
| Turma | Equipe que se reveza (A/B) | `crews` |
| OIM/GEPLAT | Gerente da plataforma | Role no sistema |
| SISPAT | Semana interna de prevenção de acidentes | `sispat_records` |
| Sobreaviso | Regime de prontidão | Campo em `work_regimes` |
| CIPLAT | Comissão de prevenção de acidentes na plataforma | Registro em SMS |

---

## Conclusão: um sistema de alta complexidade regulatória com núcleo em crew rotation

O Drake não é apenas um sistema de RH offshore — é uma **plataforma de orquestração operacional** que conecta cinco domínios críticos: gestão de pessoas (trabalhadores, qualificações, documentos), planejamento operacional (escalas, trocas de turma, vagas), logística multimodal (helicóptero, avião, hotel, ônibus, van), cadeia de suprimento de mão de obra (serviços terceirizados, credenciamento de fornecedores) e conformidade regulatória (NR-37, SGSO, ASO, treinamentos obrigatórios). O módulo de **Crew Rotation** é o coração do sistema — tudo gira em torno do ciclo de embarque/desembarque. A conformidade é o "guardião" que bloqueia operações quando requisitos regulatórios não são atendidos. A logística é o "executor" que materializa o planejamento em passagens, hotéis e voos. E o POB é o "espelho da realidade" que registra o que efetivamente acontece a bordo. Construir este clone exigirá atenção especial à modelagem de dados (entidades altamente relacionadas), regras de negócio complexas (validações cruzadas de conformidade), interfaces visuais ricas (timelines de escala, dashboards de POB) e robustez (modo offline, auditoria completa, segurança de dados). A metodologia obra/superpowers é ideal para atacar cada módulo como uma "obra" independente com entregas incrementais, usando IA para acelerar a geração de Models, Migrations, Controllers e telas CRUD que compõem a maior parte do sistema.