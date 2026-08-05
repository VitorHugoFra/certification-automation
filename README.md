# Azure Voucher Watcher

Agente em Python que procura na internet oportunidades de **voucher de exame Microsoft Azure com 100% de desconto**, descarta resultados de 50%, páginas encerradas e treinamentos gratuitos sem exame, e envia apenas novidades por:

- e-mail, usando Gmail ou SMTP genérico;
- Telegram, opcionalmente;
- ambos ao mesmo tempo.

O projeto foi preparado para rodar automaticamente no **GitHub Actions a cada 6 horas**, sem servidor próprio.

## O que o agente faz

1. Executa seis pesquisas em português e inglês pela API de busca Tavily.
2. Lê os resultados e, quando possível, a página da campanha.
3. Exige relação direta com Azure ou códigos como `AZ-305`, `AI-102` e `DP-100`.
4. Exige indicação forte de voucher/exame gratuito ou desconto de 100%.
5. Penaliza ou rejeita:
   - desconto parcial, como 50%;
   - curso ou treinamento gratuito sem exame;
   - simulado, renovação ou segunda tentativa;
   - inscrições encerradas, voucher expirado ou prazo passado.
6. Procura um botão/link de inscrição e o coloca diretamente no alerta.
7. Guarda um histórico em `data/seen.json` para não repetir o mesmo aviso.

> Nenhum filtro automático consegue garantir elegibilidade ou estoque do voucher. O alerta sempre inclui a fonte para você conferir regras, região, empresa parceira, quantidade e prazo.

## Arquivos principais

```text
azure-voucher-watcher/
├── .github/workflows/watch.yml   # agendamento automático
├── data/seen.json                # histórico antirrepetição
├── tests/test_watcher.py         # testes dos filtros
├── azure_voucher_watcher.py      # agente
├── config.json                   # consultas e limites
├── requirements.txt
└── .env.example
```

## Instalação recomendada: GitHub Actions

### 1. Criar a chave de busca

1. Crie uma conta na Tavily.
2. Abra o painel de API Keys.
3. Copie a chave que começa com `tvly-`.

Documentação: https://docs.tavily.com/documentation/quickstart

O plano gratuito informado pela Tavily oferece 1.000 créditos mensais. Esta configuração usa seis buscas a cada seis horas, aproximadamente 720 a 744 créditos por mês. O agente usa busca básica, que a documentação atual contabiliza como um crédito por consulta.

### 2. Preparar o Gmail

Para usar Gmail via SMTP:

1. Ative a verificação em duas etapas na Conta Google.
2. Crie uma **Senha de app** para este agente.
3. Guarde os 16 caracteres; não use sua senha normal do Gmail.

Ajuda oficial: https://support.google.com/mail/answer/185833?hl=pt-BR

Contas corporativas/escolares ou com Proteção Avançada podem não mostrar a opção de senha de app. Nesse caso, configure Telegram ou um SMTP permitido pela sua organização.

### 3. Criar o repositório

1. No GitHub, crie um repositório **privado**.
2. Envie todos os arquivos desta pasta, preservando `.github/workflows/watch.yml`.
3. No repositório, abra:
   `Settings` → `Secrets and variables` → `Actions` → `New repository secret`.

Adicione estes segredos:

| Segredo | Valor |
|---|---|
| `TAVILY_API_KEY` | chave `tvly-...` |
| `GMAIL_USER` | seu endereço Gmail |
| `GMAIL_APP_PASSWORD` | senha de app do Google |
| `ALERT_TO` | endereço que receberá os avisos |

O `ALERT_TO` pode conter vários e-mails separados por vírgula.

### 4. Permitir que o histórico seja salvo

O agente atualiza `data/seen.json` somente após enviar um alerta. No GitHub, confira:

`Settings` → `Actions` → `General` → `Workflow permissions` → **Read and write permissions**.

Em organizações com políticas restritas ou branch protegida, o commit automático pode ser bloqueado. Sem o histórico salvo, uma mesma oportunidade pode ser avisada novamente.

### 5. Fazer o primeiro teste

1. Abra a aba `Actions`.
2. Escolha **Azure Voucher Watcher**.
3. Clique em `Run workflow`.
4. Veja o log da execução.

A partir daí, o arquivo `watch.yml` executa o agente a cada seis horas. O horário do cron é UTC.

## Testar apenas a notificação

No computador ou temporariamente no workflow:

```bash
python azure_voucher_watcher.py --test-notification
```

Esse comando envia uma mensagem falsa de teste e não consome crédito de busca.

## Executar localmente

Requer Python 3.11 ou mais recente. O workflow usa Python 3.13.

```bash
python -m venv .venv
```

Linux/macOS:

```bash
source .venv/bin/activate
pip install -r requirements.txt
export TAVILY_API_KEY="tvly-..."
export GMAIL_USER="seuemail@gmail.com"
export GMAIL_APP_PASSWORD="sua_senha_de_app"
export ALERT_TO="seuemail@gmail.com"
python azure_voucher_watcher.py --dry-run
python azure_voucher_watcher.py
```

Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:TAVILY_API_KEY="tvly-..."
$env:GMAIL_USER="seuemail@gmail.com"
$env:GMAIL_APP_PASSWORD="sua_senha_de_app"
$env:ALERT_TO="seuemail@gmail.com"
python azure_voucher_watcher.py --dry-run
python azure_voucher_watcher.py
```

`--dry-run` pesquisa e mostra os candidatos, mas não envia alerta nem altera o histórico.

## Telegram opcional

1. No Telegram, converse com `@BotFather` e crie um bot.
2. Envie uma mensagem para o bot recém-criado.
3. Descubra o `chat_id` chamando `getUpdates` ou usando uma ferramenta confiável.
4. Adicione os segredos ao repositório:

| Segredo | Valor |
|---|---|
| `TELEGRAM_BOT_TOKEN` | token fornecido pelo BotFather |
| `TELEGRAM_CHAT_ID` | ID da conversa que receberá os avisos |

Documentação oficial: https://core.telegram.org/bots/api

O Telegram pode ser o único canal ou funcionar junto com o e-mail.

## SMTP diferente do Gmail

Em vez das variáveis `GMAIL_*`, use:

```text
SMTP_HOST
SMTP_PORT
SMTP_USER
SMTP_PASSWORD
SMTP_SSL=true|false
SMTP_STARTTLS=true|false
EMAIL_FROM
ALERT_TO
```

Para porta 465, normalmente use `SMTP_SSL=true`. Para porta 587, normalmente use `SMTP_SSL=false` e `SMTP_STARTTLS=true`. Confirme os valores com seu provedor.

## Personalizar as pesquisas

Edite `config.json`:

```json
{
  "queries": [
    "Azure certification \"100% discount\" voucher registration",
    "voucher gratuito certificação Azure inscrição exame"
  ],
  "time_range": "year",
  "min_score": 8,
  "high_confidence_score": 15,
  "alert_cooldown_days": 7,
  "stale_result_days": 450
}
```

Para procurar também campanhas específicas, acrescente consultas como:

```text
Microsoft Certification Week AI-102 voucher
Azure Skills Challenge free exam voucher
site:certweeks.fastlane.net Azure voucher
site:dio.me Azure certificação voucher gratuito
```

Cada consulta consome crédito da API em cada execução. Ajuste a frequência ou o número de consultas para permanecer dentro do seu plano.

## Frequência

O agendamento atual é:

```yaml
- cron: "17 */6 * * *"
```

Para executar duas vezes por dia:

```yaml
- cron: "17 3,15 * * *"
```

Para executar diariamente:

```yaml
- cron: "17 12 * * *"
```

Documentação do GitHub: https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run

## Rodar os testes

```bash
pip install -r requirements-dev.txt
python -m pytest -q
```

Os testes cobrem oferta integral ativa, desconto de 50%, campanha expirada, treinamento sem voucher, normalização de URL e seleção do link de inscrição.

## Segurança

- Nunca grave chaves ou senhas diretamente no código, no `config.json` ou em commits.
- Use GitHub Actions Secrets.
- Prefira repositório privado.
- Revogue a senha de app caso suspeite de vazamento.
- O agente não compra exame, não resgata voucher e não preenche inscrição; ele apenas localiza e alerta.
