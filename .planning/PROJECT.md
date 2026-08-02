# Plataforma de Adoção de Animais

## What This Is

Uma plataforma web onde abrigos e pessoas físicas podem cadastrar animais disponíveis para adoção. Na Fase 1, pessoas interessadas navegam por uma listagem pública, consultam o detalhe do animal e entram em contato diretamente pelo telefone informado pelo doador.

## Core Value

Permitir que um animal disponível para adoção seja publicado e encontrado facilmente, com um canal direto para o interessado falar com seu doador.

## Business Context

- **Customer**: Abrigos, pessoas físicas que doam animais e pessoas interessadas em adotar.
- **Revenue model**: Não definido para o MVP; pagamentos não fazem parte da Fase 1.
- **Success metric**: Um doador consegue publicar um animal e um visitante consegue localizar o anúncio e usar o telefone de contato sem ajuda externa.
- **Strategy notes**: Fase 1 é um MVP funcional e enxuto; fases posteriores poderão incluir filtros, mensagens, administração e otimização de imagens.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Usuário pode criar conta, entrar e manter uma sessão autenticada.
- [ ] Doador pode criar, editar e excluir somente os próprios anúncios de animais, incluindo imagens.
- [ ] Visitante pode navegar pelos animais disponíveis, abrir o detalhe e ver o telefone informado pelo doador.
- [ ] Aplicação pode ser executada localmente e publicada usando serviços com plano gratuito.

### Out of Scope

- Pagamentos — não fazem parte do fluxo de adoção da Fase 1.
- Chat e mensagens internas — o contato inicial será direto por telefone.
- Notificações — não são necessárias para validar o fluxo principal.
- Painel administrativo — será considerado em fase futura.
- Aplicativo mobile — o MVP será uma aplicação web responsiva.
- Busca avançada e filtros — previstos para uma fase posterior.
- Recursos específicos para abrigos — abrigos e pessoas físicas terão o mesmo tipo de conta nesta fase.

## Context

O único fluxo principal do MVP é: um doador cria sua conta, publica um animal com suas informações e fotos; visitantes acessam a grade pública, escolhem um animal, veem seus detalhes e entram em contato pelo telefone do doador. Cada anúncio contém nome, espécie, porte, idade, sexo, cidade, descrição, status de disponibilidade, telefone de contato e imagens. A API precisa ser REST, autenticada por JWT, com autorização por proprietário do anúncio e banco de dados composto inicialmente pelas entidades Usuário e Animal.

## Constraints

- **Scope**: Fase 1 deve permanecer um MVP funcional com um único fluxo principal — orçamento e cronograma são deliberadamente enxutos.
- **Backend**: API REST com cadastro/login, JWT e CRUD de animais — requisito explícito da entrega.
- **Data**: Duas entidades centrais, usuários e animais — manter o modelo inicial simples.
- **Images**: Upload em Cloudinary no plano gratuito ou serviço equivalente — evitar custo de armazenamento no MVP.
- **UX**: Web responsiva, limpa e funcional, sem elaboração visual excessiva — foco no fluxo de adoção.
- **Delivery**: Aplicação publicada, repositório GitHub do cliente, README de instalação e seed com cerca de 10 registros — critérios de aceite.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Contato por telefone exibido no anúncio | Entrega um canal imediato entre interessado e doador sem construir chat. | — Pending |
| Conta única para abrigo e pessoa física | Evita regras e recursos institucionais fora do escopo do MVP. | — Pending |
| Campos padronizados do animal | Nome, espécie, porte, idade, sexo, cidade, descrição, status, telefone e imagens cobrem a publicação inicial. | — Pending |
| Stack a definir na fase de pesquisa | A escolha deve equilibrar velocidade do MVP, implantação gratuita e manutenção futura. | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `$gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `$gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-08-02 after initialization*
