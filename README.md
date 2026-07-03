Correção Mestra
App (web + Android/APK) para correção automática de provas objetivas usando a câmera do celular — também conhecido internamente como "Leitor de Gabaritos". Professores criam provas, geram cartões-resposta com fiduciais/QR code, fotografam as folhas respondidas e o app lê as bolhas marcadas, calcula a nota e permite exportar/importar o resultado para o Nota Mestra (sistema de diário de classe / boletins, um produto separado da mesma equipe).
---
1. Nomes e marcas — não confundir
Este projeto teve dois nomes ao longo do desenvolvimento, e o arquivo ainda carrega marcas dos dois. É importante manter essa distinção clara ao editar:
Nome	O que é
Correção Mestra	O nome atual deste app (antes chamado só de "Nota Mestra" por engano em alguns lugares — corrigido em 03/07/2026). É o app de leitura de gabarito em si.
Nota Mestra	Um app separado, com Firebase próprio, usado para diário de classe / notas por unidade letiva. O Correção Mestra se conecta a ele para importar turmas e exportar notas corrigidas.
Regra prática ao editar o código: qualquer texto que fala sobre o próprio app (splash, título da aba, termos de uso, rodapé do menu "Mais") deve dizer Correção Mestra. Qualquer texto sobre a integração/conexão/importação/exportação (menu "Conectar ao Nota Mestra", "Exportar para Nota Mestra" etc.) deve continuar dizendo Nota Mestra, porque se refere ao outro sistema de verdade.
Onde aparece o nome do app (branding)
`<title>` da aba do navegador
Tela de splash (`#screen-splash`): logotipo "CORREÇÃO MESTRA" + selo de afiliação
Tela de login (`.login-title`): mesmo logotipo compacto
Termos de uso na tela de login
Rodapé do menu "Mais" (`Correção Mestra v7a · Etapa A`)
Selo de afiliação (novo, adicionado em 03/07/2026)
Abaixo do logotipo (splash e login), aparece um texto pequeno: "Um aplicativo Nota Mestra" ou "Um site Nota Mestra", dependendo de onde o app está rodando. A lógica está em `isRunningAsInstalledApp()` / `updateNmAffiliationLabel()`, perto do início do script da splash:
"aplicativo" → quando roda dentro do APK Android (`window.LeituraAndroid` presente), como PWA instalado (`display-mode: standalone`), ou adicionado à tela de início no iOS (`navigator.standalone`).
"site" → qualquer outro caso (aba normal do navegador).
> Observação: para o `display-mode: standalone` funcionar de verdade num PWA instalado (Android/desktop), o app precisaria de um `manifest.json` vinculado no `<head>` — hoje esse manifest **não existe** no HTML, então nesses casos o selo cai no fallback "site" até que o manifest seja adicionado.
---
2. Arquitetura e integrações
Frontend: HTML/CSS/JS puro em um único arquivo (`correcao-mestra-v15.html`), sem framework — várias "telas" (`#screen-*`) alternadas via `showScreen()`.
Backend: Firebase (Compat SDK v9.23.0) — Auth, Firestore.
Dois projetos Firebase distintos são usados ao mesmo tempo:
Projeto Firebase	Uso	Onde está configurado no HTML
`gabarito-nota-mestra`	Projeto principal do Correção Mestra (login, turmas, alunos, provas, resultados)	`firebaseConfig` (~linha 3331)
`nota-mestra`	Projeto secundário, é o app externo Nota Mestra, conectado via app Firebase nomeado `'notaMestra'`	`NM_FIREBASE_CONFIG` (~linha 4626)
App Android: pacote `br.com.leitordegabaritos`, usa o `google-services.json` do projeto `gabarito-nota-mestra`. Web Client ID (OAuth) usado nativamente:
`1052754016695-pu9c81ccffi2lhfsrfpmjc779frr5ra8.apps.googleusercontent.com`
Leitura de gabarito por câmera: usa OpenCV.js (`cv`) no navegador — `detectarFiduciais`, `corrigirPerspectiva`, `lerBolhas` — para achar os marcadores impressos no cartão-resposta, corrigir a perspectiva da foto e ler quais bolhas foram marcadas.
Geração de cartão-resposta: `gerarCartao` → desenha em `<canvas>` (fiduciais, QR code com o ID da prova, grade de bolhas) e exporta como PNG ou PDF.
---
3. Funcionalidades principais
CRUD de turmas, alunos e provas (multi-turma, multi-escola, papéis: professor / coordenador / gestor)
Wizard de criação de prova (gabarito, disciplinas, professores destinatários, provas multidisciplinares)
Geração de cartão-resposta para impressão (com QR code e fiduciais para leitura automática)
Correção via câmera do celular, com tela de conferência manual antes de salvar
Cálculo de nota (simples ou ponderada por disciplina), com anulação de questões
Arquivamento de provas e exclusão de correções
Modo escuro
Integração com o Nota Mestra:
Login cruzado (e-mail/senha ou Google) no projeto Firebase do Nota Mestra
Importação de turmas e unidades letivas do Nota Mestra para casar alunos e enviar notas
Exportação de notas de uma prova em formato JSON compatível com o backup de turma do Nota Mestra
---
4. Incidente resolvido: erro de OAuth ao "Conectar ao Nota Mestra" (Android)
Sintoma: no APK, ao clicar em "Conectar ao Nota Mestra" → "Entrar com Google", aparecia a mensagem de erro pedindo pra usar "E-mail / Senha" em vez disso (código `auth/invalid-credential`).
Causa: a ponte nativa de Google Sign-In do Android (`window.LeituraAndroid.googleSignIn()`) sempre gera um token de identidade usando o Client ID OAuth do projeto gabarito-nota-mestra (o projeto principal do app). Esse token funciona sem problema pro login principal, mas o fluxo "Conectar ao Nota Mestra" tenta autenticar esse mesmo token no Firebase Auth do projeto nota-mestra — um projeto diferente, que por padrão não reconhece Client IDs OAuth de outros projetos e rejeita o token.
Correção aplicada (03/07/2026): no Firebase Console do projeto nota-mestra → Authentication → Sign-in method → Google → seção "Safelist client IDs from external projects", foi adicionado o Client ID:
```
1052754016695-pu9c81ccffi2lhfsrfpmjc779frr5ra8.apps.googleusercontent.com
```
A mudança é salva automaticamente pelo console (não precisa clicar em "Salvar" separadamente). Testado e confirmado funcionando no app instalado.
Se o erro voltar no futuro, o motivo mais provável é a troca do certificado de assinatura (keystore) do APK — isso muda o SHA-1 e pode gerar um novo Client ID, que precisaria ser adicionado de novo nessa mesma safelist. O fallback de "E-mail / Senha" sempre funciona, independente desse problema.
Trecho relevante no código: `loginNMGoogle()` e `_traduzErroAuthNM()` (~linha 4764 e 4794).
---
5. Favicon
Versão antiga: ícone simples de "check" branco sobre círculo azul-navy — parecido demais com o ícone do próprio Nota Mestra (fundo azul com um "V"), o que gerava confusão visual entre os dois apps.
Versão atual: ilustração de folha de cartão-resposta (linhas com bolhas tipo "A B C D", uma marcada por linha), remetendo à função real do app — ler gabaritos. Arquivo separado: `favicon-gabarito.svg` (entregue junto com este README).
---
6. Versão
Build atual identificado no rodapé do app: v7a · Etapa A (comentário no `<head>`: "Etapa 7 — Exportação JSON para Nota Mestra, formato de backup por turma, prova ou atividade").
---
Este README foi gerado a partir de uma sessão de manutenção em 03/07/2026, cobrindo: rename de Nota Mestra → Correção Mestra, selo de afiliação dinâmico, favicon novo, e correção do erro de OAuth do Google na conexão com o Nota Mestra.
