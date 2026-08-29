# A3 – Atividade n8n: análise de imagem com envio real de e-mail

Evolução do fluxo feito em aula: a imagem entra no n8n, é analisada por um modelo
multimodal (Cloudflare Workers AI) e o resultado é **enviado de verdade por e-mail**,
com a **imagem original no corpo da mensagem** + a **análise gerada pelo modelo**.

> **Status: fluxo validado de ponta a ponta em 29/08/2026** — n8n em Docker, análise via
> Cloudflare Workers AI (LLaVA) e envio real pelo SMTP do Gmail, com a imagem original e a
> análise chegando no corpo do e-mail do destinatário.

Arquivos:

- `n8n-workflow-analise-imagem-email.json` – fluxo importável no n8n.
- `docker-compose.yml` + `.env.example` – sobe o n8n (com Postgres) em containers.
- `video_funcionando_n8n-image-workflow.mp4` – vídeo demonstrativo (fluxo no n8n + e-mail recebido).
- `car.jpg`, `car2.png` – imagens usadas nos testes/demonstração.
- `README.md` – este guia.

```
Receber imagem ─▶ Preparar imagem ─▶ Analisar imagem (Workers AI) ─▶ Montar e-mail ─▶ Enviar e-mail (SMTP real)
 (Form Trigger)     (Code)              (HTTP Request)                 (Code)           (Send Email)
```

## O que muda em relação ao fluxo da aula

| Aula (experimento)                                        | Atividade (entrega real)                                                  |
|-----------------------------------------------------------|---------------------------------------------------------------------------|
| **Send Email** com credencial SMTP do **Ethereal** (fake) | Mesmo nó **Send Email**, credencial SMTP real (Gmail, Outlook, etc.)      |
| SSL desmarcado, porta 587 (Ethereal)                      | Gmail: porta 465 + SSL marcado (ou 587 sem SSL → STARTTLS)                |
| HTML só com a análise (`{{ $json.html }}`)                | HTML com **imagem inline** (`<img src="cid:imagem">`) + análise           |
| Destinatário fictício ("e-mail feio" do Ethereal)         | Destinatário real, informado no formulário                                 |
| Conferência no painel do Ethereal                         | Conferência na **caixa de entrada real** (é o que o vídeo deve mostrar)   |

### Caminho mínimo: partir do fluxo feito em aula

Em aula o último nó era o **Send Email** apontando para o **Ethereal** (credencial SMTP com
*SSL desmarcado*, porta 587, destinatário fictício) e o corpo vinha de `{{ $json.html }}` do nó
anterior. Para cumprir a atividade **sem reconstruir nada**, são só 3 mudanças:

1. **Credencial SMTP** → troque host/porta/usuário/senha do Ethereal pelo provedor real.
   Atenção ao SSL: no Ethereal ele ficava **desmarcado** (STARTTLS na 587); no Gmail use
   **porta 465 com SSL marcado** (ou 587 com SSL desmarcado, que o n8n negocia STARTTLS).
2. **To Email** → em vez do "e-mail feio" do Ethereal, um destinatário real
   (fixo ou vindo do formulário, ex.: `{{ $('Receber imagem').item.json['E-mail do destinatário'] }}`).
3. **Imagem no corpo** → o HTML da aula tinha só a análise. Acrescente
   `<img src="cid:imagem">` no HTML e, no nó Send Email, *Options → Attachments* = `imagem`
   (nome da propriedade binária que carrega a imagem; o Code *Montar e-mail* deste repositório já faz isso).

Dicas vindas da própria aula:

- Se seu nó de análise ficou com **On Error → Continue** (ou um IF com saídas sucesso/erro) para
  destravar o teste, desligue isso na versão final: uma falha do modelo não deve gerar e-mail vazio.
- Depois de mexer no HTML, rode o fluxo **completo** (subindo a imagem de novo); o nó de e-mail
  sozinho falha porque depende da saída dos nós anteriores.

### Reaproveitando só os nós de e-mail deste repositório

Se você preferir **continuar o seu fluxo** em vez de importar este inteiro, basta:

1. Manter o seu trigger e o seu nó de análise.
2. Colocar depois deles o nó **Montar e-mail** (Code) e o nó **Enviar e-mail (SMTP real)**
   deste arquivo (copie/cole os dois nós no editor do n8n).
3. No Code "Montar e-mail", ajustar duas linhas:
   - `$('Preparar imagem')` → nome do nó que ainda tem o **binário da imagem**;
   - `resposta?.result?.description` → o campo onde o **seu** modelo devolve o texto.
4. Garantir que a propriedade binária da imagem se chame `imagem`
   (é o nome usado no `cid:` do HTML e na opção *Attachments* do nó de e-mail).

## Rodando tudo em containers (Docker)

Sobe o **n8n** com **Postgres** (persistência de fluxos/credenciais) e, opcionalmente, o **Mailpit**
(SMTP fake para testar o e-mail antes do envio real — faz o papel do Ethereal usado em aula).
O modelo (Cloudflare Workers AI) e o SMTP real continuam sendo serviços externos.

```bash
# 1. Variáveis de ambiente
cp .env.example .env
openssl rand -hex 32          # cole o resultado em N8N_ENCRYPTION_KEY no .env
#    e defina POSTGRES_PASSWORD

# 2. Sobe n8n + Postgres
docker compose up -d
#    -> http://localhost:5678 (na primeira vez, crie o usuário dono)

# 3. (opcional) Importa o fluxo direto pelo CLI do n8n
docker compose --profile import run --rm n8n-import
#    ou importe pela interface: Workflows -> Import from File

# 4. (opcional) SMTP fake para testar antes do envio real
docker compose --profile test up -d mailpit
#    caixa de entrada: http://localhost:8025
#    credencial SMTP no n8n: host "mailpit", porta 1025, sem SSL, sem usuário/senha

# Logs / parar / remover tudo (inclui volumes!)
docker compose logs -f n8n
docker compose down
docker compose down -v
```

Observações:

- `N8N_ENCRYPTION_KEY` criptografa as credenciais salvas; se perdê-la, as credenciais viram inúteis.
  Guarde-a fora do repositório (o `.env` já está no `.gitignore`).
- `WEBHOOK_URL` é a URL usada nos links do Form Trigger. Para demonstrar de outro dispositivo
  (ex.: celular), exponha com um túnel (`ngrok http 5678` / `cloudflared tunnel`) e coloque a URL
  pública no `.env`, depois `docker compose up -d` de novo.
- Binários (imagens) ficam em disco (`N8N_DEFAULT_BINARY_DATA_MODE=filesystem`) e o limite de
  upload está em 16 MB; o fluxo ainda rejeita imagens acima de 2 MB antes de chamar o modelo.
- Para reprodutibilidade, fixe `N8N_VERSION` no `.env` (ex.: `1.90.2`) em vez de `latest`.
- Depois de validar com o Mailpit, troque a credencial SMTP do nó **Enviar e-mail (SMTP real)**
  para o provedor real — é exatamente a "substituição do nó de experimentação" pedida na atividade.

## Pré-requisitos

- n8n (Cloud, self-hosted ≥ 1.x, ou o `docker-compose.yml` desta pasta — requer Docker + Docker Compose v2).
- Conta Cloudflare com **Workers AI** habilitado:
  - **Account ID** (Dashboard → Workers & Pages → lado direito da página).
  - **API Token** com permissão `Workers AI – Read` (My Profile → API Tokens → *Create Token*).
- Uma conta de e-mail com SMTP. Exemplo com Gmail:
  - Ative a verificação em 2 etapas.
  - Gere uma **senha de app** em https://myaccount.google.com/apppasswords.
  - Host `smtp.gmail.com`, porta `465`, SSL/TLS ligado, usuário = seu e-mail, senha = senha de app.

## Passo a passo

### 1. Importar o fluxo
n8n → **Workflows → Import from File** → selecione o `.json`.

### 2. Credencial da Cloudflare (nó *Analisar imagem (Workers AI)*)

O Workers AI já vem habilitado em qualquer conta Cloudflare gratuita (sem cartão, sem domínio,
sem criar Worker) — usamos apenas a API REST. Cota gratuita: 10.000 Neurons/dia, mais do que
suficiente para a atividade.

**Obter Account ID e API Token**

1. Entre em https://dash.cloudflare.com → menu **AI → Workers AI**. O **Account ID** aparece no
   lado direito (também está na URL: `dash.cloudflare.com/<ACCOUNT_ID>/...`).
2. Perfil (canto superior direito) → **My Profile → API Tokens → Create Token** → template
   **"Workers AI"** (ou manual: *Account · Workers AI · Read*). Copie o token — ele só aparece uma vez.
3. Teste no terminal antes de ir para o n8n:

   ```bash
   export CF_ACCOUNT_ID="seu_account_id"; export CF_API_TOKEN="seu_token"
   curl -s https://api.cloudflare.com/client/v4/user/tokens/verify \
     -H "Authorization: Bearer $CF_API_TOKEN"            # esperado: "status":"active"
   curl -s "https://api.cloudflare.com/client/v4/accounts/$CF_ACCOUNT_ID/ai/run/@cf/meta/llama-3.1-8b-instruct" \
     -H "Authorization: Bearer $CF_API_TOKEN" -H "Content-Type: application/json" \
     -d '{"prompt":"Diga olá em uma frase."}'              # esperado: "success":true
   ```

   Sem terminal: **AI → Workers AI → Playground**, escolha o LLaVA e suba uma imagem.

**Configurar no n8n** (o token é usado em um único lugar: a credencial *Header Auth* abaixo,
que o nó envia no cabeçalho `Authorization` de cada chamada)

1. Abra o nó e, na **URL**, troque `SEU_ACCOUNT_ID` pelo seu Account ID (só o ID; o token não vai na URL).
2. **Authentication** → `Generic Credential Type` → **Generic Auth Type** → `Header Auth` →
   **Create new credential**:
   - **Name:** `Authorization`
   - **Value:** `Bearer SEU_API_TOKEN` (a palavra `Bearer`, um espaço e o token inteiro)
3. **Save**. A credencial fica criptografada no n8n (`N8N_ENCRYPTION_KEY`), nunca no JSON do fluxo.
4. Conferência: execute o nó; a saída deve trazer `"success": true` e `result.description`.

O modelo usado é `@cf/llava-hf/llava-1.5-7b-hf` (imagem → texto). Ele recebe a imagem como
array de bytes, por isso o nó *Preparar imagem* converte o binário com
`this.helpers.getBinaryDataBuffer(...)`.

> Quer outro modelo? Troque o final da URL, por exemplo
> `@cf/meta/llama-3.2-11b-vision-instruct`. O Code *Montar e-mail* já aceita tanto
> `result.description` (LLaVA) quanto `result.response` (Llama Vision).

### 3. Credencial SMTP do Gmail (nó *Enviar e-mail (SMTP real)*)

Este é o passo que substitui o Ethereal da aula por um envio real.

**3a. Gerar a senha de app no Google** (o Gmail não aceita a senha normal da conta em SMTP)

1. https://myaccount.google.com/security → **Verificação em duas etapas** → ative, se necessário.
2. https://myaccount.google.com/apppasswords → nome `n8n` → **Criar**.
3. Copie as 16 letras **sem espaços** (`abcd efgh ijkl mnop` → `abcdefghijklmnop`). Só aparece uma vez.

> Contas Google Workspace (empresa/faculdade) podem ter senhas de app bloqueadas pelo administrador;
> nesse caso use um Gmail pessoal.

**3b. Cadastrar no n8n**

Abra o nó → **Credential to connect with → Create new credential (SMTP)**:

| Campo | Valor |
|---|---|
| **User** | `seuemail@gmail.com` |
| **Password** | senha de app (16 letras, sem espaços) |
| **Host** | `smtp.gmail.com` |
| **Port** | `465` |
| **SSL/TLS** | **marcado** |
| **Disable STARTTLS** | desmarcado |

**Save** → deve aparecer *Connection tested successfully*. Alternativa equivalente: porta `587` com
SSL/TLS **desmarcado** (STARTTLS) — a mesma combinação usada com o Ethereal em aula. Não misture
(`465` sem SSL ou `587` com SSL dá `Couldn't connect with these settings`).

**3c. Ajustar o nó**

1. Em **From Email**, use o **mesmo endereço** do campo *User* (ex.: `Análise de Imagem <seuemail@gmail.com>`);
   o Gmail sobrescreve remetentes diferentes da conta autenticada e um *from* divergente cai em spam.
2. **To Email** já vem como `{{ $json.destinatario }}` (e-mail digitado no formulário).
3. Confira as opções já configuradas:
   - **Email Format:** `both` (HTML + texto puro como fallback).
   - **Attachments:** `imagem` → o n8n anexa o binário com `cid = imagem`, que é referenciado no HTML
     por `<img src="cid:imagem">`. Assim a imagem aparece **no corpo** do e-mail.

### 4. Testar
1. Clique em **Test workflow** → abre o formulário (ou use a URL de *Test* do Form Trigger).
2. Envie uma imagem (≤ 2 MB), o e-mail do destinatário e, opcionalmente, uma pergunta.
3. Acompanhe a execução nó a nó; ao final, abra a caixa de entrada do destinatário.
4. Se o primeiro e-mail cair em spam, marque "não é spam" — os seguintes chegam na caixa de entrada.
5. Quando estiver satisfeito, **Activate** o workflow e use a URL de *Production* do formulário.

### 5. Demonstração

O vídeo `video_funcionando_n8n-image-workflow.mp4` mostra a execução completa: formulário com a
imagem (`car.jpg` / `car2.png`), execução dos nós no n8n e o e-mail recebido no destinatário com
a imagem original e a análise do modelo.

## Roteiro sugerido para o vídeo (2–4 min)

1. **Visão geral do fluxo (30 s):** mostrar os 5 nós no canvas e explicar em uma frase cada um;
   destacar que o nó de experimentação (Ethereal) foi substituído pelo *Send Email* com SMTP real.
2. **Entrada (30 s):** abrir o formulário, escolher a imagem, digitar o e-mail do destinatário, enviar.
3. **Execução (30 s):** mostrar a execução ficando verde; abrir a saída do nó *Analisar imagem*
   para exibir o texto retornado pelo modelo.
4. **Resultado (1 min — o mais importante):** abrir a caixa de e-mail do destinatário e mostrar
   a mensagem recebida com **a imagem original no corpo** e **a análise do modelo** logo abaixo.
   Vale abrir também no celular para reforçar que é um envio real.
5. **Encerramento (15 s):** citar melhorias possíveis (vários destinatários, redimensionar imagem,
   trocar o modelo, salvar histórico em planilha).

## Problemas comuns

| Sintoma                                                    | Causa / solução                                                                                                   |
|------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| `401` no nó HTTP                                           | Token colado errado, sem o prefixo `Bearer `, ou expirado.                                                        |
| `403` / erro `10000` no nó HTTP                            | Token sem a permissão *Workers AI · Read* ou criado para outra conta.                                             |
| `404` no nó HTTP                                           | Account ID errado na URL ou nome do modelo com typo (precisa do prefixo `@cf/`).                                  |
| Erro de cota (Neurons)                                     | Estourou os 10.000 Neurons gratuitos do dia; aguarde o dia virar ou use um modelo menor (`uform-gen2`).           |
| `Nenhuma imagem recebida no fluxo`                         | O trigger não entregou binário. No Form Trigger, o campo precisa ser do tipo *File*.                              |
| Imagem aparece só como anexo, não no corpo                 | Versão antiga do nó *Send Email* sem suporte a `cid`. Atualize o n8n ou troque `cid:imagem` por uma URL pública.  |
| E-mail não chega / cai em spam                             | Confira a senha de app, use porta 465 (SSL) e um *From* igual ao usuário autenticado.                             |
| `Couldn't connect with these settings` na credencial SMTP  | Combinação porta/SSL errada: 465 → SSL **marcado**; 587 → SSL **desmarcado**. Salve a credencial após alterar.     |
| `535-5.7.8 Username and Password not accepted`             | Usou a senha normal do Gmail em vez da senha de app, ou copiou a senha de app com espaços.                        |
| `Application-specific password required`                   | Verificação em 2 etapas não está ativa na conta Google.                                                           |
| Chamada muito lenta                                        | Imagem grande. Reduza para ≤ 1 MB (pode inserir um nó *Edit Image → Resize* antes de *Preparar imagem*).           |
| `Cannot read properties of undefined (reading 'first')`    | O nome `Preparar imagem` no Code *Montar e-mail* não bate com o nome real do nó no seu fluxo.                     |

## Segurança

- Token da Cloudflare e senha SMTP ficam **apenas nas credenciais do n8n** — nunca no JSON do fluxo
  nem em texto do vídeo (borre/oculte ao gravar).
- Use **senha de app**, nunca a senha principal da conta de e-mail. Se vazar, revogue em
  https://myaccount.google.com/apppasswords e gere outra.
- O conteúdo do modelo é escapado (`&`, `<`, `>`) antes de entrar no HTML, evitando injeção de marcação.
- O fluxo valida `mimeType` (só imagens) e limita o tamanho a 2 MB antes de chamar a API.
- Se publicar o formulário em produção, considere restringir o acesso (autenticação básica no Form Trigger).
