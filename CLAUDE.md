# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este projeto

Simulação estática (HTML/CSS/JS puro, sem build) do sistema PJE (Processo Judicial Eletrônico) para fins de treinamento. Não há `package.json`, framework ou bundler — cada página é um arquivo `.html` autocontido com `<style>` e `<script>` inline.

## Repositório e infraestrutura

- Código-fonte: https://github.com/marcelmfpm-ai/pje-treino (branch `main`).
- Projeto Firebase: `pje-treino` (Realtime Database em `pje-treino-default-rtdb.firebaseio.com`). Não há `firebase.json`/`.firebaserc` no repo — a integração é feita só via REST direto nas páginas (ver seção "Persistência de dados").

## Comandos

Não há etapa de build/lint/test. Para rodar localmente, basta abrir os arquivos `.html` direto no navegador ou servir a pasta com um servidor estático simples, ex.: `python3 -m http.server`.

## Fluxo de navegação entre páginas

- [index.html](index.html) — tela de login (mock), botão "Entrar" leva a `pagina2.html`.
- [pagina2.html](pagina2.html) — Home pós-login, com navbar (Home/Audiência/Consulta/Cadastro etc). "Consulta de Processo" e "Painel do Advogado - Procurador" → `consulta.html`; "Cadastro > Processo" → `pagina3.html`.
- [pagina3.html](pagina3.html) — Cadastro de Processo, etapa 1 (Seção/Subseção + Classe judicial). Botão "Incluir" → `pagina4.html`.
- [pagina4.html](pagina4.html) — formulário completo de cadastro/instrução do processo (abas, partes, assuntos, IPL, prioridades, anexos). Ao protocolar, gera número CNJ, grava no Firebase e gera um PDF/HTML do processo.
- [consulta.html](consulta.html) — busca um processo já salvo no Firebase pelo número e exibe resultado com link para `detalhe.html`.
- [detalhe.html](detalhe.html) — exibe/edita detalhes de um processo existente (lido via `?numero=` na URL), incluindo upload e listagem de documentos/vídeos anexados.

Todas as páginas internas repetem o mesmo header/navbar (HTML+CSS duplicados); não há componentização — ao alterar a navbar/header, replicar a mudança em cada arquivo (`pagina2.html`, `pagina3.html`, `pagina4.html`, `consulta.html`, `detalhe.html`).

Nos itens de menu ainda não implementados (`href="#"`), a regra `a[href="#"] { cursor: not-allowed !important; }` sinaliza visualmente que o link não faz nada. Gatilhos de dropdown que têm ao menos uma função ativa dentro (ex.: "Painel", "Consulta", "Cadastro" em `pagina2.html`/`pagina3.html`/`pagina4.html`) usam a classe `dropdown-ok` para manter o cursor normal.

Na navbar do `pagina2.html`, ao lado do dropdown "Cadastro", há um atalho (imagem + texto) que abre [Compressor de Vídeo.html](Compressor%20de%20V%C3%ADdeo.html) em nova aba — uma ferramenta auxiliar standalone incluída no repo, sem relação com o fluxo de cadastro de processo. É 100% client-side (mp4box.js + `VideoDecoder`/`VideoEncoder` do navegador + mp4-muxer, sem envio pra servidor); aceita apenas `.mp4`/`.mov`/`.m4v` e não impõe limite de tamanho de arquivo de entrada (diferente dos 50MB de vídeo aplicados no upload via Cloudinary em `pagina4.html`/`detalhe.html`).

## Persistência de dados

- **Firebase Realtime Database** (`https://pje-treino-default-rtdb.firebaseio.com`), projeto `pje-treino`, acessado via REST simples (`fetch` com `GET`/`PUT`, sem SDK do Firebase nem autenticação).
  - Caminho: `processos/{chave}.json`, onde `chave` é o número do processo (formato CNJ) com `.` e `-` substituídos por `_`.
  - `pagina4.html` (`salvarNoFirebase()`) grava o processo completo (número, partes, assuntos, IPLs, sigilo, prioridades, documentos etc).
  - `detalhe.html` (`salvarDocumentosNoFirebase()`) atualiza apenas o sub-caminho `processos/{chave}/documentos.json`.
  - `consulta.html` faz apenas leitura (`GET`) para buscar um processo pelo número informado.
- **Cloudinary** (cloud `dvgk1ocvv`, upload preset `pje_treino`) é usado em `pagina4.html` e `detalhe.html` para upload de anexos (`/auto/upload`), retornando a URL que é armazenada no array `documentos` salvo no Firebase.
  - Limites de tamanho aplicados no client antes do upload: PDFs até 10MB, vídeos (`.mp4/.avi/.mov/.mkv/.webm`) até 50MB.
  - No console do Cloudinary (Settings → Security), a opção "Allow delivery of PDF and ZIP files" precisa estar marcada — por padrão o Cloudinary bloqueia (401) a entrega de PDFs enviados via upload não assinado, mesmo já hospedados.
- **Regras do Firebase Realtime Database** ficam em Console → Realtime Database → Regras. O projeto usa `.read`/`.write: true` sem autenticação (compatível com o acesso REST anônimo do app); regras temporárias com expiração (`"now < <timestamp>"`, geradas por padrão ao criar o banco) bloqueiam todo acesso após a data indicada — se leitura/escrita começarem a falhar com "Permission denied", checar essa aba primeiro.
- Não há backend próprio — toda lógica de geração de número de processo (CNJ fictício), validações e geração de PDF acontece em JS no navegador.

## Convenções observadas

- Idioma da UI e de nomes de funções/variáveis: português (`gravarDocumento`, `protocolarProcesso`, `salvarNoFirebase` etc.) — manter consistência ao adicionar código novo.
- `gerarPDF()` em `pagina4.html` monta um HTML imprimível dinamicamente concatenando strings (`p.push(...)`) — não há template engine.
- Na aba "Anexar Petições/Documentos" de `pagina4.html`, o botão "Gravar" está desabilitado de propósito; quem dispara `gravarDocumento()` é o botão "Assinar Digitalmente" — o rótulo do botão não corresponde ao nome da função, atenção ao mexer nesse trecho.
- Em `pagina4.html` e `detalhe.html`, anexar um arquivo é em duas etapas: escolher o arquivo e clicar em "Adicionar" (`adicionarArquivos()`) para ele entrar na lista de pendentes, e só depois clicar em "Gravar"/"Assinar Digitalmente" (`gravarDocumento()`) para de fato subir pro Cloudinary e salvar no Firebase. `gravarDocumento()` valida que a lista de pendentes não está vazia antes de prosseguir, evitando salvar um documento sem nenhum arquivo anexado.
- `brasao.png` é a imagem do brasão usada em todos os headers.
- Cuidado com nomes de arquivo acentuados (`Compressor de Vídeo.html`, `imagem compressor de video.png`): no macOS local, HFS+/APFS trata as formas Unicode NFC ("í" = `U+00ED`) e NFD ("i" + `U+0301`) do mesmo nome como equivalentes na busca, então tudo parece funcionar mesmo se ficarem dessincronizadas. O GitHub Pages, porém, faz correspondência exata de bytes — um `href` em NFC apontando pra um arquivo salvo em NFD (ou vice-versa) resulta em 404 só no site publicado. Se recriar/renomear esses arquivos (ex. via Finder ou drag-and-drop), confirme a normalização antes de commitar: `python3 -c "import unicodedata,sys; print(unicodedata.normalize('NFC', sys.argv[1]) == sys.argv[1])" "nome do arquivo"`.
