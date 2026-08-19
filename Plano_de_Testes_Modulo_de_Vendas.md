# Plano de Testes — Módulo de Vendas ERP

## 1. Resumo do fluxo ponta a ponta

1. Validar empresa, usuário, vendedor, caixa e configurações comerciais/fiscais.
2. Identificar cliente e carregar suas regras comerciais e de crédito.
3. Selecionar tabela de preço, produtos, estoque, lote/serial e quantidades.
4. Calcular subtotal, descontos, acréscimos, frete, despesas e total.
5. Informar uma ou mais formas de pagamento, parcelamento ou rateio bancário.
6. Finalizar, suspender ou cancelar a venda.
7. Persistir a venda de forma atômica e produzir os efeitos em estoque, caixa, bancos, contas a receber, limite, comissão e relatórios.
8. Quando solicitado, faturar e emitir NF-e/NFC-e ou gerar boletos, mantendo rastreabilidade e evitando duplicidade.

## 2. Regras de negócio identificadas

- Dados e operações devem respeitar a empresa da sessão.
- Produtos controlados exigem lote ou serial válido antes da conclusão.
- O total pago deve fechar o total da venda; troco só pode existir em modalidade autorizada.
- Venda a prazo exige cliente identificado e condição de crédito válida.
- Toda operação financeira, fiscal ou de estoque deve ser idempotente e rastreável.
- Cancelamentos devem estornar apenas os efeitos efetivamente criados.
- Uma falha parcial não pode deixar estoque, financeiro ou documento sem vínculo consistente com a venda.
- Valores monetários devem usar regra única de precisão e arredondamento.

## 3. Regras a confirmar com Produto/Desenvolvimento

| Regra a confirmar | Impacto se indefinida | Risco |
|---|---|---|
| Venda sem cliente é permitida? Em quais valores e para NF-e/NFC-e? | Altera obrigatoriedade e documento fiscal | Fiscal/alto |
| Trocar vendedor recalcula tabela e itens já incluídos? | Pode alterar total sem clareza | Financeiro/alto |
| Trocar cliente ou tabela recalcula preços automaticamente ou pede confirmação? | Pode sobrescrever desconto negociado | Financeiro/alto |
| Venda suspensa reserva estoque? Há expiração? | Pode permitir venda duplicada do último item | Estoque/crítico |
| Estoque negativo é permitido por empresa, produto ou departamento? | Define bloqueio de conclusão | Estoque/crítico |
| Produtos repetidos são agrupados ou mantidos em linhas distintas? | Afeta lote, serial, promoção e desconto | Médio |
| Quais pagamentos aceitam troco? | Pode gerar caixa incorreto | Financeiro/alto |
| É permitido repetir o mesmo tipo de pagamento? | Afeta múltiplos cartões/contas | Financeiro/alto |
| Desconto máximo é definido por usuário, produto, tabela ou empresa? Qual prevalece? | Risco de venda abaixo do autorizado | Financeiro/crítico |
| Arredondamento ocorre por item, tributo, parcela ou somente no total? | Diferença de centavos | Fiscal/alto |
| Cancelamento após NF-e autorizada exige primeiro cancelar o documento? | Pode criar divergência fiscal | Fiscal/crítico |
| Falha fiscal impede a venda ou permite faturamento posterior? | Define atomicidade do processo | Fiscal/crítico |
| Cliente inadimplente bloqueia ou apenas alerta? Quem pode liberar? | Exposição de crédito | Financeiro/crítico |
| Vendedor da comissão é o vigente na conclusão ou na inclusão do item? | Divergência em comissão/ranking | Alto |
| Faturamento agrupado admite vendas com clientes, empresas ou condições diferentes? | Risco de documento inválido | Fiscal/crítico |

## 4. Matriz de riscos

| Risco | Probabilidade | Impacto | Nível | Controles/testes |
|---|---:|---:|---|---|
| Cobrança ou venda duplicada | Média | Crítico | Crítico | CT-020, CT-029 |
| Estoque negativo ou dupla baixa | Alta | Crítico | Crítico | CT-013, CT-015, CT-030 |
| Venda criada sem financeiro/estoque | Média | Crítico | Crítico | CT-021, CT-027 |
| Vazamento entre empresas | Média | Crítico | Crítico | CT-003, CT-004, CT-028 |
| Crédito acima do limite | Média | Alto | Alto | CT-023 |
| Boleto duplicado após nova tentativa | Média | Alto | Alto | CT-025 |
| Documento fiscal duplicado/incorreto | Média | Crítico | Crítico | CT-026, CT-027 |
| Erro de centavos em total/parcelas | Alta | Alto | Alto | CT-017 a CT-019, CT-024 |
| Comissão/relatório inconsistente | Média | Alto | Alto | CT-006, CT-032 |
| Perda do carrinho após falha | Média | Médio | Médio | CT-021, CT-031 |

## 5. Smoke test crítico

Executar antes da regressão completa: **CT-001, CT-003, CT-005, CT-008, CT-011, CT-013, CT-016, CT-017, CT-020, CT-022, CT-023, CT-025, CT-026, CT-027 e CT-032**.

## 6 a 10. Casos de teste

> Convenção de evidência: captura da tela, payload e resposta da API, IDs gerados, consulta posterior e registros dos módulos impactados. Onde não houver acesso à API ou banco, validar pela interface e módulos relacionados.

### CT-001 — Finalizar venda simples em dinheiro

- **Objetivo:** comprovar o fluxo mínimo de venda e seus efeitos.
- **Classificação:** feliz | **Etapa:** finalização | **Prioridade:** crítica | **Tipo:** funcional/integração/regressão.
- **Risco coberto:** venda ou efeitos não persistidos.
- **Pré-condições:** empresa e usuário ativos; permissão de venda; caixa aberto; vendedor, cliente consumidor final, produto e tabela ativos; estoque 10 UN.
- **Massa:** produto P-NORMAL, 2 UN × R$ 50,00; dinheiro R$ 100,00.
- **Passos e resultados esperados:**
  1. Abrir nova venda → empresa, vendedor e tabela corretos são apresentados.
  2. Adicionar 2 UN → subtotal e total ficam em R$ 100,00.
  3. Informar R$ 100,00 em dinheiro → restante fica R$ 0,00 e troco R$ 0,00.
  4. Finalizar → uma única venda é criada e mensagem de sucesso contém referência consultável.
  5. Atualizar e consultar → valores e vínculos permanecem iguais.
- **Resultado final:** venda concluída; estoque 8 UN; movimento de caixa +R$ 100,00; listagem, histórico e relatório refletem uma única venda.
- **Pós-condições:** venda disponível para faturamento conforme configuração.
- **Evidências:** tela antes/depois, ID, detalhe da venda, saldo do produto e movimento do caixa.
- **BDD:** Dado um caixa aberto e um produto com saldo, quando concluo a venda pagando o valor exato em dinheiro, então uma única venda e seus efeitos corretos são persistidos.

### CT-002 — Bloquear venda com caixa fechado, bloqueado ou em auditoria

- **Objetivo:** impedir movimentação fora de caixa operacional.
- **Classificação:** triste | **Etapa:** pré-condição | **Prioridade:** crítica | **Tipo:** funcional/regressão.
- **Pré-condições:** repetir com caixa fechado, bloqueado e em auditoria; usuário autorizado.
- **Massa:** 1 UN × R$ 10,00.
- **Passos e resultados esperados:** 1. Abrir venda → sistema bloqueia imediatamente ou identifica o estado; 2. tentar finalizar → ação indisponível ou erro de negócio claro; 3. consultar módulos → nenhum registro é criado.
- **Resultado final:** sem venda, baixa, recebimento ou numeração consumida indevidamente.
- **Pós-condições:** caixa permanece no estado original.
- **Evidências:** estado do caixa, mensagem, ausência na listagem e nos movimentos.
- **Risco:** venda fora de turno/auditoria.
- **BDD:** Dado um caixa não operacional, quando tento concluir uma venda, então a operação é recusada sem efeitos parciais.

### CT-003 — Isolamento de dados entre empresas

- **Objetivo:** garantir segregação multiempresa.
- **Classificação:** triste | **Etapa:** empresa | **Prioridade:** crítica | **Tipo:** segurança/funcional.
- **Pré-condições:** empresas A e B; usuário na A; vendedor, produto, estoque, tabela e conta exclusivos da B.
- **Massa:** IDs válidos da empresa B.
- **Passos e resultados esperados:** 1. Pesquisar cada cadastro na empresa A → não aparece; 2. tentar informar os IDs da B pela requisição → API retorna 403/404 ou erro de negócio sem revelar dados; 3. consultar listagens → nenhum dado da B é exibido.
- **Resultado final:** nenhum vínculo cruzado ou alteração na empresa B.
- **Pós-condições:** dados intactos.
- **Evidências:** pesquisas, requisições e respostas sanitizadas.
- **Risco:** vazamento e corrupção multi-tenant.
- **BDD:** Dado que estou na empresa A, quando tento acessar recursos exclusivos da B, então o sistema nega o acesso sem expor informações.

### CT-004 — Trocar empresa com venda em andamento

- **Objetivo:** evitar carrinho órfão ou associação à empresa errada.
- **Classificação:** borda | **Etapa:** empresa | **Prioridade:** crítica | **Tipo:** funcional/segurança.
- **Pré-condições:** venda iniciada na A com produto, desconto e pagamento parcial; acesso também à B.
- **Massa:** total R$ 100,00; pagamento parcial R$ 40,00.
- **Passos e resultados esperados:** 1. Solicitar troca para B → sistema alerta sobre dados pendentes; 2. confirmar descarte ou cancelar troca → comportamento é explícito; 3. se confirmar, abrir venda na B → carrinho, pagamentos e IDs da A não são reaproveitados.
- **Resultado final:** não existe venda híbrida nem efeito parcial na A/B.
- **Pós-condições:** sessão aponta somente para a empresa escolhida.
- **Evidências:** confirmação, estado antes/depois e ausência de registros.
- **Risco:** associação financeira e fiscal à empresa incorreta.
- **BDD:** Dado um carrinho da empresa A, quando troco para B, então o sistema impede o reaproveitamento silencioso dos dados da A.

### CT-005 — Aplicar vendedor padrão

- **Objetivo:** validar seleção automática e persistência do vendedor.
- **Classificação:** feliz | **Etapa:** vendedor | **Prioridade:** alta | **Tipo:** funcional/integração.
- **Pré-condições:** vendedor V1 ativo, vinculado à empresa e definido como padrão.
- **Massa:** venda de R$ 200,00; comissão configurada em 5%.
- **Passos e resultados esperados:** 1. Abrir venda → V1 aparece selecionado; 2. concluir → venda registra V1; 3. consultar comissão/ranking → base R$ 200,00 e comissão R$ 10,00, se a regra for sobre venda bruta.
- **Resultado final:** mesmo vendedor em venda, comissão e relatórios.
- **Pós-condições:** comissão rastreável.
- **Evidências:** cabeçalho, detalhe, comissão e ranking.
- **Risco:** atribuição incorreta de venda.
- **BDD:** Dado um vendedor padrão válido, quando finalizo uma venda, então ele é registrado consistentemente em todos os módulos.

### CT-006 — Alterar vendedor após itens, desconto e pagamento

- **Objetivo:** validar efeitos comerciais da troca tardia.
- **Classificação:** borda | **Etapa:** vendedor/tabela | **Prioridade:** alta | **Tipo:** funcional/regressão.
- **Pré-condições:** V1 e V2 com tabelas/limites de desconto diferentes.
- **Massa:** item a R$ 100,00; desconto R$ 10,00; pagamento parcial R$ 50,00.
- **Passos e resultados esperados:** 1. Montar venda com V1 → total R$ 90,00; 2. trocar para V2 → sistema aplica a regra confirmada e informa qualquer recálculo; 3. validar pagamento → restante é recalculado; 4. concluir → somente V2 é persistido, salvo regra documentada diferente.
- **Resultado final:** preço, desconto, pagamento e vendedor ficam coerentes.
- **Pós-condições:** comissão atribuída conforme regra confirmada.
- **Evidências:** valores antes/depois e detalhe persistido.
- **Risco:** desconto/comissão indevidos. **Regra a confirmar:** recálculo e vendedor prevalente.
- **BDD:** Dado uma venda parcialmente configurada, quando altero o vendedor, então as regras afetadas são recalculadas ou preservadas com confirmação explícita.

### CT-007 — Pesquisar e cadastrar cliente durante a venda

- **Objetivo:** validar busca e retorno ao fluxo sem perda do carrinho.
- **Classificação:** feliz | **Etapa:** cliente | **Prioridade:** alta | **Tipo:** funcional/usabilidade.
- **Pré-condições:** carrinho com um item; permissão para cadastrar cliente.
- **Massa:** nome, telefone, CPF válido e único, endereço completo.
- **Passos e resultados esperados:** 1. Pesquisar por código, nome, CPF e telefone → cliente correto é encontrado; 2. abrir cadastro e salvar novo cliente → validações são aplicadas; 3. retornar à venda → novo cliente fica selecionado e carrinho permanece; 4. atualizar → vínculo é persistido somente após salvar/suspender conforme desenho.
- **Resultado final:** cliente único criado e associado sem perda de dados.
- **Pós-condições:** cadastro disponível na empresa autorizada.
- **Evidências:** pesquisas, cadastro e carrinho.
- **Risco:** duplicidade e abandono de venda.
- **BDD:** Dado um carrinho em andamento, quando cadastro um cliente válido, então retorno à mesma venda com o cliente selecionado e os itens preservados.

### CT-008 — Bloquear CPF/CNPJ inválido ou duplicado

- **Objetivo:** preservar qualidade cadastral.
- **Classificação:** triste | **Etapa:** cliente | **Prioridade:** alta | **Tipo:** funcional/regressão.
- **Pré-condições:** documento já cadastrado e massas com dígito verificador inválido.
- **Massa:** CPF inválido 111.111.111-11; CPF/CNPJ existente na mesma empresa.
- **Passos e resultados esperados:** 1. Informar inválido → erro no campo e salvamento bloqueado; 2. informar duplicado → sistema localiza/informa cadastro existente sem criar outro; 3. voltar à venda → carrinho permanece.
- **Resultado final:** nenhum cliente inválido/duplicado criado.
- **Pós-condições:** quantidade de cadastros inalterada.
- **Evidências:** mensagens e pesquisa posterior.
- **Risco:** documento fiscal e cobrança com dados incorretos.
- **BDD:** Dado um documento inválido ou já existente, quando tento cadastrar o cliente, então o sistema bloqueia a duplicidade sem perder a venda.

### CT-009 — Trocar cliente após definir itens e pagamento

- **Objetivo:** validar atualização de tabela, desconto, crédito e condição comercial.
- **Classificação:** borda | **Etapa:** cliente | **Prioridade:** crítica | **Tipo:** funcional/integração.
- **Pré-condições:** C1 com tabela A e crédito; C2 bloqueado ou com tabela B.
- **Massa:** carrinho R$ 500,00; crediário parcial R$ 300,00.
- **Passos e resultados esperados:** 1. Montar com C1; 2. trocar para C2 → sistema alerta e reaplica as regras confirmadas; 3. verificar preço/desconto → sem alteração silenciosa; 4. verificar crediário → pagamento incompatível é removido/bloqueado com explicação; 5. concluir → apenas dados de C2 são usados.
- **Resultado final:** não há crédito, tabela ou endereço herdado de C1.
- **Pós-condições:** venda coerente com C2.
- **Evidências:** antes/depois, limite e detalhe final.
- **Risco:** concessão de crédito indevido. **Regra a confirmar:** estratégia de recálculo.
- **BDD:** Dado uma venda configurada para C1, quando seleciono C2, então todas as dependências do cliente são revalidadas antes da conclusão.

### CT-010 — Vigência e prioridade da tabela de preço

- **Objetivo:** escolher somente tabela aplicável.
- **Classificação:** borda | **Etapa:** tabela | **Prioridade:** alta | **Tipo:** funcional/regressão.
- **Pré-condições:** tabelas vencida, futura e ativa; mais de uma aplicável ao vendedor/cliente/empresa.
- **Massa:** data/hora nos limites inicial e final; produto com preços R$ 90,00/R$ 100,00.
- **Passos e resultados esperados:** 1. Abrir antes, exatamente no início, exatamente no fim e após o fim → inclusão/exclusão respeita precisão documentada; 2. selecionar contexto com múltiplas tabelas → prioridade determinística; 3. atualizar a página → mesma tabela e preço.
- **Resultado final:** somente tabela vigente e prioritária é aplicada.
- **Pós-condições:** venda registra ID e preço de origem.
- **Evidências:** relógio, tabela selecionada, preço e payload.
- **Risco:** preço incorreto. **Regra a confirmar:** inclusividade das datas, fuso e prioridade.
- **BDD:** Dado tabelas com vigências e vínculos diferentes, quando abro a venda, então é aplicada apenas a tabela vigente de maior prioridade.

### CT-011 — Produto sem preço, com preço zero ou negativo

- **Objetivo:** impedir preço inválido ou exigir autorização explícita.
- **Classificação:** triste | **Etapa:** tabela/produto | **Prioridade:** crítica | **Tipo:** funcional/segurança.
- **Pré-condições:** três produtos: ausente da tabela, preço R$ 0,00 e preço -R$ 1,00.
- **Massa:** quantidades 1 UN.
- **Passos e resultados esperados:** 1. Adicionar cada produto → sistema bloqueia ou solicita preço/autorização conforme regra; 2. manipular preço negativo na API → rejeição 4xx; 3. consultar → nenhuma venda/baixa é gerada.
- **Resultado final:** total inválido não é concluído.
- **Pós-condições:** estoque inalterado.
- **Evidências:** mensagens e resposta API.
- **Risco:** perda financeira. **Regra a confirmar:** venda gratuita/preço manual.
- **BDD:** Dado um produto sem preço comercial válido, quando tento vendê-lo, então a conclusão é bloqueada ou autorizada de forma auditável.

### CT-012 — Pesquisar produto e tratar GTIN duplicado

- **Objetivo:** garantir seleção inequívoca.
- **Classificação:** borda | **Etapa:** produtos | **Prioridade:** média | **Tipo:** funcional/usabilidade.
- **Pré-condições:** produtos pesquisáveis por código, nome, referência e GTIN; dois cadastros com mesmo GTIN em condição permitida.
- **Massa:** termos completos e parciais.
- **Passos e resultados esperados:** 1. Pesquisar em cada campo → resultados corretos, paginados e contados; 2. ler GTIN duplicado → sistema apresenta seleção inequívoca ou bloqueia duplicidade cadastral; 3. selecionar → item escolhido corresponde a código, unidade e preço exibidos.
- **Resultado final:** produto correto incluído uma vez.
- **Pós-condições:** carrinho consistente.
- **Evidências:** filtros, resultados e item.
- **Risco:** venda do SKU errado.
- **BDD:** Dado produtos pesquisáveis por múltiplos identificadores, quando realizo a busca, então seleciono inequivocamente o cadastro correto.

### CT-013 — Vender acima do estoque com bloqueio ativo

- **Objetivo:** impedir saldo indevido.
- **Classificação:** triste | **Etapa:** estoque | **Prioridade:** crítica | **Tipo:** funcional/integração.
- **Pré-condições:** estoque negativo bloqueado; saldo 5 UN no departamento D1.
- **Massa:** solicitar 6 UN.
- **Passos e resultados esperados:** 1. Adicionar 6 → sistema informa disponível 5; 2. tentar finalizar → bloqueio também na API; 3. consultar saldo → continua 5.
- **Resultado final:** nenhuma venda ou baixa parcial.
- **Pós-condições:** saldo preservado.
- **Evidências:** saldo antes/depois e resposta.
- **Risco:** estoque negativo.
- **BDD:** Dado saldo 5 e bloqueio de negativo, quando tento vender 6, então a operação é recusada sem alterar o estoque.

### CT-014 — Distribuir quantidade entre lotes

- **Objetivo:** validar soma, validade, saldo e rastreabilidade.
- **Classificação:** feliz/borda | **Etapa:** lote | **Prioridade:** crítica | **Tipo:** funcional/integração.
- **Pré-condições:** produto por lote; L1 válido saldo 2; L2 válido saldo 3; L3 vencido saldo 10.
- **Massa:** venda 5 UN: L1=2 e L2=3.
- **Passos e resultados esperados:** 1. Solicitar 5 → seleção obrigatória; 2. tentar L3 → bloqueio/alerta conforme regra; 3. distribuir 2+3 → total selecionado 5; 4. concluir → baixas exatas por lote; 5. consultar documento/detalhe → lotes rastreáveis.
- **Resultado final:** L1 e L2 zerados; L3 intacto; total baixado 5.
- **Pós-condições:** movimentos vinculados à venda.
- **Evidências:** seleção, saldos e rastreabilidade.
- **Risco:** lote incorreto/vencido. **Regra a confirmar:** venda de lote vencido com autorização.
- **BDD:** Dado dois lotes válidos que totalizam a quantidade, quando distribuo e concluo, então cada lote é baixado exatamente conforme selecionado.

### CT-015 — Serial obrigatório, duplicado, inexistente ou de outra empresa

- **Objetivo:** garantir unicidade e propriedade do serial.
- **Classificação:** triste/borda | **Etapa:** serial | **Prioridade:** crítica | **Tipo:** funcional/segurança.
- **Pré-condições:** produto serializado; S1 disponível; S2 já vendido; S3 da empresa B.
- **Massa:** quantidade 2; testar ausência, S1 repetido, S2, inexistente e S3.
- **Passos e resultados esperados:** 1. Concluir sem dois seriais → bloqueio; 2. repetir S1 → duplicidade bloqueada; 3. testar S2/inexistente/S3 → rejeição sem exposição da empresa B; 4. usar dois seriais válidos → conclusão permitida.
- **Resultado final:** um serial único e disponível por unidade, baixado uma vez.
- **Pós-condições:** seriais válidos vinculados à venda.
- **Evidências:** mensagens, movimentos e detalhe.
- **Risco:** perda de rastreabilidade/fraude.
- **BDD:** Dado um produto serializado, quando tento concluir com seriais inválidos ou insuficientes, então o sistema bloqueia a venda.

### CT-016 — Unidade, embalagem e fator de conversão

- **Objetivo:** validar preço e estoque em unidade base.
- **Classificação:** feliz/borda | **Etapa:** produto/estoque | **Prioridade:** alta | **Tipo:** funcional/integração.
- **Pré-condições:** caixa CX = 12 UN; estoque 24 UN; preço da caixa R$ 120,00.
- **Massa:** 1,5 CX, se fracionamento permitido.
- **Passos e resultados esperados:** 1. Adicionar 1 CX → total R$ 120,00 e baixa prevista 12 UN; 2. adicionar 0,5 CX → total adicional R$ 60,00 e baixa 6 UN; 3. concluir → total R$ 180,00 e baixa 18 UN; 4. se fracionamento proibido, 1,5 é recusado antes da conclusão.
- **Resultado final:** estoque final 6 UN quando permitido.
- **Pós-condições:** unidade comercial e base rastreáveis.
- **Evidências:** fator, cálculo e movimento.
- **Risco:** baixa e preço incorretos. **Regra a confirmar:** fracionamento por embalagem.
- **BDD:** Dado uma embalagem de 12 unidades, quando vendo 1,5 caixa permitido, então são cobrados R$ 180,00 e baixadas 18 unidades.

### CT-017 — Cálculo de subtotal, desconto e acréscimo

- **Objetivo:** validar ordem e precisão dos cálculos.
- **Classificação:** feliz | **Etapa:** cálculos | **Prioridade:** crítica | **Tipo:** funcional/regressão.
- **Pré-condições:** descontos/acréscimos permitidos.
- **Massa e fórmula:** 2 × R$ 50,00 + 3 × R$ 20,00 = subtotal R$ 160,00; desconto global 10% = R$ 16,00; acréscimo 5% sobre R$ 144,00 = R$ 7,20; total = R$ 151,20.
- **Passos e resultados esperados:** 1. Adicionar itens → R$ 160,00; 2. aplicar 10% → R$ 144,00; 3. aplicar 5% → R$ 151,20; 4. atualizar e concluir → valores permanecem.
- **Resultado final:** total R$ 151,20 e memória de cálculo auditável.
- **Pós-condições:** detalhes registram bases e valores.
- **Evidências:** tela, payload e detalhe.
- **Risco:** cobrança divergente. **Regra a confirmar:** ordem desconto/acréscimo.
- **BDD:** Dado subtotal R$ 160,00, quando aplico desconto de 10% e acréscimo de 5% sobre o líquido, então o total é R$ 151,20.

### CT-018 — Limites de desconto e valores inválidos

- **Objetivo:** bloquear total ou percentual impossível.
- **Classificação:** triste/borda | **Etapa:** cálculos | **Prioridade:** crítica | **Tipo:** funcional/segurança.
- **Pré-condições:** usuário com limite de desconto conhecido.
- **Massa:** total R$ 1,00; descontos 0%, 100%, 100,01%, R$ 1,01; acréscimo -1%; quantidade 0 e -1.
- **Passos e resultados esperados:** 1. Aplicar 0% → total R$ 1,00; 2. aplicar 100% → aceitar somente se regra permitir venda gratuita e registrar autorização; 3. aplicar >100%, valor maior que total, acréscimo negativo e quantidades não positivas → interface e API rejeitam; 4. concluir tentativas inválidas → nenhum efeito.
- **Resultado final:** nenhuma venda com total/quantidade inválida.
- **Pós-condições:** carrinho mantém último valor válido.
- **Evidências:** validações e resposta API.
- **Risco:** perda financeira/manipulação.
- **BDD:** Dado uma venda de R$ 1,00, quando envio desconto superior ao total ou quantidade não positiva, então o sistema recusa a entrada em todas as camadas.

### CT-019 — Arredondamento e diferença de um centavo

- **Objetivo:** garantir fechamento determinístico.
- **Classificação:** borda | **Etapa:** cálculos/parcelas | **Prioridade:** crítica | **Tipo:** funcional/regressão.
- **Pré-condições:** moeda BRL e precisão monetária definida.
- **Massa e fórmula:** 3 × R$ 0,05 = R$ 0,15; 1/3 de R$ 100,00 → R$ 33,33 + R$ 33,33 + R$ 33,34; testar itens R$ 0,01, R$ 0,10 e preço de origem R$ 1,005.
- **Passos e resultados esperados:** 1. Calcular cada massa → totais fecham em duas casas conforme regra; 2. parcelar R$ 100,00 em 3 → soma exata R$ 100,00; 3. finalizar → restante R$ 0,00, sem sobra de R$ 0,01; 4. reabrir → valores iguais.
- **Resultado final:** soma de itens/pagamentos/parcelas coincide exatamente com o total.
- **Pós-condições:** financeiro sem diferença residual.
- **Evidências:** cálculo e parcelas.
- **Risco:** venda não fecha ou título diverge. **Regra a confirmar:** half-up, half-even e etapa de arredondamento.
- **BDD:** Dado total R$ 100,00, quando divido em três parcelas, então as parcelas são R$ 33,33, R$ 33,33 e R$ 33,34, totalizando R$ 100,00.

### CT-020 — Múltiplos pagamentos e repetição do mesmo tipo

- **Objetivo:** validar composição, repetição e saldo.
- **Classificação:** feliz/borda | **Etapa:** pagamento | **Prioridade:** crítica | **Tipo:** funcional/integração.
- **Pré-condições:** dinheiro, Pix e crédito habilitados; regra permite dois cartões.
- **Massa:** total R$ 300,00; dinheiro R$ 50,00; Pix R$ 100,00; crédito cartão A R$ 70,00; crédito cartão B R$ 80,00.
- **Passos e resultados esperados:** 1. Adicionar pagamentos → restante evolui 250, 150, 80 e 0; 2. remover R$ 70,00 → restante R$ 70,00; 3. reinserir → restante zero; 4. concluir → uma venda e quatro recebimentos; 5. atualizar → composição preservada.
- **Resultado final:** soma R$ 300,00, sem duplicação ou perda.
- **Pós-condições:** caixa/adquirentes refletem os valores corretos.
- **Evidências:** pagamentos, transações e detalhe.
- **Risco:** recebimento incorreto. **Regra a confirmar:** repetição do mesmo tipo.
- **BDD:** Dado uma venda de R$ 300,00, quando pago com quatro lançamentos que somam o total, então a venda registra exatamente os quatro recebimentos.

### CT-021 — Falha/timeout após acionar finalização

- **Objetivo:** preservar atomicidade e permitir recuperação segura.
- **Classificação:** triste | **Etapa:** finalização | **Prioridade:** crítica | **Tipo:** integração/regressão.
- **Pré-condições:** possibilidade de simular timeout/erro 5xx em etapas distintas.
- **Massa:** venda R$ 100,00 com produto, Pix e cliente.
- **Passos e resultados esperados:** 1. Interromper antes da criação → mensagem recuperável e nenhum efeito; 2. interromper após criação e antes da resposta → nova consulta identifica o resultado; 3. tentar novamente → mesma chave idempotente ou reconciliação, sem nova cobrança/venda; 4. atualizar → carrinho não desaparece sem confirmação.
- **Resultado final:** estado final é totalmente concluído ou totalmente revertido/reconciliável.
- **Pós-condições:** nenhuma operação órfã.
- **Evidências:** horário, chave, IDs e consultas cruzadas.
- **Risco:** venda, cobrança ou baixa órfã.
- **BDD:** Dado uma falha durante a conclusão, quando repito a operação, então o sistema recupera o resultado anterior sem duplicar efeitos.

### CT-022 — Suspender e retomar venda

- **Objetivo:** preservar dados e aplicar política de reserva.
- **Classificação:** feliz/borda | **Etapa:** suspensão | **Prioridade:** alta | **Tipo:** funcional/integração.
- **Pré-condições:** carrinho com cliente, vendedor, tabela, desconto, observação e lote/serial.
- **Massa:** 2 itens; total R$ 250,00.
- **Passos e resultados esperados:** 1. Suspender → identificador e confirmação; 2. verificar estoque → reserva ou não reserva conforme regra, nunca baixa definitiva; 3. retomar pelo mesmo usuário → todos os dados voltam; 4. atualizar → estado preservado; 5. retomar por outro usuário/empresa → somente se autorizado, sem cruzamento de empresa.
- **Resultado final:** retomada fiel e auditável; conclusão posterior gera efeitos uma única vez.
- **Pós-condições:** suspensão removida após conclusão/cancelamento.
- **Evidências:** antes/depois, reserva e auditoria.
- **Risco:** perda de carrinho/dupla venda. **Regra a confirmar:** reserva, expiração e permissão de retomada.
- **BDD:** Dado uma venda suspensa, quando a retomo com autorização, então todos os dados retornam sem baixa definitiva ou duplicidade.

### CT-023 — Crediário com limite insuficiente ou cliente inadimplente

- **Objetivo:** impedir crédito não autorizado.
- **Classificação:** triste | **Etapa:** crediário | **Prioridade:** crítica | **Tipo:** funcional/integração.
- **Pré-condições:** C1 limite disponível R$ 200,00; C2 inadimplente/bloqueado.
- **Massa:** venda R$ 500,00; entrada R$ 100,00; saldo a prazo R$ 400,00.
- **Passos e resultados esperados:** 1. Selecionar C1 e crediário → limite insuficiente de R$ 200,00 é informado; 2. tentar concluir via interface/API → bloqueio ou liberação formal auditada; 3. repetir com C2 → bloqueio conforme regra; 4. consultar → nenhum título/uso de limite parcial.
- **Resultado final:** crédito só é criado após aprovação válida.
- **Pós-condições:** limite permanece inalterado nas recusas.
- **Evidências:** limite antes/depois, mensagem e ausência de títulos.
- **Risco:** inadimplência. **Regra a confirmar:** alerta versus bloqueio e alçada de liberação.
- **BDD:** Dado saldo a prazo maior que o limite, quando tento concluir em crediário, então o sistema bloqueia ou exige autorização auditável.

### CT-024 — Crediário: entrada, parcelas, juros e vencimentos

- **Objetivo:** validar plano e contas a receber.
- **Classificação:** feliz/borda | **Etapa:** crediário | **Prioridade:** crítica | **Tipo:** funcional/integração.
- **Pré-condições:** cliente apto, limite R$ 1.000,00; plano de 3 parcelas a cada 30 dias.
- **Massa e fórmula:** venda R$ 400,00; entrada R$ 100,00; saldo R$ 300,00; três parcelas de R$ 100,00. Para juros de 3% no saldo, se aplicável: R$ 300,00 × 1,03 = R$ 309,00; 3 × R$ 103,00.
- **Passos e resultados esperados:** 1. Informar entrada → saldo R$ 300,00; 2. selecionar plano → datas e valores corretos; 3. concluir → entrada no caixa e três títulos; 4. consultar limite → uso aumenta pelo principal/regra confirmada; 5. cancelar → títulos, limite e entrada são estornados corretamente.
- **Resultado final:** soma das parcelas igual ao saldo financiado com encargos explícitos.
- **Pós-condições:** títulos vinculados à venda.
- **Evidências:** cronograma, títulos, caixa e limite.
- **Risco:** parcelas/limite divergentes. **Regra a confirmar:** base de juros e consumo do limite.
- **BDD:** Dado venda de R$ 400,00 com entrada de R$ 100,00, quando financio R$ 300,00 em três parcelas sem juros, então são gerados três títulos de R$ 100,00.

### CT-025 — Falha na geração do boleto e nova tentativa

- **Objetivo:** evitar cobrança duplicada e venda inconsistente.
- **Classificação:** triste | **Etapa:** boleto | **Prioridade:** crítica | **Tipo:** integração/regressão.
- **Pré-condições:** venda criada; banco/carteira configurados; simular falha após reservar identificador e antes de retornar boleto.
- **Massa:** boleto R$ 150,00, vencimento +10 dias, multa 2%, juros 1% a.m.
- **Passos e resultados esperados:** 1. Gerar boleto e provocar falha → status pendente/falha claro; 2. tentar novamente → reutiliza/reconcilia operação, sem segundo título; 3. consultar provedor/financeiro → apenas um identificador ativo; 4. gerar segunda via → mesmos dados de cobrança.
- **Resultado final:** uma venda, um título e um boleto válido.
- **Pós-condições:** falha anterior auditada.
- **Evidências:** IDs, linha digitável, título e logs funcionais.
- **Risco:** boleto duplicado/cobrança dupla.
- **BDD:** Dado uma falha após iniciar a geração do boleto, quando tento novamente, então o sistema retorna ou reconcilia a mesma cobrança sem duplicá-la.

### CT-026 — Emitir NF-e/NFC-e autorizada

- **Objetivo:** validar faturamento fiscal e vínculos.
- **Classificação:** feliz | **Etapa:** faturamento | **Prioridade:** crítica | **Tipo:** integração/regressão.
- **Pré-condições:** homologação; certificado válido; série/número; operação fiscal; produto com NCM/CEST e tributos; cliente válido.
- **Massa:** venda concluída; testar NF-e com contribuinte e NFC-e consumidor final.
- **Passos e resultados esperados:** 1. Faturar → prévia usa empresa, cliente, itens e totais da venda; 2. transmitir → status processamento sem liberar retransmissão cega; 3. receber autorização → chave de 44 dígitos e protocolo; 4. consultar → documento vinculado à venda e XML/DANFE acessíveis; 5. repetir comando → documento não é duplicado.
- **Resultado final:** um documento autorizado por venda/faturamento e totais coerentes.
- **Pós-condições:** numeração consumida e histórico registrado.
- **Evidências:** venda, XML, chave, protocolo e status.
- **Risco:** documento inválido/duplicado.
- **BDD:** Dado uma venda fiscalmente válida, quando transmito o documento, então uma única NF-e/NFC-e é autorizada e vinculada à venda.

### CT-027 — Rejeição fiscal ou indisponibilidade da SEFAZ

- **Objetivo:** manter venda, estoque e financeiro coerentes diante de falha fiscal.
- **Classificação:** triste | **Etapa:** faturamento | **Prioridade:** crítica | **Tipo:** integração/regressão.
- **Pré-condições:** simular NCM/CFOP inválido, certificado vencido, rejeição e indisponibilidade.
- **Massa:** venda finalizada ainda não faturada.
- **Passos e resultados esperados:** 1. Transmitir com dado inválido → rejeição clara e corrigível; 2. conferir venda → não há segundo estoque/financeiro; 3. corrigir e retransmitir → uma autorização; 4. simular indisponibilidade → status não é marcado como autorizado; contingência somente se configurada; 5. atualizar → status real permanece.
- **Resultado final:** falha fiscal não duplica nem desfaz efeitos comerciais indevidamente; emissão posterior é possível conforme regra.
- **Pós-condições:** histórico guarda tentativas e motivo.
- **Evidências:** rejeição, status, consultas e autorização posterior.
- **Risco:** venda sem documento ou emissão duplicada. **Regra a confirmar:** venda síncrona versus faturamento posterior/contingência.
- **BDD:** Dado uma rejeição fiscal, quando corrijo e retransmito, então a venda permanece consistente e apenas um documento é autorizado.

### CT-028 — Manipular CompanyId, preço, desconto e estoque na API

- **Objetivo:** garantir validação server-side.
- **Classificação:** triste | **Etapa:** segurança | **Prioridade:** crítica | **Tipo:** segurança.
- **Pré-condições:** usuário comum autenticado na empresa A.
- **Massa:** CompanyId B; preço R$ 0,01 para item de R$ 1.000,00; desconto 150%; quantidade -1; departamento alheio.
- **Passos e resultados esperados:** 1. Interceptar requisição válida; 2. alterar um campo por vez → API rejeita com 4xx e mensagem segura; 3. reenviar IDs válidos sem permissão → 403/404; 4. consultar módulos → nenhum efeito.
- **Resultado final:** regras críticas não dependem do frontend.
- **Pós-condições:** tentativa registrada em auditoria quando aplicável.
- **Evidências:** requisições/respostas e ausência de registros.
- **Risco:** fraude financeira e acesso indevido.
- **BDD:** Dado uma requisição manipulada, quando envio valores ou IDs não autorizados, então a API recusa sem criar efeitos.

### CT-029 — Duplo clique e repetição da mesma requisição

- **Objetivo:** comprovar idempotência da conclusão.
- **Classificação:** borda | **Etapa:** finalização | **Prioridade:** crítica | **Tipo:** concorrência/segurança.
- **Pré-condições:** venda pronta e latência simulada.
- **Massa:** total R$ 99,90; pagamento eletrônico.
- **Passos e resultados esperados:** 1. Clicar duas vezes rapidamente → botão entra em carregamento e fica desabilitado; 2. repetir a mesma requisição/chave → resposta representa o mesmo resultado; 3. consultar → uma venda, uma baixa, uma cobrança e um financeiro.
- **Resultado final:** nenhuma duplicidade em qualquer módulo.
- **Pós-condições:** chave idempotente rastreável.
- **Evidências:** rede, IDs e movimentos.
- **Risco:** cobrança dupla.
- **BDD:** Dado uma venda pronta, quando aciono a finalização duas vezes, então somente uma operação efetiva é processada.

### CT-030 — Dois usuários vendendo o último item

- **Objetivo:** validar concorrência e consistência do saldo.
- **Classificação:** borda | **Etapa:** estoque | **Prioridade:** crítica | **Tipo:** concorrência/integração.
- **Pré-condições:** saldo 1 UN; estoque negativo bloqueado; usuários U1 e U2.
- **Massa:** ambos vendem 1 UN do mesmo produto/departamento.
- **Passos e resultados esperados:** 1. Ambos adicionam o item; 2. finalizar simultaneamente → apenas uma operação confirma; 3. a outra recebe indisponibilidade e mantém opção de corrigir carrinho; 4. consultar → saldo 0, uma venda e uma baixa.
- **Resultado final:** nunca saldo -1 nem duas vendas concluídas.
- **Pós-condições:** perdedor pode remover/substituir item.
- **Evidências:** horários, respostas, vendas e saldo.
- **Risco:** overselling.
- **BDD:** Dado apenas uma unidade disponível, quando dois usuários finalizam simultaneamente, então somente uma venda é confirmada.

### CT-031 — Atualização, perda de internet e sessão expirada

- **Objetivo:** validar recuperação segura do estado.
- **Classificação:** triste/borda | **Etapa:** resiliência | **Prioridade:** alta | **Tipo:** funcional/usabilidade/segurança.
- **Pré-condições:** carrinho com dados não sensíveis e pagamento ainda não confirmado.
- **Massa:** venda R$ 80,00.
- **Passos e resultados esperados:** 1. Atualizar antes de salvar → restaura rascunho ou alerta sobre perda conforme regra; 2. perder internet → não indica sucesso falso; 3. expirar sessão → solicita autenticação e não envia com contexto inválido; 4. autenticar novamente → recupera com empresa correta ou reconcilia; 5. consultar → não há operação fantasma.
- **Resultado final:** estado e mensagem correspondem ao que foi persistido.
- **Pós-condições:** dados sensíveis de pagamento não ficam expostos localmente.
- **Evidências:** estados, rede e consulta posterior.
- **Risco:** perda de trabalho/sucesso falso. **Regra a confirmar:** persistência de rascunho.
- **BDD:** Dado uma interrupção antes da resposta, quando restabeleço a sessão, então consigo identificar inequivocamente se a venda foi concluída.

### CT-032 — Consistência ponta a ponta após venda, cancelamento e estorno

- **Objetivo:** reconciliar todos os módulos.
- **Classificação:** feliz/triste | **Etapa:** integração | **Prioridade:** crítica | **Tipo:** integração/regressão.
- **Pré-condições:** venda completa com vendedor, cliente, lote/serial, desconto, duas formas, crediário e documento conforme combinação suportada.
- **Massa:** total R$ 1.000,00; dinheiro R$ 200,00; crediário R$ 800,00; 4 parcelas de R$ 200,00.
- **Passos e resultados esperados:** 1. Concluir → uma venda; 2. conferir estoque/lote/serial → baixa exata; 3. conferir caixa/bancos/receber/limite → valores e vínculos corretos; 4. conferir comissão, relatórios, ranking, ticket e margem → venda incluída uma vez; 5. cancelar no fluxo permitido → estornos inversos e auditáveis; 6. atualizar e consultar novamente → estado permanece cancelado e métricas excluem/segregam conforme regra.
- **Resultado final:** soma e status reconciliados entre venda, estoque, financeiro, fiscal e BI; nenhum órfão.
- **Pós-condições:** histórico conserva criação e estorno; documento fiscal segue procedimento legal próprio.
- **Evidências:** IDs cruzados e telas de todos os módulos.
- **Risco:** inconsistência sistêmica. **Regra a confirmar:** tratamento de canceladas nos indicadores.
- **BDD:** Dado uma venda concluída com efeitos integrados, quando ela é cancelada pelo fluxo permitido, então todos os efeitos reversíveis são estornados uma única vez e permanecem auditáveis.

### CT-033 — Ratear depósito entre contas bancárias

- **Objetivo:** validar distribuição e lançamentos bancários.
- **Classificação:** feliz/triste | **Etapa:** depósito | **Prioridade:** alta | **Tipo:** funcional/integração.
- **Pré-condições:** contas A e B ativas na empresa; conta C inativa; conta D de outra empresa.
- **Massa:** venda R$ 500,00; A=R$ 300,00, B=R$ 200,00; testar somas R$ 499,99 e R$ 500,01.
- **Passos e resultados esperados:** 1. Ratear 300+200 → saldo zero; 2. testar soma inferior/superior → conclusão bloqueada; 3. selecionar C/D → rejeição; 4. concluir massa válida → lançamentos A=300 e B=200; 5. cancelar → ambos estornados uma vez.
- **Resultado final:** total distribuído igual ao depósito e contas da empresa correta.
- **Pós-condições:** lançamentos vinculados à venda.
- **Evidências:** rateio, contas, lançamentos e estornos.
- **Risco:** saldo bancário incorreto.
- **BDD:** Dado depósito de R$ 500,00, quando rateio R$ 300,00 e R$ 200,00 em contas válidas, então são gerados dois lançamentos que totalizam R$ 500,00.

### CT-034 — Listagem, filtros, paginação e detalhes

- **Objetivo:** validar consulta consistente e isolada.
- **Classificação:** feliz/borda | **Etapa:** consulta | **Prioridade:** alta | **Tipo:** funcional/desempenho.
- **Pré-condições:** vendas com diferentes datas, clientes, vendedores, status, pagamentos, lote/serial e documentos.
- **Massa:** mais registros que o tamanho de página; cliente sem nome fantasia.
- **Passos e resultados esperados:** 1. Ordenar/paginar → sem duplicidade ou omissão; 2. pesquisar cliente → resultados corretos; 3. conferir ID, data/hora, cliente, vendedor e total → iguais ao detalhe; 4. abrir detalhes → pagamentos, lotes, seriais, parcelas, boletos e fiscais corretos; 5. trocar empresa/permissão → restrição aplicada; 6. medir volume → resposta dentro do SLA definido.
- **Resultado final:** listagem e detalhe consistentes, paginados e seguros.
- **Pós-condições:** nenhuma alteração de dados.
- **Evidências:** páginas, filtros, detalhe e tempo.
- **Risco:** informação operacional enganosa. **Regra a confirmar:** SLA e campo substituto sem nome fantasia.
- **BDD:** Dado um grande conjunto de vendas, quando navego e filtro a listagem, então não há registros ausentes, duplicados ou de outra empresa.

## 11. Checklist de regressão

- [ ] Empresa, permissões, usuário, caixa e vendedor.
- [ ] Cliente, limite, bloqueio, endereço e troca durante a venda.
- [ ] Tabela, vigência, prioridade, promoção, desconto e arredondamento.
- [ ] Produto normal, unidade/embalagem, estoque, lote e serial.
- [ ] Dinheiro, crédito, débito, Pix, cheque, depósito, vale, crediário e outros.
- [ ] Múltiplos pagamentos, repetição, troco, saldo e remoção.
- [ ] Suspender, retomar, cancelar e estornar.
- [ ] Boleto, segunda via, baixa, rejeição e nova tentativa.
- [ ] Faturamento, NF-e/NFC-e, rejeição, contingência e duplicidade.
- [ ] Estoque, caixa, bancos, contas a receber, limite e comissão.
- [ ] Relatórios, ranking, ticket, margem e fluxo de caixa.
- [ ] Listagem, filtros, paginação, detalhes e auditoria.
- [ ] Atualização, sessão expirada, perda de rede, concorrência e idempotência.
- [ ] Isolamento multiempresa e validações server-side.

## 12. Melhorias de usabilidade

- Exibir memória de cálculo do total, parcelas e troco.
- Indicar origem do preço e motivo de bloqueio do desconto.
- Alertar antes de trocar empresa, cliente, vendedor ou tabela quando houver impactos.
- Desabilitar “Finalizar” durante o processamento e mostrar estado reconciliável em timeout.
- Preservar carrinho após erro corrigível.
- Mostrar saldo, departamento, lote, validade e seriais em uma mesma etapa.
- Informar claramente o que será reservado ao suspender a venda.
- Oferecer atalho para corrigir o pagamento que deixou diferença de centavos.
- Padronizar mensagens em português, com causa e ação sugerida.

## 13. Funcionalidades ausentes ou incompletas a verificar

Não é possível confirmar apenas pelo arquivo se existem: reserva com expiração; chave de idempotência; reconciliação de pagamentos; autorização de desconto/crédito; histórico de troca de preço; contingência fiscal; faturamento agrupado; segunda via/retorno de boleto; estorno automático; auditoria detalhada; retomada entre usuários; e conciliação entre venda e módulos. A ausência comprovada deve ser registrada como **funcionalidade ausente**, não como defeito, até existir requisito aprovado.

## 14. Riscos de implementação

- Transações distribuídas sem compensação podem criar efeitos parciais.
- Confiar nos totais enviados pelo frontend permite manipulação.
- Falta de trava otimista/pessimista favorece venda concorrente do mesmo saldo.
- Retentativas sem idempotência duplicam venda, cobrança, boleto ou nota.
- Recalcular tabela/cliente silenciosamente pode alterar negociação já aceita.
- Arredondamentos diferentes entre web, API, fiscal e financeiro geram resíduos.
- Cancelar venda sem considerar o estado fiscal pode violar regras legais.
- Relatórios atualizados por processos assíncronos precisam de SLA e estado de processamento.

## 15. Massa de dados sugerida

| Entidade | Massa |
|---|---|
| Empresas | A ativa; B ativa; C inativa |
| Usuários | vendedor autorizado; sem permissão; gestor com alçada |
| Caixas | aberto; fechado; bloqueado; auditoria |
| Vendedores | V1 padrão; V2 outra tabela; V3 inativo; V4 de outra empresa |
| Clientes | consumidor final; contribuinte; bloqueado; inadimplente; limite 0/200/1.000; sem endereço |
| Produtos | normal; sem estoque; lote; serial; embalagem 12 UN; inativo; não compartilhado |
| Tabelas | ativa; vencida; futura; vazia; preços zero/negativo; vigência no limite |
| Estoque | 0, 1, 5, negativo permitido; dois departamentos; lotes válido/vencido/futuro |
| Valores | R$ 0,01; 0,05; 0,10; 1,00; 99,90; 100,00; 1.000,00; preço 1,005 |
| Pagamentos | exato; inferior; superior; repetido; timeout; falha após aprovação |
| Fiscal | certificado válido/vencido; série válida/inválida; homologação; NCM/CFOP/tributos válidos e ausentes |

## 16. Matriz de rastreabilidade

| Funcionalidade | Risco principal | Casos |
|---|---|---|
| Pré-condições/caixa | venda indevida | CT-001, CT-002 |
| Multiempresa | vazamento/corrupção | CT-003, CT-004, CT-028 |
| Vendedor/comissão | atribuição incorreta | CT-005, CT-006, CT-032 |
| Cliente/crédito | cadastro ou crédito indevido | CT-007, CT-008, CT-009, CT-023, CT-024 |
| Tabela/preço | perda financeira | CT-006, CT-009, CT-010, CT-011 |
| Produto/estoque | SKU ou saldo incorreto | CT-012 a CT-016, CT-030 |
| Cálculos | divergência monetária | CT-017 a CT-019 |
| Pagamentos | cobrança duplicada/divergente | CT-020, CT-021, CT-029 |
| Suspensão/resiliência | perda/duplicidade | CT-022, CT-031 |
| Depósito | saldo bancário incorreto | CT-033 |
| Crediário | títulos/limite divergentes | CT-023, CT-024 |
| Boleto | duplicidade | CT-025 |
| Fiscal | documento inválido/duplicado | CT-026, CT-027 |
| Consulta/integrações | informação inconsistente | CT-032, CT-034 |

## Lacunas para completar a análise

Solicitar: telas e payloads de finalização; regras de tabela e desconto; configuração de estoque negativo/reserva; planos de pagamento; fluxo de boleto; estados da venda; tela de faturamento; regras de NF-e/NFC-e; política de cancelamento; fórmula de comissão, margem, ticket e ranking; contratos de API; códigos de erro; e SLA das integrações. Essas informações permitem converter as “Regras a confirmar” em resultados esperados definitivos.
