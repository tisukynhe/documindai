# DocuMind AI — Cloudflare + GitHub Pages MVP

Arquitetura: frontend estático no GitHub Pages; API em Cloudflare Workers; D1 para usuários/sessões/histórico; R2 para resultados; Workers AI para análise.

## Estado deste MVP
- Login do administrador com sessão bearer de 8 horas.
- Provisionamento inicial de `admin` por endpoint protegido por segredo, executável uma única vez.
- Senha com PBKDF2-SHA-256 (210.000 iterações) e salt individual.
- Análise real via Workers AI para arquivos TXT e CSV (até 10 arquivos e 2 MB somados).
- Resultado TXT salvo no R2 e apresentado para download no frontend.
- Os parsers PDF, DOCX e XLSX e exportadores PDF/DOCX/XLSX/PPTX ainda precisam ser adicionados. Não são anunciados como funcionais nesta versão.

## 1. Criar recursos Cloudflare
1. Crie um banco D1 chamado `documind-db` e copie o database ID para `worker/wrangler.toml`.
2. Crie um bucket R2 chamado `documind-files`.
3. Faça login pelo Wrangler: `cd worker && npm install && npx wrangler login`.
4. Aplique a migração: `npx wrangler d1 migrations apply documind-db --remote`.
5. Configure o segredo de bootstrap: `npx wrangler secret put SETUP_TOKEN` e informe um valor aleatório longo.
6. Ajuste `ALLOWED_ORIGIN` em `wrangler.toml` para a URL exata do GitHub Pages, por exemplo `https://SEU-USUARIO.github.io`.
7. Publique: `npx wrangler deploy`.
8. Em `frontend/config.js`, substitua a URL de exemplo pela URL real do Worker.

O binding Workers AI é declarado no `wrangler.toml`; habilite Workers AI na conta Cloudflare. O modelo inicial é `@cf/meta/llama-3.1-8b-instruct`; confirme disponibilidade e limites na conta antes de uso contínuo.

## 2. Criar o administrador
Depois de publicar o Worker, faça uma chamada POST para `/api/setup` com o cabeçalho `X-Setup-Token` igual ao segredo configurado e corpo JSON `{"password":"SUA_SENHA_FORTE"}`. O nome do administrador será `admin`. Use senha exclusiva com pelo menos 12 caracteres. Após criar a conta, remova/rotacione `SETUP_TOKEN` e não mantenha credenciais em arquivos do GitHub.

Exemplo usando curl:

```bash
curl -X POST 'https://SEU-WORKER.workers.dev/api/setup' \
  -H 'Content-Type: application/json' \
  -H 'X-Setup-Token: SEU_SEGREDO_DE_BOOTSTRAP' \
  -d '{"password":"SUA_SENHA_FORTE"}'
```

## 3. Publicar frontend no GitHub Pages
1. Crie um repositório GitHub e envie o conteúdo desta pasta.
2. Em Settings → Pages, escolha GitHub Actions como fonte.
3. O workflow `.github/workflows/deploy-pages.yml` publica a pasta `frontend` em cada push na branch `main`.
4. Confira se `ALLOWED_ORIGIN` no Worker corresponde exatamente ao domínio publicado.

## Segurança antes de uso real
- GitHub Pages publica o frontend; não coloque segredos ou arquivos privados no repositório. O código do frontend é público mesmo que a API exija login.
- A senha `0101` não é configurada: é fraca e previsível. Crie uma senha forte no provisionamento inicial.
- Configure rate limiting/WAF no endpoint de login, limites de uso e alertas de consumo.
- Adicione verificação de propriedade por usuário para cada operação de leitura/download de objetos R2 antes de criar endpoints de arquivos.
- Não use dados pessoais, financeiros, médicos ou confidenciais no MVP.
- O CORS não substitui autenticação; mantenha a autorização no Worker.
- Configure política de retenção e exclusão dos arquivos.

## Próximas entregas
1. Parsers PDF/DOCX/XLSX e OCR para PDFs digitalizados.
2. Geração DOCX/XLSX/PPTX/PDF com bibliotecas compatíveis com o runtime escolhido (ou serviço de processamento separado).
3. Tela de histórico e endpoints de listagem/download com autorização por usuário.
4. Fila/Jobs para arquivos grandes, limites, progresso e cancelamento.
5. Rate limiting, auditoria, testes automatizados e política de privacidade.
