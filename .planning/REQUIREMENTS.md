# Requirements: Plataforma de Adoção de Animais

**Defined:** 2026-08-02
**Core Value:** Permitir que um animal disponível para adoção seja publicado e encontrado facilmente, com um canal direto para o interessado falar com seu doador.

## v1 Requirements

### Authentication

- [ ] **AUTH-01**: Usuário pode criar uma conta com e-mail e senha.
- [ ] **AUTH-02**: Usuário pode entrar com e-mail e senha e manter uma sessão autenticada ao navegar ou atualizar a página.
- [ ] **AUTH-03**: Usuário autenticado pode encerrar a própria sessão.

### Listings

- [ ] **LIST-01**: Usuário autenticado pode criar um anúncio com nome, espécie, porte, idade, sexo, cidade, descrição, status de disponibilidade, telefone de contato e consentimento para exibir o telefone publicamente.
- [ ] **LIST-02**: Usuário autenticado pode alterar os dados e a disponibilidade de um anúncio que lhe pertence.
- [ ] **LIST-03**: Usuário autenticado pode excluir um anúncio que lhe pertence.
- [ ] **LIST-04**: Usuário não consegue editar, excluir ou alterar imagens de anúncio pertencente a outro usuário.

### Images

- [ ] **IMAG-01**: Usuário autenticado pode enviar imagens válidas para um anúncio que lhe pertence, com limite de quantidade e tamanho definido pela aplicação.
- [ ] **IMAG-02**: Usuário autenticado pode remover imagens de um anúncio que lhe pertence, e a mídia correspondente deixa de ser servida pelo provedor.

### Discovery

- [ ] **DISC-01**: Visitante pode ver uma grade pública responsiva contendo apenas anúncios disponíveis, com foto, nome, espécie, porte, cidade e link para detalhes.
- [ ] **DISC-02**: Visitante pode abrir uma página pública de detalhe com as informações, galeria e disponibilidade de um animal disponível.
- [ ] **DISC-03**: Visitante pode ver no detalhe o telefone consentido pelo doador e acioná-lo por um link `tel:`; números não aparecem nos cartões da listagem.
- [ ] **DISC-04**: Formulários e páginas públicas são utilizáveis em telas pequenas e por teclado, incluindo rótulos e alternativas textuais para imagens.

### Delivery

- [ ] **DELV-01**: Desenvolvedor pode executar frontend, API, banco de dados e upload de imagens localmente seguindo um README e um `.env.example`.
- [ ] **DELV-02**: Desenvolvedor pode carregar de modo idempotente cerca de 10 anúncios fictícios e utilizáveis para demonstração.
- [ ] **DELV-03**: Usuário pode completar o fluxo cadastro, publicação com imagem, descoberta e contato por telefone na aplicação publicada em serviços de plano gratuito.

## v2 Requirements

### Discovery

- **FILT-01**: Visitante pode filtrar e ordenar anúncios por atributos como espécie, porte, idade, sexo e cidade.

### Communication

- **CHAT-01**: Interessado e doador podem trocar mensagens dentro da plataforma.
- **NOTF-01**: Usuários recebem notificações relacionadas a interesse, mensagens ou alterações de anúncios.

### Operations

- **ADMN-01**: Administrador pode revisar usuários e anúncios em um painel administrativo.
- **IMOP-01**: Plataforma otimiza imagens, incluindo transformações e entrega adaptada por dispositivo.

## Out of Scope

| Feature | Reason |
|---------|--------|
| Pagamentos | Não apoiam o fluxo principal de descoberta e contato da Fase 1. |
| Chat ou formulário de interesse | Telefone público consentido resolve o contato inicial sem construir comunicação interna. |
| Notificações | Dependem de fluxos de interação que não pertencem ao MVP. |
| Painel administrativo e moderação | Aumentam significativamente a complexidade operacional e ficam para fase futura. |
| Busca avançada, filtros, ordenação e mapas | A listagem pública simples valida primeiro a descoberta de animais. |
| Perfis, papéis ou recursos exclusivos para abrigos | Abrigos e pessoas físicas usarão uma conta única nesta fase. |
| Aplicativo mobile nativo | A aplicação web responsiva é suficiente para a Fase 1. |
| Login social, redefinição de senha e 2FA | E-mail e senha atendem a autenticação inicial; recursos adicionais não são necessários para o fluxo principal. |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| AUTH-01 | Unmapped | Pending |
| AUTH-02 | Unmapped | Pending |
| AUTH-03 | Unmapped | Pending |
| LIST-01 | Unmapped | Pending |
| LIST-02 | Unmapped | Pending |
| LIST-03 | Unmapped | Pending |
| LIST-04 | Unmapped | Pending |
| IMAG-01 | Unmapped | Pending |
| IMAG-02 | Unmapped | Pending |
| DISC-01 | Unmapped | Pending |
| DISC-02 | Unmapped | Pending |
| DISC-03 | Unmapped | Pending |
| DISC-04 | Unmapped | Pending |
| DELV-01 | Unmapped | Pending |
| DELV-02 | Unmapped | Pending |
| DELV-03 | Unmapped | Pending |

**Coverage:**
- v1 requirements: 16 total
- Mapped to phases: 0
- Unmapped: 16 ⚠️

---
*Requirements defined: 2026-08-02*
*Last updated: 2026-08-02 after initial definition*
