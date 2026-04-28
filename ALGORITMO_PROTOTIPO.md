# Algoritmo — Protótipo da **Plataforma de Evolução de Ideias**

## 1) Visão do algoritmo central

A plataforma deve tratar uma ideia como um **objeto evolutivo** com histórico imutável. O fluxo principal é:

1. Usuário autenticado cria ideia bruta (`titulo`, `descricao`).
2. Sistema gera **versão inicial** da ideia.
3. Usuário estrutura (manual e/ou com sugestão da IA) em:
   - problema
   - solução
   - público-alvo
   - diferenciais
   - riscos
4. Cada edição cria nova versão.
5. Outros usuários enviam feedback (scores + comentário).
6. Sistema recalcula reputação do autor e ranking da ideia.

---

## 2) Estruturas de dados (modelo lógico)

```text
usuarios(id, email, senha_hash, reputacao, criado_em)
ideias(id, titulo, descricao, categoria, user_id, criado_em, atualizado_em)
estruturas(id, ideia_id, problema, solucao, publico, diferencial, riscos, atualizado_em)
versoes(id, ideia_id, numero_versao, conteudo_json, data, autor_id)
feedback(id, ideia_id, usuario_id, clareza, viabilidade, utilidade, comentario, criado_em)
```

### Regras de consistência
- `ideias.user_id` deve existir em `usuarios.id`.
- `estruturas.ideia_id` é `UNIQUE` (1 estrutura ativa por ideia).
- `versoes.numero_versao` cresce sequencialmente por `ideia_id`.
- `feedback` pode ter restrição `(ideia_id, usuario_id)` para evitar spam (ou permitir múltiplos com janela de tempo).

---

## 3) Algoritmo de autenticação (JWT)

### Registro
```pseudocode
function register(email, senha):
    validarFormato(email, senha)
    if existeUsuario(email):
        return ERRO_409

    senha_hash = bcrypt.hash(senha)
    usuario = insert usuarios(email, senha_hash, reputacao=0)

    token = jwt.sign({sub: usuario.id, email: usuario.email}, JWT_SECRET, exp=7d)
    return {usuario, token}
```

### Login
```pseudocode
function login(email, senha):
    usuario = buscarUsuarioPorEmail(email)
    if not usuario: return ERRO_401

    if not bcrypt.compare(senha, usuario.senha_hash):
        return ERRO_401

    token = jwt.sign({sub: usuario.id, email: usuario.email}, JWT_SECRET, exp=7d)
    return {usuario, token}
```

### Middleware de proteção
```pseudocode
function authMiddleware(req, res, next):
    token = extrairBearerToken(req.headers.authorization)
    payload = jwt.verify(token, JWT_SECRET)
    req.user = {id: payload.sub, email: payload.email}
    next()
```

---

## 4) Algoritmo de ideias + versionamento

### Criar ideia (POST /ideias)
```pseudocode
function criarIdeia(userId, titulo, descricao, categoria):
    validarCamposObrigatorios(titulo, descricao)

    ideia = insert ideias(titulo, descricao, categoria, user_id=userId)

    snapshotInicial = {
        titulo: ideia.titulo,
        descricao: ideia.descricao,
        estrutura: null,
        metadata: {origem: "manual", evento: "criacao"}
    }

    insert versoes(
      ideia_id=ideia.id,
      numero_versao=1,
      conteudo_json=snapshotInicial,
      autor_id=userId
    )

    return ideia
```

### Atualizar ideia (PUT /ideias/:id)
```pseudocode
function atualizarIdeia(userId, ideiaId, payload):
    ideia = buscarIdeia(ideiaId)
    if not ideia: return ERRO_404
    if ideia.user_id != userId: return ERRO_403

    iniciarTransacao()

    update ideias set titulo=?, descricao=?, categoria=?, atualizado_em=now()

    estruturaAtual = buscarEstrutura(ideiaId)
    ultimaVersao = buscarUltimoNumeroVersao(ideiaId)

    novoSnapshot = {
      titulo: payload.titulo,
      descricao: payload.descricao,
      estrutura: estruturaAtual,
      metadata: {origem: "manual", evento: "edicao_ideia"}
    }

    insert versoes(
      ideia_id=ideiaId,
      numero_versao=ultimaVersao+1,
      conteudo_json=novoSnapshot,
      autor_id=userId
    )

    commitTransacao()
    return OK
```

### Listagem com paginação e busca (GET /ideias)
```pseudocode
function listarIdeias(query):
    pagina = max(1, query.page || 1)
    limite = clamp(query.limit || 20, 1, 100)
    termo = sanitize(query.search)
    categoria = sanitize(query.categoria)

    where = []
    if termo: where += (titulo ILIKE %termo% OR descricao ILIKE %termo%)
    if categoria: where += (categoria = categoria)

    total = count(ideias where where)
    dados = select ideias where where order by atualizado_em desc
            limit limite offset (pagina-1)*limite

    return {dados, paginacao: {pagina, limite, total}}
```

---

## 5) Algoritmo de estruturação manual + IA

### Estruturação manual (POST /ideias/:id/estrutura)
```pseudocode
function salvarEstrutura(userId, ideiaId, estruturaPayload):
    validarEstrutura(estruturaPayload)
    ideia = buscarIdeia(ideiaId)
    if ideia.user_id != userId: return ERRO_403

    iniciarTransacao()

    upsert estruturas(
      ideia_id=ideiaId,
      problema=payload.problema,
      solucao=payload.solucao,
      publico=payload.publico,
      diferencial=payload.diferencial,
      riscos=payload.riscos,
      atualizado_em=now()
    )

    ideiaAtual = buscarIdeia(ideiaId)
    ultimaVersao = buscarUltimoNumeroVersao(ideiaId)

    snapshot = {
      titulo: ideiaAtual.titulo,
      descricao: ideiaAtual.descricao,
      estrutura: estruturaPayload,
      metadata: {origem: "manual", evento: "estruturacao"}
    }

    insert versoes(... numero_versao=ultimaVersao+1, conteudo_json=snapshot)

    commitTransacao()
    return OK
```

### Endpoint IA (POST /ia/estruturar)
```pseudocode
function estruturarComIA(textoBruto):
    validarTexto(textoBruto)

    prompt = "Transforme a ideia em JSON com campos: problema, solucao, publico, diferencial, riscos"
    resposta = llm.generate(prompt + textoBruto)

    estrutura = parseJsonSeguro(resposta)
    estrutura = aplicarFallbackCamposVazios(estrutura)
    estrutura = validarSchemaEstrutura(estrutura)

    return estrutura
```

> A IA retorna **sugestão**. O usuário confirma/edita antes de persistir.

---

## 6) Algoritmo de histórico de versões (GET /ideias/:id/versoes)

```pseudocode
function obterVersoes(userId, ideiaId):
    ideia = buscarIdeia(ideiaId)
    if not ideia: return ERRO_404

    // Regra: dono sempre pode ver. Colaboradores/comunidade podem ver se política permitir.
    if ideia.user_id != userId and not permissaoLeituraPublica(ideiaId):
        return ERRO_403

    versoes = select * from versoes
              where ideia_id=ideiaId
              order by numero_versao desc

    return versoes
```

---

## 7) Algoritmo de feedback e reputação

### Enviar feedback (POST /ideias/:id/feedback)
```pseudocode
function enviarFeedback(userId, ideiaId, clareza, viabilidade, utilidade, comentario):
    validarRange(clareza, 1..5)
    validarRange(viabilidade, 1..5)
    validarRange(utilidade, 1..5)
    validarTamanhoComentario(comentario, max=1000)

    ideia = buscarIdeia(ideiaId)
    if not ideia: return ERRO_404
    if ideia.user_id == userId: return ERRO_400("autor não avalia própria ideia")

    insert feedback(...)

    recalcularReputacao(ideia.user_id)
    recalcularRankingIdeia(ideiaId)

    return OK
```

### Fórmula sugerida para reputação
```pseudocode
function recalcularReputacao(userId):
    ideiasDoUsuario = buscarIdeiasDoUsuario(userId)

    mediaQualidade = media(
      para cada ideia: media(clareza, viabilidade, utilidade)
    )

    engajamento = log10(1 + totalFeedbackRecebido(userId))
    consistencia = min(totalVersoesUsuario / 50, 1.0)

    reputacao = (mediaQualidade * 12) + (engajamento * 20) + (consistencia * 20)
    update usuarios set reputacao = round(reputacao, 2)
```

### Fórmula sugerida para ranking de ideia
```pseudocode
function recalcularRankingIdeia(ideiaId):
    q = mediaScores(ideiaId)           // 1..5
    n = totalFeedback(ideiaId)
    v = totalVersoes(ideiaId)

    score = (q * 0.6) + (log10(1+n) * 0.25) + (min(v,20)/20 * 0.15)
    cache.set("ideia:ranking:"+ideiaId, score, ttl=10min)
```

---

## 8) API REST (resumo operacional)

### Auth
- `POST /auth/register`
- `POST /auth/login`

### Ideias
- `POST /ideias`
- `GET /ideias?page=1&limit=20&search=&categoria=`
- `GET /ideias/:id`
- `PUT /ideias/:id`

### Estrutura
- `POST /ideias/:id/estrutura`

### Versões
- `GET /ideias/:id/versoes`

### Feedback
- `POST /ideias/:id/feedback`

### IA
- `POST /ia/estruturar`

---

## 9) Estratégia de implementação por fases

### Fase 1 (MVP básico)
- Auth JWT
- CRUD de ideias
- Versionamento automático na criação/edição

### Fase 2
- Estruturação manual
- Histórico de versões completo (comparação simples por JSON)

### Fase 3
- Endpoint IA para sugestão de estrutura
- Feedback com score triplo + comentário

### Fase 4
- Reputação e ranking
- Cache Redis + fila IA (RabbitMQ)
- Observabilidade, rate limit, hardening de segurança

---

## 10) Pseudofluxo ponta a ponta (happy path)

```pseudocode
[Usuário registra/loga] -> recebe JWT
[Usuário cria ideia bruta] -> cria ideia + versão 1
[Usuário chama IA /ia/estruturar] -> recebe sugestão estruturada
[Usuário edita e salva estrutura] -> upsert estrutura + versão 2
[Usuário refina descrição] -> versão 3
[Comunidade envia feedback] -> recalcula reputação/ranking
[Usuário consulta histórico] -> visualiza evolução completa
```

---

## 11) Regras de segurança e qualidade (mínimo obrigatório)

- Validar payload com schema (Zod/Joi).
- Sanitizar entrada textual (evitar XSS em comentários).
- Hash de senha com bcrypt + salt robusto.
- JWT curto + refresh token opcional.
- Rate limit em `/auth/*` e `/ia/*`.
- Logs estruturados (sem dados sensíveis).
- Testes de integração para endpoints críticos.

---

## 12) Stack sugerida para protótipo

- **Frontend:** React + Tailwind + React Query + React Router.
- **Backend:** Node.js + Express + Prisma/Knex.
- **Banco:** PostgreSQL.
- **Auth:** JWT + bcrypt.
- **IA:** serviço dedicado (`/ia`) com timeout e fallback.
- **Infra futura:** Redis, RabbitMQ, Docker, AWS, CDN.

---

## Resultado prático esperado

Com esse algoritmo, o protótipo garante:
- criação simples de ideias;
- estruturação progressiva assistida por IA;
- histórico completo sem perda de contexto;
- ciclo contínuo de melhoria via feedback;
- evolução de qualidade até estado executável.
