# A3 – Atividade n8n: análise de imagem com envio real de e-mail

Evolução do fluxo feito em aula: a imagem entra no n8n, é analisada por um modelo
multimodal (Cloudflare Workers AI) e o resultado é **enviado de verdade por e-mail**,
com a **imagem original no corpo da mensagem** + a **análise gerada pelo modelo**.

Arquivo do fluxo: `n8n-workflow-analise-imagem-email.json` (importável no n8n).

```
Receber imagem ─▶ Preparar imagem ─▶ Analisar imagem (Workers AI) ─▶ Montar e-mail ─▶ Enviar e-mail (SMTP real)
 (Form Trigger)     (Code)              (HTTP Request)                 (Code)           (Send Email)
```

## O que muda em relação ao fluxo da aula

| Aula (experimento)                              | Atividade (entrega real)                                             |
|-------------------------------------------------|----------------------------------------------------------------------|
| Nó de teste (Ethereal / "no-op" / visualização) | Nó **Send Email** com credencial SMTP real (Gmail, Outlook, etc.)    |
| Só a análise (texto)                            | HTML com **imagem inline** (`<img src="cid:imagem">`) + análise      |
| Destinatário fictício                           | Destinatário real, informado no formulário                            |

Se você preferir **continuar o seu fluxo** em vez de importar este inteiro, basta:

1. Manter o seu trigger e o seu nó de análise.
2. Colocar depois deles o nó **Montar e-mail** (Code) e o nó **Enviar e-mail (SMTP real)**
   deste arquivo (copie/cole os dois nós no editor do n8n).
3. No Code "Montar e-mail", ajustar duas linhas:
   - `$('Preparar imagem')` → nome do nó que ainda tem o **binário da imagem**;
   - `resposta?.result?.description` → o campo onde o **seu** modelo devolve o texto.
4. Garantir que a propriedade binária da imagem se chame `imagem`
   (é o nome usado no `cid:` do HTML e na opção *Attachments* do nó de e-mail).

## Pré-requisitos

- n8n (Cloud ou self-hosted ≥ 1.x).
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
1. Abra o nó e, na **URL**, troque `SEU_ACCOUNT_ID` pelo seu Account ID.
2. Em *Authentication → Generic Credential Type → Header Auth*, crie a credencial:
   - **Name:** `Authorization`
   - **Value:** `Bearer SEU_API_TOKEN`

O modelo usado é `@cf/llava-hf/llava-1.5-7b-hf` (imagem → texto). Ele recebe a imagem como
array de bytes, por isso o nó *Preparar imagem* converte o binário com
`this.helpers.getBinaryDataBuffer(...)`.

> Quer outro modelo? Troque o final da URL, por exemplo
> `@cf/meta/llama-3.2-11b-vision-instruct`. O Code *Montar e-mail* já aceita tanto
> `result.description` (LLaVA) quanto `result.response` (Llama Vision).

### 3. Credencial SMTP (nó *Enviar e-mail (SMTP real)*)
1. Abra o nó → **Credential to connect with → Create new (SMTP)** e preencha host/porta/usuário/senha
   conforme o provedor (ver pré-requisitos).
2. Em **From Email**, troque `SEU_EMAIL@gmail.com` pelo seu endereço (o Gmail ignora um *from*
   diferente da conta autenticada).
3. Confira as opções já configuradas:
   - **Email Format:** `both` (HTML + texto puro como fallback).
   - **Attachments:** `imagem` → o n8n anexa o binário com `cid = imagem`, que é referenciado no HTML
     por `<img src="cid:imagem">`. Assim a imagem aparece **no corpo** do e-mail.

### 4. Testar
1. Clique em **Test workflow** → abre o formulário (ou use a URL de *Test* do Form Trigger).
2. Envie uma imagem (≤ 2 MB), o e-mail do destinatário e, opcionalmente, uma pergunta.
3. Acompanhe a execução nó a nó; ao final, abra a caixa de entrada do destinatário.
4. Quando estiver satisfeito, **Activate** o workflow e use a URL de *Production* do formulário.

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
| `401/403` no nó HTTP                                       | Token sem permissão *Workers AI* ou Account ID errado na URL.                                                     |
| `Nenhuma imagem recebida no fluxo`                         | O trigger não entregou binário. No Form Trigger, o campo precisa ser do tipo *File*.                              |
| Imagem aparece só como anexo, não no corpo                 | Versão antiga do nó *Send Email* sem suporte a `cid`. Atualize o n8n ou troque `cid:imagem` por uma URL pública.  |
| E-mail não chega / cai em spam                             | Confira a senha de app, use porta 465 (SSL) e um *From* igual ao usuário autenticado.                             |
| Chamada muito lenta                                        | Imagem grande. Reduza para ≤ 1 MB (pode inserir um nó *Edit Image → Resize* antes de *Preparar imagem*).           |
| `Cannot read properties of undefined (reading 'first')`    | O nome `Preparar imagem` no Code *Montar e-mail* não bate com o nome real do nó no seu fluxo.                     |

## Segurança

- Token da Cloudflare e senha SMTP ficam **apenas nas credenciais do n8n** — nunca no JSON do fluxo
  nem em texto do vídeo (borre/oculte ao gravar).
- Use **senha de app**, nunca a senha principal da conta de e-mail.
- O conteúdo do modelo é escapado (`&`, `<`, `>`) antes de entrar no HTML, evitando injeção de marcação.
- O fluxo valida `mimeType` (só imagens) e limita o tamanho a 2 MB antes de chamar a API.
- Se publicar o formulário em produção, considere restringir o acesso (autenticação básica no Form Trigger).
