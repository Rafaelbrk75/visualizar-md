# Tarefa 200824: roteiros de teste

O PR 2180 já está na HMG, no commit `ed7502b`. Conferi o código do diálogo e dos textos em português antes de escrever, e **este roteiro diverge do guia da Claudete** em dois pontos:

- **Sem trava de divergência de cartão.** Na pendente de TEF local não existe botão desabilitado nem alerta de "tipo de cartão diferente". O reprocesso conclui com os dados da transação já cobrada.
- **Falha de rede vira toast.** O erro aparece como toast e o diálogo continua aberto. Não há aviso vermelho dentro do diálogo.

## 1. Roteiro de teste manual
**Ambiente:** HMG, ou local apontando para a HMG. Empresa e usuário com permissão de PDV. O TEF local (GetCard ou Stone Mobile) exige uma maquininha física, como a Positivo D400.

**Como criar a pendente de TEF local sem repetir o incidente.** Peça ao backend para inserir uma `TefTransaction` com `Status = PENDING` na venda de teste. Ela precisa estar vinculada a um terminal de gateway local, com NSU, código de autorização, produto de cartão e valor preenchidos. Use um valor diferente do que você vai digitar depois. O reprocesso precisa de log de entrada já gravado; sem ele, o endpoint responde erro e vale como teste do passo 7.

| # | Passo | Resultado esperado |
|---|---|---|
| 1 | No PDV, monte uma venda de R$ 100,00 com a pendente local criada (valor da pendente: R$ 60,00). Escolha pagamento no cartão e toque em **Cobrar**. | O diálogo de transação pendente abre **antes** de a maquininha ser acionada. A maquininha não acorda. |
| 2 | Leia o diálogo. | O texto diz que a maquininha aprovou a cobrança e o lançamento não chegou ao fim. Mostra NSU e data. Aparece **só** o botão **Registrar o pagamento já cobrado**. Não aparecem "Retomar transação" nem "Cancelar e iniciar nova". |
| 3 | Clique em **Registrar o pagamento já cobrado**. | O botão fica bloqueado durante a chamada. O pagamento conclui com NSU, autorização, produto de cartão e valor (R$ 60,00) **da pendente**, não com o que você digitou (R$ 100,00). A maquininha não é acionada. |
| 4 | Confira o pagamento na venda e, no backend, a transação. | A transação não está mais `PENDING`. Existe um único lançamento, sem cobrança duplicada. |
| 5 | Com uma nova pendente, abra o diálogo e feche no **X** ou em Fechar. | O diálogo fecha, a maquininha não é acionada e a pendente continua no backend. |
| 6 | Repita o passo 1 com GetCard e depois com Stone Mobile. | O comportamento é igual nos dois gateways. A GetCard era o caso que pulava a checagem. |
| 7 | Derrube a rede (ou provoque erro no `Reprocess`) e clique em **Registrar o pagamento já cobrado**. | Aparece um toast de erro com a mensagem da API. O diálogo continua aberto, o botão volta a ficar habilitado e a venda **não** é marcada como paga. Com a rede de volta, clicar de novo conclui o pagamento. |
| 8 | **TEF web** (operadora online): crie uma pendente, tente cobrar de novo e abra o diálogo. | Aparecem **Retomar transação** e **Cancelar e iniciar nova**. Não aparece "Registrar o pagamento já cobrado". Os textos são os antigos. |
| 9 | Venda limpa, sem pendente: toque em **Cobrar** no cartão (local e web). | Nenhum diálogo aparece. A maquininha é acionada normalmente, a venda finaliza e o NSU é gravado. A única diferença é uma leitura a mais antes da cobrança. |
| 10 | Troque o idioma para en e es e abra o diálogo da pendente local. | Os três textos novos aparecem traduzidos, sem chave crua como `pages.paymentFlow...`. |
| 11 | Abra o diálogo em tema escuro e na largura de celular. | O botão ocupa a largura toda, o contraste está legível e nada fica cortado. |
| 12 | Pagamento dividido: uma parcela já paga no cartão e uma pendente local na segunda. Reprocesse. | O pagamento correto (o da vez) recebe a autorização. A parcela anterior não é alterada. |

**Regressão de escopo (esperado manter o comportamento antigo):**

| # | Passo | Resultado esperado |
|---|---|---|
| 13 | **Devolução** com cartão e uma pendente local. | O diálogo segue com **Retomar** e **Cancelar e iniciar nova**. Não aparece o botão novo. Isso é intencional e fica para outra tarefa. |
| 14 | **Caixa** (fila de cartão) com pendente local. | Igual ao passo 13. |

**Para o dev:** rode `mirror:check` com o PDVMobile ao lado e confirme que os cinco arquivos espelhados não aparecem como editados no app.

## 2. Roteiro de teste para o QA (linguagem de usuário)

**Tarefa 200824 (BUG).** Quando a maquininha cobra o cartão mas o sistema não termina o registro, a venda trava. Toda nova cobrança é recusada com a mensagem "Já existe uma transação TEF pendente" (ER-PAG-1222), e o operador só tem "Tentar de novo". Agora o sistema avisa **antes** de usar a maquininha e oferece uma saída segura.

**Pré-condições**
- HMG, usuário com acesso ao PDV, empresa com maquininha configurada.
- Uma maquininha de TEF local (GetCard ou Stone Mobile) e, para o cenário 5, uma operadora de TEF web.
- Uma venda com pagamento pendente de TEF local. Peça a alguém do suporte técnico para criá-la (a transação precisa ficar como "pendente" na venda de teste). Anote o valor da pendente e use outro valor ao cobrar.
- Sem essa venda preparada, só os cenários 4 e 6 são possíveis. Registre os demais como não testados.

**Cenários**

1. **Pendente de maquininha local abre a saída antes de cobrar.**
   - Passos: abra a venda, escolha cartão e toque em **Cobrar**.
   - Esperado: aparece uma janela explicando que a maquininha aprovou a cobrança e que o lançamento não terminou. A maquininha não é acionada. A janela tem um único botão, **Registrar o pagamento já cobrado**, e não tem Retomar nem Cancelar.
2. **Registrar o pagamento já cobrado.**
   - Passos: clique no botão do cenário 1.
   - Esperado: a venda recebe o pagamento com o **valor e o NSU da cobrança original**, não com o valor digitado agora. O cliente não é cobrado de novo. A janela fecha e a venda segue normalmente.
3. **Falha ao registrar.**
   - Passos: com a pendente, tire a rede e clique no botão. Depois restabeleça a rede e clique de novo.
   - Esperado: aparece uma mensagem de erro, a janela continua aberta, a venda não é marcada como paga e o botão pode ser clicado outra vez. Com a rede de volta, o registro conclui.
4. **Venda sem pendente (regressão).**
   - Passos: venda nova, cartão, **Cobrar**.
   - Esperado: nenhuma janela extra. A maquininha pede o cartão e a venda finaliza como sempre.
5. **Pendente de operadora web.**
   - Esperado: a janela mostra **Retomar transação** e **Cancelar e iniciar nova**, com os textos de antes. Não aparece "Registrar o pagamento já cobrado".
6. **Fechar a janela sem decidir.**
   - Esperado: nada é cobrado, a maquininha não é acionada e, ao tentar cobrar de novo, a janela volta a aparecer.
7. **Pagamento dividido (cartão + outra forma).**
   - Esperado: o registro vai para o pagamento certo e os pagamentos já feitos não mudam.
8. **Idiomas e aparência.**
   - Esperado: em português, inglês e espanhol, os textos aparecem completos e sem chaves técnicas. Em tema escuro e no celular, o botão fica inteiro e legível.

**Regressão (podem ter sido afetados)**
- Cobrança normal de cartão (crédito, débito e parcelado) em GetCard, Stone e TEF web.
- Devolução com cartão e Caixa (fila de cartão): devem funcionar como antes.
- Retomar e Cancelar numa pendente de TEF web.

**O que NÃO deve acontecer**
- A maquininha ser acionada quando já existe pendente na venda.
- O cliente ser cobrado duas vezes.
- A janela da pendente local mostrar Retomar ou Cancelar.
- O pagamento ser registrado com o valor digitado agora em vez do valor cobrado.
- Aparecer erro "Já existe uma transação TEF pendente" sem nenhuma saída na tela.
- Texto em idioma errado ou chave de tradução crua.

**Devolução e Caixa com pendente local:** continuam com Retomar e Cancelar. É uma limitação conhecida e intencional, e não deve ser reprovada nesta tarefa.

Esse roteiro é a reapresentação para a tarefa já mergeada. Nenhum código mudou depois do PR.