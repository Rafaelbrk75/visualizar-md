# Requisicoes TEF GP

Documentacao dos endpoints para verificar status do terminal TEF e iniciar transacoes.

## Variaveis

| Variavel | Descricao | Exemplo |
| --- | --- | --- |
| `paymentHubUrl` | URL base do Payment Hub | `http://localhost:8000` |

## Verificar Status do Terminal TEF

Verifica se o terminal TEF esta ativo/disponivel.

**Metodo:** `POST`  
**Endpoint:** `http://localhost:8000/api/tefgp-status`

### Resposta de sucesso

**HTTP:** `201 OK`

```json
{
  "comando": "ATV",
  "idReq": 0,
  "valor": 0.0,
  "status": "",
  "adquirente": "",
  "tipoTransacao": "",
  "nsu": "",
  "codigoAutorizacao": "",
  "parcelas": 0,
  "dataHora": "1899-12-30T00:00:00.000Z",
  "controle": "",
  "bandeira": "",
  "codigoAutorizacaoGp": "",
  "txId": "",
  "end2end": "",
  "cnpjAdiquirente": "",
  "codigoBandeira": "",
  "codigoCredenciadora": "",
  "codigoPdv": "",
  "operacao": "",
  "tipoCartao": "",
  "tipoFinanc": "",
  "codigoRedeAdiquirente": "",
  "bin": "",
  "erro": "",
  "regFinalizador": "0",
  "impressoCliente": [],
  "impressoLoja": []
}
```

## Transacionar Credito Parcelado

Inicia uma transacao de credito parcelado.

**Metodo:** `POST`  
**Endpoint:** `{{paymentHubUrl}}/api/tefgp-req`  
**Content-Type:** `application/json`

### Exemplo

```json
{
  "comando": "CRT",
  "identificacao": 5000,
  "valorTotal": 2000,
  "empresaAutomacao": "VISUAL SOFWARE LTDA",
  "dataHoraFiscal": "2026-06-06T14:16:14.9995946Z",
  "dataHoraComprovante": "2026-06-06T14:16:14.9995946Z",
  "nomeAutomacao": "ASTRO ERP",
  "tipoTransacao": "11",
  "parcelas": "4"
}
```

## Transacionar Debito

Inicia uma transacao de debito.

**Metodo:** `POST`  
**Endpoint:** `{{paymentHubUrl}}/api/tefgp-req`  
**Content-Type:** `application/json`

### Exemplo

```json
{
  "comando": "CRT",
  "identificacao": 5000,
  "valorTotal": 2000,
  "empresaAutomacao": "VISUAL SOFWARE LTDA",
  "dataHoraFiscal": "2026-06-06T14:16:14.9995946Z",
  "dataHoraComprovante": "2026-06-06T14:16:14.9995946Z",
  "nomeAutomacao": "ASTRO ERP",
  "tipoTransacao": "20"
}
```

## Transacionar Pix

Inicia uma transacao Pix.

**Metodo:** `POST`  
**Endpoint:** `{{paymentHubUrl}}/api/tefgp-req`  
**Content-Type:** `application/json`

### Exemplo

```json
{
  "comando": "CRT",
  "identificacao": 5000,
  "valorTotal": 2000,
  "empresaAutomacao": "VISUAL SOFWARE LTDA",
  "dataHoraFiscal": "2026-06-06T14:16:14.9995946Z",
  "dataHoraComprovante": "2026-06-06T14:16:14.9995946Z",
  "nomeAutomacao": "ASTRO ERP",
  "tipoTransacao": "51"
}
```

## Transacionar Credito a Vista

Inicia uma transacao de credito a vista.

**Metodo:** `POST`  
**Endpoint:** `{{paymentHubUrl}}/api/tefgp-req`  
**Content-Type:** `application/json`

### Exemplo

```json
{
  "comando": "CRT",
  "identificacao": 5000,
  "valorTotal": 2000,
  "empresaAutomacao": "VISUAL SOFWARE LTDA",
  "dataHoraFiscal": "2026-06-06T14:16:14.9995946Z",
  "dataHoraComprovante": "2026-06-06T14:16:14.9995946Z",
  "nomeAutomacao": "ASTRO ERP",
  "tipoTransacao": "10"
}
```

## Campos

| Campo | Descricao |
| --- | --- |
| `comando` | Comando TEF. Por padrao, usar sempre `CRT` para cartao. |
| `identificacao` | Identificador do TEF. Pode ser o ID da venda. |
| `valorTotal` | Valor contado a partir de centavos. Exemplo: `2000` equivale a `R$ 20,00`. |
| `empresaAutomacao` | Empresa da automacao. Valor padrao: `VISUAL SOFWARE LTDA`. |
| `dataHoraFiscal` | Data/hora fiscal da transacao. Usar a data atual. |
| `dataHoraComprovante` | Data/hora do comprovante. Usar a mesma data de `dataHoraFiscal`. |
| `nomeAutomacao` | Nome da automacao. Valor padrao: `ASTRO ERP`. |
| `tipoTransacao` | Tipo da transacao: `11` credito parcelado, `20` debito, `51` Pix, `10` credito a vista. |
| `parcelas` | Numero de parcelas. Elegivel apenas para credito parcelado. |

## Tipos de Transacao

| Codigo | Tipo |
| --- | --- |
| `11` | Credito parcelado |
| `20` | Debito |
| `51` | Pix |
| `10` | Credito a vista |
