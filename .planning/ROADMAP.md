# Roadmap: Plataforma de Adoção de Animais

## Overview

Este MVP entrega um único percurso verificável: um visitante encontra um animal disponível, abre seus detalhes e usa o telefone consentido para falar com o doador; em seguida, um doador pode criar uma conta, publicar e administrar o próprio anúncio com imagens. A sequência estabelece primeiro o catálogo público e seus limites de privacidade, depois identidade e autorização, publicação, mídia e a entrega publicada. Cobertura de requisitos v1: **16/16 mapeados, sem duplicações ou órfãos**.

O escopo permanece deliberadamente enxuto: não inclui filtros ou busca avançada, chat, pagamentos, notificações, administração, aplicativo mobile ou papéis especiais para abrigos.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Catálogo Público** - Visitantes descobrem animais de demonstração disponíveis, consultam detalhes e iniciam contato por telefone.
- [ ] **Phase 2: Contas e Propriedade** - Doador autentica-se de forma persistente e a aplicação impõe a propriedade de cada anúncio.
- [ ] **Phase 3: Publicação de Anúncios** - Doador cria e administra os dados e a disponibilidade dos próprios animais.
- [ ] **Phase 4: Galeria de Imagens** - Doador inclui e remove fotos autorizadas sem deixar mídia removida acessível.
- [ ] **Phase 5: Entrega Publicada** - O percurso completo funciona localmente e em serviços de plano gratuito.

## Phase Details

### Phase 1: Catálogo Público
**Goal**: Visitantes conseguem descobrir animais disponíveis e entrar em contato com o doador pelo telefone consentido.
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: DISC-01, DISC-02, DISC-03, DELV-02
**Success Criteria** (what must be TRUE):
  1. Um visitante vê uma grade pública responsiva somente com anúncios disponíveis; cada cartão mostra foto, nome, espécie, porte, cidade e acesso aos detalhes, sem expor telefone.
  2. Um visitante abre o detalhe de um animal disponível e vê suas informações, galeria e disponibilidade.
  3. Um visitante vê no detalhe apenas o telefone cujo doador consentiu a divulgação e consegue iniciar uma ligação pelo link `tel:`.
  4. Uma instância recém-preparada exibe cerca de 10 anúncios fictícios utilizáveis para demonstrar a navegação pública.
**Plans**: TBD
**UI hint**: yes

### Phase 2: Contas e Propriedade
**Goal**: Doador consegue acessar sua conta e somente recebe autorização para administrar os próprios anúncios.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, AUTH-03, LIST-04
**Success Criteria** (what must be TRUE):
  1. Uma pessoa cria uma conta com e-mail e senha, entra nela e continua autenticada ao navegar ou atualizar a página.
  2. Um usuário autenticado encerra a própria sessão e deixa de acessar ações protegidas.
  3. Um usuário não consegue editar, excluir nem alterar imagens de um anúncio pertencente a outra conta, mesmo ao tentar acessar a ação diretamente.
**Plans**: TBD
**UI hint**: yes

### Phase 3: Publicação de Anúncios
**Goal**: Doador consegue publicar, atualizar a disponibilidade e remover seus próprios anúncios com os dados necessários para adoção.
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: LIST-01, LIST-02, LIST-03, DISC-04
**Success Criteria** (what must be TRUE):
  1. Um doador autenticado cria um anúncio próprio informando nome, espécie, porte, idade, sexo, cidade, descrição, disponibilidade, telefone e consentimento explícito para exibir o telefone publicamente.
  2. Um doador altera os dados ou a disponibilidade de um anúncio próprio, e a exibição pública passa a refletir a disponibilidade atual.
  3. Um doador exclui um anúncio próprio, que deixa de aparecer na descoberta pública.
  4. Formulários e páginas públicas são utilizáveis em telas pequenas e pelo teclado, com rótulos claros e alternativas textuais para imagens.
**Plans**: TBD
**UI hint**: yes

### Phase 4: Galeria de Imagens
**Goal**: Doador consegue administrar com segurança as imagens dos próprios anúncios.
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: IMAG-01, IMAG-02
**Success Criteria** (what must be TRUE):
  1. Um doador autenticado envia imagens válidas para um anúncio próprio dentro dos limites de quantidade e tamanho definidos pela aplicação, e consegue vê-las na galeria do anúncio.
  2. Ao tentar enviar uma imagem fora dos limites ou em formato não aceito, o doador recebe uma mensagem clara e a imagem não é adicionada ao anúncio.
  3. Um doador remove uma imagem de um anúncio próprio, e a mídia removida deixa de ser servida pelo provedor nem aparece na galeria.
**Plans**: TBD
**UI hint**: yes

### Phase 5: Entrega Publicada
**Goal**: O MVP completo pode ser executado, demonstrado e utilizado em produção sem custo de infraestrutura no fluxo principal.
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: DELV-01, DELV-03
**Success Criteria** (what must be TRUE):
  1. Um desenvolvedor consegue executar localmente a aplicação, a API, o banco de dados e o upload de imagens seguindo o README e o `.env.example`.
  2. Na aplicação publicada em serviços de plano gratuito, uma pessoa consegue concluir cadastro, publicação com imagem, descoberta pública, detalhe e contato por telefone.
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Catálogo Público | 0/TBD | Not started | - |
| 2. Contas e Propriedade | 0/TBD | Not started | - |
| 3. Publicação de Anúncios | 0/TBD | Not started | - |
| 4. Galeria de Imagens | 0/TBD | Not started | - |
| 5. Entrega Publicada | 0/TBD | Not started | - |
