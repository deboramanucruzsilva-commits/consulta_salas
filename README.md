# Consulta de Salas — UNIFEI Itabira

Portal para consultar a ocupação **prevista** das salas conforme a grade semestral.
Páginas públicas (início, objetivo, como funciona), painel de consulta com filtros,
cadastro/login por **e-mail e senha** e área administrativa (salas, grade, importação de planilha, textos).

**Leia o [GUIA.md](GUIA.md)**: como rodar, usar o banco, publicar de graça e o que fazer se algo quebrar.

Início rápido (precisa de Docker):

```bash
cp .env.example .env          # edite APP_SESSION_SECRET e, se quiser, ADMIN_EMAIL / ADMIN_PASSWORD
docker compose up --build -d
# abra http://localhost:3000
```

Não há localização de professores em tempo real nem integração com o agendador institucional.
Os dados vêm do que a administração cadastra ou importa por planilha.
