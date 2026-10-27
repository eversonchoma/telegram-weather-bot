# Bot de Clima no Telegram

Chatbot no Telegram desenvolvido com n8n para informar a temperatura atual de cidades brasileiras. Envie a cidade e a UF como `Cidade,UF` ou `Cidade,UF,BR`; o workflow valida a combinação cidade/estado, consulta a OpenWeather, arredonda a temperatura em Celsius e responde em português brasileiro.

## Visão geral do workflow

1. **Telegram Message Trigger** recebe uma mensagem de texto.
2. **Capture City Input** guarda o texto em `queue` e captura o `chatId` do Telegram.
3. **Parse and Validate City Input** remove espaços excedentes e acentos do nome da cidade, converte o nome para minúsculas e valida o formato de entrada.
4. **Validate Brazil Input** encaminha entradas malformadas para a resposta de erro.
5. **Get City Candidates** consulta a Geocoding API da OpenWeather. **Match Requested State** verifica se o candidato corresponde à cidade solicitada, à UF brasileira e ao Brasil.
6. **Get Current Weather** consulta o endpoint de clima atual usando `q` com o valor normalizado de `queue`, além de Celsius e português brasileiro.
7. **Validate Weather Response** verifica o status HTTP e os campos necessários da resposta.
8. **Format Weather Message** arredonda a temperatura e usa o nome da cidade validada. Os nós do Telegram enviam a resposta de sucesso ou a mensagem de erro exigida pelo desafio.

## Como cidade e UF são validadas

Antes da consulta de clima, o workflow usa a Geocoding API para procurar candidatos e confirmar que um corresponde à cidade, à UF brasileira informada e ao Brasil. A consulta de clima então usa `q` com a variável `queue`, conforme o requisito do desafio. Como a busca por código de estado no parâmetro `q` da Current Weather API é documentada pela OpenWeather para locais dos Estados Unidos, a validação da UF brasileira é feita separadamente pela Geocoding API ([documentação da Current Weather API](https://openweathermap.org/api/current)).

A resposta de clima pode identificar uma localidade central ou próxima no campo `name`. Por isso, a mensagem ao usuário usa o nome validado pelo passo de geocodificação e a temperatura retornada pela consulta de clima.

## Requisitos

- Uma instância do n8n com os nós integrados Telegram, Code, IF, Edit Fields e HTTP Request.
- Um bot do Telegram criado pelo BotFather.
- Uma chave da OpenWeather com acesso à Geocoding API e ao endpoint de clima atual.
- Para receber atualizações do Telegram em uma instância própria do n8n, uma URL HTTPS pública para o webhook. Um Cloudflare Quick Tunnel pode ser usado em testes locais.

## Credenciais e variáveis de ambiente

### OpenWeather

Configure esta variável no ambiente em que o n8n está rodando e reinicie o n8n:

```text
OPENWEATHER_API_KEY=<sua chave da OpenWeather>
```

O workflow lê a chave por meio de `{{$env.OPENWEATHER_API_KEY}}`. Se a instância do n8n bloquear o acesso a variáveis de ambiente em expressões, configure `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` nessa instância.

### Telegram

Crie uma credencial Telegram API no n8n e informe nela o token fornecido pelo BotFather. O desafio também chama esse valor de `TELEGRAM_BOT_TOKEN`, mas este workflow usa a credencial do n8n e não lê um token literal do JSON exportado. Selecione a mesma credencial em **Telegram Message Trigger**, **Send Weather Message** e **Send Error Message**.

Nunca publique chaves reais da API ou tokens do Telegram no repositório, neste README ou no workflow exportado.

## Webhook HTTPS para testes locais com Docker

Ao usar um Cloudflare Tunnel, configure o n8n com o endereço HTTPS atual do túnel em `N8N_WEBHOOK_URL` e defina `N8N_PROXY_HOPS=1`. Use a URL base HTTPS, terminada em `/`, correspondente ao túnel em execução. Mantenha o processo do túnel aberto durante os testes. Um endereço Quick Tunnel pode mudar quando o túnel é reiniciado. Se mudar, atualize a variável no ambiente do n8n e reinicie/recrie o container antes de ativar novamente o Telegram Trigger.

## Importar e executar

1. No n8n, importe `workflow-chatbot-telegram.json` pela opção **Import from File** do menu de workflows.
2. Configure `OPENWEATHER_API_KEY` no ambiente do n8n e crie/selecione a credencial do Telegram descrita acima.
3. Confira se a mesma credencial do Telegram está selecionada no trigger e nos dois nós de envio.
4. Em uma instalação local, confirme que `N8N_WEBHOOK_URL` aponta para o túnel HTTPS em execução e que `N8N_PROXY_HOPS=1` está configurado.
5. Publique/ative o workflow depois de configurar as credenciais e o webhook.
6. Abra uma conversa com o bot e envie uma cidade e UF, por exemplo `Joinville, SC`.

## Exemplos de resposta

Consulta válida (a temperatura varia):

```text
🌤️ A temperatura em Joinville é de 16°C.
```

Cidade desconhecida, entrada inválida ou UF incompatível:

```text
❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

## Testes realizados

Consultas válidas:

- `Joinville, SC`
- `São Paulo, SP`
- `Natal, RN`
- `mallet, pr, br` (entrada em minúsculas)

Casos de erro:

- `CidadeInexistenteZXQ, SP` (cidade inexistente)
- `Joinville, PR` (UF incompatível)
- `Joinville, SC,` (formato malformado)

O Google Gemini não foi incluído porque é opcional no desafio.
