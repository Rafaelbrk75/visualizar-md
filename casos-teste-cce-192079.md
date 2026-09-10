# Casos de Teste — Protocolo e Download de XML da Carta de Correção (CC-e)

**Referência:** Tarefa #192079
**Tela:** Documentos Fiscais Emitidos → Listagem de Cartas de Correção
**Escopo:** Exibição do `protocolNumber` e ação de download do XML do evento (`fileName` / `storagePath`), a partir dos dados já retornados por `GET /erp/v1/Sales/Invoice/{id}/CorrectionLetter`.

**Fora de escopo (não deve haver regressão):** fluxo de envio de nova CC-e (geração/homologação do evento 110110).

---

## Convenções

- **Prioridade:** Alta / Média / Baixa
- **Tipo:** Funcional / Regressão / Negativo / Dados / UI
- Pré-condição comum a todos os casos, salvo indicação contrária: usuário autenticado, com permissão de acesso à tela de Documentos Fiscais Emitidos, visualizando uma nota fiscal emitida.

---

## 1. Exibição do protocolo

### CT-01 — Protocolo exibido para CC-e única
**Tipo:** Funcional | **Prioridade:** Alta

**Pré-condições:** Nota com exatamente 1 CC-e homologada, com `protocolNumber` preenchido na API.

**Passos:**
1. Abrir a nota fiscal na tela de Documentos Fiscais Emitidos.
2. Expandir/acessar a listagem de Cartas de Correção da nota.

**Resultado esperado:**
- A carta é listada com sequência, data do evento, texto da correção **e** número do protocolo.
- O protocolo exibido é idêntico (mesmo valor, sem truncamento ou formatação incorreta) ao `protocolNumber` retornado pela API.

---

### CT-02 — Protocolo posicionado junto de sequência e data
**Tipo:** UI | **Prioridade:** Média

**Resultado esperado:**
- O protocolo aparece próximo/agrupado visualmente aos campos sequência e data, conforme critério de aceite ("junto da sequência e da data"), sem exigir ação adicional do usuário (ex.: expandir detalhe) para ser visto.

---

### CT-03 — Múltiplas CC-e: nenhuma troca de protocolo entre sequências
**Tipo:** Funcional / Dados | **Prioridade:** Alta

**Pré-condições:** Nota com 3 ou mais CC-e (sequências 1, 2, 3...), cada uma com `protocolNumber`, `fileName` e `storagePath` distintos e conhecidos previamente (massa de dados controlada).

**Passos:**
1. Abrir a listagem de CC-e da nota.
2. Conferir, linha a linha, se o protocolo exibido corresponde à sequência correta.
3. Repetir a conferência para o arquivo disponibilizado no download de cada linha (ver seção 2).

**Resultado esperado:**
- Cada linha exibe o protocolo correspondente à sua própria sequência.
- Não há inversão, duplicação ou repetição do mesmo protocolo em sequências diferentes.

**Observação:** este é o caso crítico de regressão de mapeamento no `SaleInvoiceCorrectionLetterPresenter` — testar especificamente ordenação/index shift ao converter a lista da API para o modelo de tela.

---

### CT-04 — Protocolo ausente/nulo (cenário defensivo)
**Tipo:** Negativo | **Prioridade:** Média

**Pré-condições:** Registro de CC-e (massa de teste/mock) com `protocolNumber` nulo ou vazio — mesmo que a regra de negócio diga que isso não deve ocorrer em produção (backend só grava com cStat 135).

**Resultado esperado:**
- Tela não quebra, não lança erro em tela nem exceção não tratada.
- Campo de protocolo exibe um estado neutro (ex.: vazio, "-" ou equivalente ao padrão já usado na tela para dados ausentes), sem exibir "null", "undefined" ou similar.

---

## 2. Download do XML do evento

### CT-05 — Download disponível e habilitado quando há storagePath válido
**Tipo:** Funcional | **Prioridade:** Alta

**Pré-condições:** CC-e com `storagePath` e `fileName` válidos e existentes no FileStorage.

**Passos:**
1. Na linha da CC-e, acionar a ação de download do XML do evento.

**Resultado esperado:**
- O download é realizado com sucesso, reaproveitando o mesmo serviço de download já usado para o XML autorizado da nota.
- O arquivo baixado é o XML do evento (CC-e), não o XML da NF-e.

---

### CT-06 — Arquivo baixado mantém o nome real armazenado
**Tipo:** Funcional / Dados | **Prioridade:** Alta

**Passos:**
1. Baixar o XML de uma CC-e cujo `fileName` seja conhecido previamente.
2. Verificar o nome do arquivo salvo pelo navegador/sistema.

**Resultado esperado:**
- O nome do arquivo baixado é exatamente o valor de `fileName` retornado pela API, sem nome genérico (ex.: "download.xml") nem nome derivado incorretamente do `storagePath`.

---

### CT-07 — Múltiplas CC-e: nenhuma troca de arquivo entre sequências
**Tipo:** Funcional / Dados | **Prioridade:** Alta

**Pré-condições:** Mesma massa de dados do CT-03.

**Passos:**
1. Baixar o XML de cada uma das CC-e listadas, uma por vez.
2. Comparar conteúdo/nome de cada arquivo baixado com o `fileName`/`storagePath` esperado para aquela sequência específica.

**Resultado esperado:**
- Cada download corresponde exatamente à sua própria linha/sequência.
- Não há reaproveitamento do último `storagePath` clicado (atenção a bugs de closure/binding em listas renderizadas dinamicamente).

---

### CT-08 — Ação de download ausente/desabilitada quando arquivo não está disponível
**Tipo:** Negativo | **Prioridade:** Alta

**Pré-condições:** CC-e cujo `storagePath` seja nulo, vazio, ou aponte para um arquivo inexistente no FileStorage (simular remoção/indisponibilidade).

**Passos:**
1. Visualizar a linha dessa CC-e na listagem.
2. Tentar acionar o download (se o botão estiver visível).

**Resultado esperado:**
- A ação de download **não é exibida** para essa carta, ou é exibida de forma claramente desabilitada — em nenhum caso deve gerar um botão clicável que resulte em erro, tela quebrada, download vazio ou erro 404/500 exposto ao usuário.
- Critério de aceite: "carta cujo arquivo não esteja disponível no armazenamento não deve oferecer a ação de download quebrada" — validar que essa regra é aplicada por linha, não globalmente (ex.: uma nota com 2 CC-e, uma com arquivo disponível e outra sem, deve tratar cada linha de forma independente).

---

### CT-09 — Erro no serviço de download durante a tentativa (falha em runtime, não em disponibilidade prévia)
**Tipo:** Negativo | **Prioridade:** Média

**Pré-condições:** `storagePath` aparentemente válido, mas o serviço de download de arquivos retorna erro no momento da chamada (ex.: timeout, indisponibilidade momentânea do storage).

**Resultado esperado:**
- Usuário recebe uma mensagem de erro amigável (padrão já usado na tela para falha de download do XML da nota).
- Tela permanece funcional, sem travar a listagem das demais CC-e.

---

## 3. Consistência entre API e apresentação (Presenter)

### CT-10 — Propagação completa dos campos da API até a tela
**Tipo:** Dados | **Prioridade:** Alta

**Passos:**
1. Consultar diretamente a API `GET /erp/v1/Sales/Invoice/{id}/CorrectionLetter` para uma nota de teste e registrar os valores de `id`, `sequenceNumber`, `correction`, `protocolNumber`, `eventAt`, `fileName`, `storagePath`.
2. Abrir a mesma nota na tela.

**Resultado esperado:**
- Sequência, data, texto da correção e protocolo exibidos batem exatamente com os valores da API.
- O download utiliza o `storagePath` correspondente ao mesmo registro (validação indireta, já que `storagePath` não é exibido em texto na tela).

---

### CT-11 — Nota sem nenhuma CC-e
**Tipo:** Funcional | **Prioridade:** Baixa

**Pré-condições:** Nota fiscal sem nenhuma Carta de Correção emitida.

**Resultado esperado:**
- Listagem vazia é exibida normalmente (estado vazio já existente), sem erro relacionado aos novos campos (protocolo/download).

---

### CT-12 — Nota com uma única CC-e (caso mínimo não-vazio)
**Tipo:** Funcional | **Prioridade:** Média

**Resultado esperado:**
- Comportamento idêntico ao CT-01, validando que a lógica de mapeamento não depende de haver múltiplos itens (evitar bug de "funciona só com lista, quebra com item único" ou vice-versa).

---

## 4. Regressão do fluxo já homologado

### CT-13 — Envio de nova CC-e continua funcionando sem alteração
**Tipo:** Regressão | **Prioridade:** Alta

**Passos:**
1. Enviar uma nova Carta de Correção para uma nota (fluxo já existente).
2. Aguardar homologação/registro do evento.

**Resultado esperado:**
- Fluxo de envio funciona exatamente como antes da alteração (mesma UI, mesma validação, mesmo comportamento pós-envio).
- A nova CC-e enviada aparece na listagem já com protocolo e opção de download corretos, assim que disponível pela API.

---

### CT-14 — Download do XML da nota (NF-e) não é afetado
**Tipo:** Regressão | **Prioridade:** Média

**Resultado esperado:**
- A ação de download do XML autorizado da própria nota (funcionalidade pré-existente, cujo serviço está sendo reaproveitado) continua funcionando normalmente, sem conflito com a nova ação de download por CC-e.

---

## 5. Casos de borda adicionais

### CT-15 — Caracteres especiais no texto de correção e no protocolo
**Tipo:** Funcional | **Prioridade:** Baixa

**Pré-condições:** `correction` com caracteres especiais/acentuação; `protocolNumber` com formatação numérica longa (padrão SEFAZ, ex.: 15+ dígitos).

**Resultado esperado:**
- Protocolo é exibido por completo, sem truncamento, quebra de layout ou conversão numérica indevida (ex.: perda de zeros à esquerda, notação científica).

---

### CT-16 — Permissão de acesso
**Tipo:** Funcional / Segurança | **Prioridade:** Média

**Resultado esperado:**
- Usuário sem permissão de acesso à nota/CC-e não consegue acionar o download (mesma regra de permissão já aplicada à listagem/tela hoje), validando que a nova ação não abre uma brecha de acesso direto ao arquivo via `storagePath`.

---

## Resumo de rastreabilidade com os critérios de aceite

| Critério de aceite | Casos de teste |
|---|---|
| Exibir protocolo junto de sequência e data | CT-01, CT-02, CT-04, CT-10, CT-12, CT-15 |
| Download do XML do evento por carta, com nome real | CT-05, CT-06, CT-09 |
| Nota com múltiplas CC-e sem troca de dados | CT-03, CT-07 |
| Sem download quebrado quando arquivo indisponível | CT-08 |
| Nenhuma alteração no fluxo de envio | CT-13, CT-14 |
| Casos de borda / não regressão geral | CT-11, CT-15, CT-16 |
