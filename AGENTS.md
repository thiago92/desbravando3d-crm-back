# CRM Back — contexto e instruções

## Estado e escopo

- Este diretório contém apenas o scaffold ASP.NET Core em `Desbravando3dCRM.Api/` (`net10.0`, controller e WeatherForecast de exemplo). Ainda não há domínio, persistência, autenticação ou contrato de API implementados.
- Nesta etapa, analisar e documentar. Não criar projetos, entidades, endpoints, scripts SQL, migrations nem alterar o scaffold até nova solicitação do usuário.
- O foco é a API do CRM. O e-commerce e sua segunda API são futuros; os produtos deverão poder ser compartilhados por ambos através do banco comum, com contratos e responsabilidades ainda a definir.

## Fontes de verdade

- Regras e fluxos de negócio: `../CRM.Front/desbravando3d-crm-front/AGENTS.md` e, sobretudo para a passagem ao backend, `../CRM.Front/desbravando3d-crm-front/docs/backend-handoff.md`. Consultar também as implementações e testes de cada feature antes de fechar uma regra. Os mocks e IDs do front são prévia, não contrato definitivo de API ou esquema do banco.
- Referência de organização: `../../Portfolio/Portfolio.Back.Api/portfolio-back/` (solução com `PortfolioApi`, `Portfolio.Application`, `Portfolio.Domain`, `Portfolio.Infrastructure` e testes). Usar a separação de responsabilidades e bases abstratas como referência, sem copiar dependências ou comportamento que não sirvam ao CRM.
- Referência de acesso a dados: `../../FCopel.Portal/FCopel.Portal/FCopel.Portal.API/Repositories/`. O pacote usado ali é **Dapper**; “Deeper” foi o nome empregado na solicitação. `Repository.cs` concentra criação de conexões, e `AgendaOnlineRepository.cs` exemplifica consultas parametrizadas e transações. Não transportar seu modelo de múltiplos bancos Oracle/SQL Server para este projeto sem necessidade.
- Em caso de divergência entre documentação e comportamento do front, registrar a divergência e confirmar a regra de negócio antes de persistir uma interpretação definitiva.

## Arquitetura pretendida para a implementação futura

- Manter quatro camadas: **API** para HTTP, autenticação e tratamento de erros; **Application** para casos de uso, DTOs e validação de entrada; **Domain** para entidades, regras, estados, interfaces e serviços de domínio; **Infrastructure** para repositórios Dapper, conexão, transações, SQL, views e serviços externos. Testes separados por responsabilidade.
- Manter dependências apontando para dentro: Domain independente de banco, Dapper e ASP.NET; Application depende de Domain; Infrastructure implementa contratos internos; API compõe as dependências. Controllers devem ser finos.
- Usar classes abstratas onde houver invariantes comuns reais, como entidade base/auditada e bases de controller, serviço ou repositório. O portfólio traz `Entity`, `EntityAudited`, `BaseController<TEntity,TDto>`, `AppService<TEntity,TDto>`, `DomainService<T>` e `GenericRepository<T>`. Adaptar as bases a Dapper e aos casos de uso; não presumir que todo fluxo é CRUD genérico. Operações como confirmar, cancelar, registrar falha e receber pagamento precisam de casos de uso explícitos.
- Acesso ao banco com Dapper na Infrastructure, consultas SQL parametrizadas, mapeamento explícito e métodos assíncronos com `CancellationToken` quando pertinente. Definir o SGBD antes de escolher driver, sintaxe SQL e migração. Não interpolar valores do usuário em SQL.
- Gerenciar conexão e transação de forma que várias escritas de um caso de uso compartilhem a mesma transação. A abstração de Unit of Work do portfólio é referência conceitual, mas a implementação EF Core (`DbContext`, `SaveChanges`, cache) não deve ser copiada para Dapper.
- Planejar views versionadas para leituras compostas, como quadros, financeiro, dashboard e log, quando simplificarem consultas. Views comuns não garantem maior velocidade por si: definir filtros e paginação, índices nas tabelas base e medir planos de execução com dados representativos. Não escrever em views sem decisão explícita.
- Preservar histórico imutável de eventos relevantes, autor e instante; campos de último autor/resumo, se existirem, não substituem auditoria. Isolar autorização no backend por ação e área, sem depender das restrições visuais do front.

## Regras de negócio já validadas no front

- **Encomendas:** cinco etapas ativas: Em orçamento, Confirmadas, Em impressão, Prontas para entrega e Entregues. Avanços são sequenciais; pagamento é independente da etapa. Confirmar exige preço calculável, valida todos os saldos de filamento e insumos por unidade, registra baixas e congela o preço e dados usados. Consumo real informado substitui somente a quantidade baixada, sem refazer o orçamento. Retorno de Confirmadas a orçamento só antes da impressão e sem recebimento, devolvendo as baixas registradas. Cancelamento antes de Entregues tem efeito no estoque conforme a etapa; recebimentos passados continuam no histórico, e a cobrança restante termina.
- **Falhas de impressão:** Encomendas e Consignado registram falha inteira ou parcial nas etapas Em impressão/Prontas para entrega. O evento preserva material, horas e custo estimado, não altera etapa nem caixa e bloqueia avanço até nova tentativa. A nova tentativa valida e baixa apenas o filamento adicional.
- **Precificação:** material = soma de `gramas_i × preço_por_kg_i / 1000`; energia = `(potência_W / 1000) × horas × tarifa_R$/kWh`; preço sugerido = `(material + energia) × (1 + markupPercent / 100)`. O rótulo de tela “Margem de lucro” representa esse acréscimo sobre custos diretos, não margem verdadeira sobre venda nem lucro líquido. Preço por unidade e desconto final são arredondados a centavos. Preço manual do catálogo de Produtos é distinto do orçamento calculado da encomenda.
- **Consignado:** cada registro relaciona um produto e quantidade a um contato. Entrega, venda de unidades e recebimento do repasse são eventos distintos. Só unidades vendidas geram valor a receber; acertos parciais não devem mudar artificialmente a etapa operacional.
- **Financeiro e insumos:** recebimentos parciais são lançamentos datados e não avançam etapa. Compra paga de filamento aumenta saldo, atualiza preço e cria despesa na mesma transação; ajuste manual de saldo não cria despesa. A variação de caixa é recebido menos pago no período, sem equivaler a lucro ou saldo bancário.
- **Usuários:** administrador cadastra funcionário com e-mail único e permissões por área. Gerenciar implica Visualizar; Dashboard e Log são só leitura. Convite por e-mail para definição de senha, login e validação de permissões em toda operação protegida pertencem ao backend. A prévia do front não autentica.

## Decisões ainda abertas

- Escolher SGBD, estratégia de migrações e versionamento das views; definir volumes, filtros e metas de desempenho antes de otimizar.
- Fechar contratos HTTP, identificadores, associação de contatos, estados de consignação, idempotência e concorrência das transações. Os códigos e registros mockados não são definitivos.
- Definir tokens/sessões, validade e reenvio de convites, recuperação de senha, revogação, organização/isolamento de dados para futuro SaaS e associação futura à conta Google.
- Definir correções e estornos após confirmação, entrega, venda ou pagamento; consumo previsto versus real, desperdícios, insumos de novas tentativas e devoluções ao estoque; compras ainda não pagas e contas a pagar.
- Definir o formato completo de eventos de auditoria e a abrangência da busca em Log. Não tratar as views como fonte primária para eventos ou movimentações financeiras.

## Ao iniciar a implementação

- Consultar a seção correspondente do front e o `backend-handoff.md` antes de modelar cada fluxo. Distinguir regra validada, exemplo fictício e pendência.
- Projetar primeiro invariantes, operações atômicas e consultas necessárias; então definir esquema, contratos, views e índices. Cobrir transições, saldos, recebimentos e autorização com testes de comportamento, inclusive falha e concorrência quando relevante.
- Manter este arquivo atualizado à medida que decisões forem fechadas e o backend ganhar implementação real.
