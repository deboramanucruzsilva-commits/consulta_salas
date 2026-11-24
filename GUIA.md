# Guia: rodar, usar o banco, publicar e consertar

Escrito para quem **não** é programadora. Quando aparecer um bloco cinza, é para copiar e colar no terminal (PowerShell no Windows, Terminal no Mac/Linux), dentro da pasta do projeto.

---

## 1. Como o site funciona (em 30 segundos)

```
Navegador  →  Servidor (Node/Express)  →  Banco de dados (MySQL)
 (telas)       (regras, login, admin)      (salas, horários, contas)
```

- **Telas** ficam em `src/` (React).
- **Servidor** fica em `server/`. É ele que decide quem é administrador. O navegador nunca decide isso.
- **Banco** guarda tudo: contas, salas, horários e os textos das páginas. **Se o banco for apagado, os dados somem.** O código do site é só o "molde".

Duas coisas precisam existir para o site funcionar: o servidor rodando **e** um banco MySQL acessível.

---

## 2. Rodar no seu computador

### Instalar uma vez
1. **Docker Desktop**: https://www.docker.com/products/docker-desktop (abra-o e deixe rodando).
2. Só se for mexer no código: **Node.js 22** (https://nodejs.org) e **Git** (https://git-scm.com).

### Opção A: tudo com Docker (mais simples, para usar)

```bash
cp .env.example .env          # Windows PowerShell: copy .env.example .env
```

Abra o arquivo `.env` num editor de texto e troque:
- `APP_SESSION_SECRET`: cole um segredo gerado por `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` (se não tiver Node, invente 40+ letras/números aleatórios).
- Descomente e preencha `ADMIN_EMAIL` e `ADMIN_PASSWORD` (tire o `#` do início das linhas). Essa será a sua conta de administradora.

Depois:

```bash
docker compose up --build -d
```

Abra **http://localhost:3000** e entre com o e-mail/senha de admin. O botão "Gerenciar" leva à área administrativa.

Comandos do dia a dia:

| O que quer | Comando |
|---|---|
| Ver se está rodando | `docker compose ps` |
| Ver mensagens/erros do site | `docker compose logs -f web` |
| Parar (mantém os dados) | `docker compose down` |
| Ligar de novo | `docker compose up -d` |
| Depois de mudar código | `docker compose up --build -d` |
| ⚠️ Apagar TUDO, inclusive dados | `docker compose down -v` (**nunca** use se não quiser perder os dados) |

### Opção B: modo desenvolvimento (para editar o código)

```bash
docker compose up -d db        # só o banco
corepack enable                # se der erro de permissão, abra o PowerShell como administrador
pnpm install --frozen-lockfile
pnpm db:migrate                # cria as tabelas
pnpm dev                       # abra http://localhost:3000
```

Nesse modo, ao salvar um arquivo em `src/` a tela atualiza sozinha. Mudanças em `server/` exigem parar (Ctrl+C) e rodar `pnpm dev` de novo.

---

## 3. Contas e administração

- **Qualquer pessoa** pode se cadastrar em `/cadastro` e passa a ser usuário comum. Escolher "Docente" ou "Secretário(a)" **não** dá poder de admin.
- **Só você decide quem é admin.** Formas de criar um admin:

```bash
pnpm admin:create email@exemplo.com SenhaForte123 "Nome da Pessoa"
```
(com Docker: `docker compose exec web pnpm admin:create email@exemplo.com SenhaForte123`)

Esse comando também **promove** uma conta já existente e **redefine** a senha dela.

- **Esqueceu a senha de alguém** (não existe "esqueci minha senha" por e-mail):

```bash
pnpm user:password email@exemplo.com NovaSenha123
```

### Carregar a grade do semestre
Em **Gerenciar → Importar planilha**:
1. Clique em **Baixar modelo** e abra no Excel/Planilhas.
2. Preencha uma linha por aula: `sala; bloco; tipo; capacidade; dia; inicio; fim; professor; disciplina`.
   - `dia`: segunda, terça… sábado (ou 1 a 6). `inicio`/`fim`: `08:00`.
   - `bloco`, `tipo` e `capacidade` podem ficar vazios.
3. Salve como **CSV UTF-8** (Excel: *Salvar como → CSV UTF-8*). Sem UTF-8, os acentos quebram e o site avisa.
4. Escolha o semestre (ex.: `2026/2`), selecione o arquivo e clique em **Importar**.

Regras: se **qualquer linha** tiver erro, **nada** é importado e o site diz qual linha corrigir. Enviar o mesmo arquivo duas vezes não duplica nada (aulas que já existem são ignoradas). Salas novas são criadas automaticamente.

---

## 4. O banco de dados

### O que tem dentro
| Tabela | Conteúdo |
|---|---|
| `app_users` | contas (e-mail, nome, papel `user`/`admin`, **hash** da senha; a senha em si nunca é guardada) |
| `rooms` | salas (código, bloco, tipo, capacidade) |
| `class_schedules` | aulas (sala, semestre, dia, início, fim, professor, disciplina) |
| `site_content` | textos das 3 páginas públicas |

As tabelas são criadas automaticamente quando o servidor liga (ou com `pnpm db:migrate`). É seguro rodar várias vezes.

### Olhar os dados
Com Docker, abrir o MySQL no terminal:

```bash
docker compose exec db mysql -usalasfac -psalasfac_dev salasfac
```
Aí você digita SQL (termine com `;`; saia com `exit`):

```sql
SELECT code, block, room_type FROM rooms;
SELECT semester, day_of_week, starts_at, ends_at, discipline FROM class_schedules LIMIT 20;
SELECT email, role, created_at FROM app_users;
SELECT COUNT(*) FROM class_schedules WHERE semester = '2026/2';
```

Se preferir janela gráfica: instale **DBeaver** (gratuito), nova conexão MySQL, host `127.0.0.1`, porta `3306`, usuário/senha do seu `.env`.

### Mexer nos dados (com cuidado)
```sql
-- apagar a grade inteira de um semestre (para reimportar do zero)
DELETE FROM class_schedules WHERE semester = '2026/2';
-- tornar alguém admin
UPDATE app_users SET role = 'admin' WHERE email = 'pessoa@exemplo.com';
```
⚠️ `DELETE`/`UPDATE` **sem `WHERE`** atinge a tabela inteira. Faça backup antes (abaixo).

### Backup e restauração
```bash
# BACKUP (gera backup.sql)
docker compose exec db mysqldump -usalasfac -psalasfac_dev salasfac > backup.sql

# RESTAURAR (em banco limpo ou para voltar atrás)
docker compose exec -T db mysql -usalasfac -psalasfac_dev salasfac < backup.sql
```
Guarde o `backup.sql` fora do computador (Drive, pendrive). Ele contém e-mails de usuários: não publique.

---

## 5. Publicar de graça

Você precisa de **dois serviços gratuitos**: um para o banco e um para o site. As regras de planos gratuitos mudam; **confirme os limites nos sites antes** e esteja ciente de que eu não testei essa publicação na prática, só o código localmente.

### 5.1 Colocar o código no GitHub
1. Crie conta em github.com e um repositório **privado** novo.
2. Na pasta do projeto:
```bash
git init
git add .
git commit -m "primeira versão"
git branch -M main
git remote add origin https://github.com/SEU-USUARIO/SEU-REPO.git
git push -u origin main
```
O arquivo `.env` **não** sobe (está no `.gitignore`). Confira no GitHub que ele não aparece.

### 5.2 Banco gratuito: TiDB Cloud Serverless (compatível com MySQL)
1. Crie conta em tidbcloud.com → novo cluster **Serverless** (plano gratuito).
2. Em **Connect**, gere a senha e copie os dados: host, porta (4000), usuário, senha, banco.
3. Monte a URL (se a senha tiver símbolos como `@ : / #`, troque por `%40 %3A %2F %23`):
```
mysql://USUARIO:SENHA@HOST:4000/NOME_DO_BANCO
```
Guarde essa linha: ela é a `DATABASE_URL`. O certificado do TiDB é de autoridade pública, então `DATABASE_SSL=true` funciona sem configurar nada.

> ⚠️ Não testei o esquema no TiDB. Se na primeira subida aparecer erro de `FOREIGN KEY` ou similar, avise com a mensagem exata. Alternativas: outro MySQL gratuito, ou rodar o banco em um computador sempre ligado. Em provedores com certificado próprio use também `DATABASE_SSL_CA`.

### 5.3 Site gratuito: Render
1. Crie conta em render.com, ligue seu GitHub.
2. **New → Blueprint**, escolha o repositório. Ele lê o `render.yaml` e cria o serviço (usa o `Dockerfile`).
3. Preencha as variáveis que o Render pedir:
   - `DATABASE_URL`: a linha do passo 5.2
   - `ADMIN_EMAIL` e `ADMIN_PASSWORD`: sua conta de admin (a senha: 8+ caracteres)
   - `APP_SESSION_SECRET` e `COOKIE_SECURE=true` já vêm configurados.
4. Aguarde o deploy. O endereço será algo como `https://consulta-salas.onrender.com`.
5. Entre com o admin, abra **Gerenciar → Importar planilha** e carregue a grade.

Comportamento do plano gratuito do Render: o site **"dorme"** após alguns minutos sem visitas, e a primeira visita seguinte demora cerca de um minuto para acordar. É normal. Para atualizar o site depois, basta dar `git push`; o Render republica sozinho.

Depois do primeiro login, **remova `ADMIN_PASSWORD`** das variáveis do Render se preferir não deixar a senha lá (a conta continua existindo).

### 5.4 Domínio `.edu.br`
Um endereço da UNIFEI só com autorização da TI da universidade. O endereço do Render funciona sem isso.

---

## 6. Mapa do código: onde mexer em quê

| Quero… | Arquivo |
|---|---|
| Mudar cores | `src/styles.css`, bloco `:root` no topo (`--navy`, `--blue`, `--page`…) |
| Mudar textos das páginas públicas | **Sem código**: Gerenciar → Páginas informativas |
| Mudar textos fixos de botões/rótulos | `src/pages/*.tsx` (procure o texto com Ctrl+F) |
| Mudar logotipo/ícones | arquivos em `public/` (mantenha o mesmo nome) |
| Mudar o painel de consulta | `src/pages/PanelPage.tsx` |
| Mudar a área admin | `src/pages/AdminPage.tsx` |
| Mudar login/cadastro (tela) | `src/pages/AuthPage.tsx` |
| Mudar regras de login/senha (servidor) | `server/auth.ts`, `server/accounts.ts` |
| Mudar rotas da API / regras de salas e horários | `server/index.ts` |
| Mudar a leitura da planilha | `server/csv-import.ts` |
| Mudar as tabelas do banco | `server/migrate.ts` (só **acrescente**; nunca apague colunas com dados) |
| Variáveis de configuração | `.env` (local) / painel do Render (publicado) |

Fluxo seguro para qualquer alteração:
1. `git status` (veja o que está modificado).
2. Faça **uma** mudança pequena.
3. `pnpm build` (verifica erros de digitação no código; precisa terminar sem mensagens vermelhas).
4. Teste no navegador com `pnpm dev`.
5. `git add . && git commit -m "o que mudei"`, depois `git push`.

Commits pequenos e frequentes = você sempre pode voltar atrás.

---

## 7. Se quebrar

### Primeiro: desfazer
```bash
git status                 # lista o que você alterou
git diff                   # mostra as alterações
git restore src/pages/X.tsx   # descarta as alterações de UM arquivo
git restore .              # descarta TODAS as alterações não commitadas
git log --oneline          # lista versões salvas
git revert HASH            # desfaz um commit específico, mantendo o histórico
```
Se o site publicado quebrou depois de um `git push`, faça `git revert` do último commit e dê `git push` de novo.

### Segundo: ler a mensagem de erro
Ela quase sempre diz a causa. Onde ver:
- Docker: `docker compose logs --tail 50 web`
- `pnpm dev`: a própria janela do terminal
- Render: aba **Logs** do serviço
- Navegador: F12 → aba **Console** e **Rede**

### Tabela de problemas
| Sintoma / mensagem | Causa provável | O que fazer |
|---|---|---|
| `APP_SESSION_SECRET ausente ou curto demais` | falta o segredo no `.env` / Render | defina com 32+ caracteres |
| `APP_SESSION_SECRET ainda está com o valor de exemplo` | usou o texto "troque-isto…" em produção | gere um segredo próprio |
| `DATABASE_URL não está disponível` | `.env` inexistente ou sem a linha | copie `.env.example` para `.env` |
| `ECONNREFUSED 127.0.0.1:3306` | o banco não está ligado | `docker compose up -d db` e aguarde ~20 s |
| `Access denied for user` | usuário/senha do banco errados | confira `DATABASE_URL`; senha com símbolos precisa de percent-encoding |
| `O serviço não iniciou porque a base de dados não está pronta` | o servidor não alcança o banco | veja a linha acima dessa no log |
| Erro de certificado/SSL no banco publicado | provedor com certificado próprio | use `DATABASE_SSL_CA`, ou confirme `DATABASE_SSL=true` |
| Site no Render demora ~1 min ou dá 502 ao abrir | plano gratuito acordando | espere e recarregue |
| Entra, mas "cai" logo (sessão não fica) | `COOKIE_SECURE` errado | `false` em `http://localhost`, `true` em `https://` |
| `E-mail ou senha incorretos` para o admin | senha diferente da definida | `pnpm admin:create email NovaSenha123` |
| `Muitas tentativas` | 10 erros de senha em 15 min | espere 15 min (ou reinicie o servidor) |
| Menu "Gerenciar" não aparece | conta não é admin | `pnpm admin:create` com o e-mail dela |
| Painel mostra "demonstração" | a API falhou ao carregar | veja o Console (F12) e os logs do servidor |
| Painel diz "Nenhuma sala cadastrada" | banco sem grade | importe a planilha |
| Importação: "Corrija a planilha" | erro de formato na linha indicada | corrija a linha citada e reenvie |
| Acentos estranhos (�) na importação | CSV não está em UTF-8 | salve como "CSV UTF-8" |
| `port is already allocated` / porta 3000 ocupada | outro programa usa a porta | feche-o, ou troque `PORT` no `.env` e a porta em `compose.yaml` |
| `pnpm: command not found` | falta ativar o pnpm | `corepack enable` (PowerShell como administrador) |
| Tela branca após mudar código | erro de sintaxe no React | olhe o terminal do `pnpm dev` e o Console (F12) |
| `pnpm build` mostra erros vermelhos | erro de tipo/sintaxe | o arquivo e a linha vêm na mensagem; ou `git restore` desse arquivo |
| Perdi dados | volume/banco apagado | restaure o `backup.sql` (seção 4). Sem backup, não há como recuperar |

### Se nada resolver
Peça ajuda a uma pessoa ou a uma IA mandando: (1) o que você fez logo antes, (2) a **mensagem de erro completa** do log, (3) o resultado de `git status`. **Nunca cole** o conteúdo do `.env`, senhas ou `DATABASE_URL` completa. Troque por `***`.

---

## 8. Limites atuais (para você saber)

- **Não há "esqueci minha senha" por e-mail.** Redefinição é feita por você com `pnpm user:password`.
- O limite de tentativas de login fica na memória do servidor (zera ao reiniciar).
- Não há integração com o agendador da UNIFEI; os horários entram pelo admin ou por planilha. Para atualizar um semestre inteiro: apague com SQL (seção 4) e reimporte.
- O site guarda nome, e-mail e matrícula (opcional) dos cadastrados. Para uso institucional, confirme com a universidade as regras de proteção de dados (LGPD).
- O painel consulta a ocupação **prevista pela grade**; não sabe se a aula realmente aconteceu.
