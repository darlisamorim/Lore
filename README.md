# Lore — Biblioteca de Módulos Laravel

> **Escrever o módulo uma vez e, daí em diante, só instalar.**

Documento único do projeto: une o plano (`PLANO.md`), o contrato (`CLAUDE.md`), as notas
do laboratório e a experiência do primeiro módulo (`lore-support`). Os arquivos de antes
foram consolidados aqui em 07/10/2026, quando o repositório foi zerado para recomeçar.

---

## 1. Por que o Lore existe

Um levantamento em `~/Herd` e no `htdocs` do XAMPP encontrou **6 aplicações Laravel +
Filament** reescrevendo os mesmos módulos com nomes diferentes:

| Projeto | Laravel | Filament | Do que trata |
|---|---|---|---|
| Syndicate | 13 | 5 | Prospecção de clientes |
| Valhalla | 13 | 5 | E-commerce modular |
| WebInclude | 12 | 3 | Plataforma da agência |
| Darlisalvesamorim (antigo Flag) | 12 | 3 | Site pessoal |
| Leylis | 13 | 4 | Leilões multi-tenant |
| Rogue | 12 | 3 | SaaS de currículos |

**A duplicação é literal, não só de ideia:**

| Conceito | WebInclude | Leylis | Outros |
|---|---|---|---|
| Suporte | `Ticket`, `SupportChat`, `SupportSetting`… | `SupportTicket`, `SupportChat`… | Chat no Valhalla |
| Alerta de não respondido | `SupportUnansweredAlertJob` | `SendUnansweredTicketAlertJob` + `SendUnansweredChatAlertJob` | — |
| Configurações | `SiteSettings`, `SmtpSetting`, `PaymentSetting`… | `PlatformSetting` | `Setting` (duas versões incompatíveis), `PricingConfig` |

O Lore transforma cada um desses módulos num **pacote Composer independente**
(`darlisamorim/lore-<nome>`, público, MIT). Nos projetos, ele entra com `composer require`,
sem reescrever nada.

Se a peça só faz sentido dentro de um projeto específico, **não é módulo do Lore.**

---

## 2. Regras de ouro (inegociáveis)

1. **Módulo nunca chama módulo.** A comunicação é só por evento, sem exceção.
2. **O módulo não conhece o Lore.** Em runtime é um pacote comum: nada de caminho
   absoluto nem referência ao laboratório.
3. **Só usa do hospedeiro o que declarou** em `module.json` → `requer`.
4. **Teste de ligar/desligar.** `composer remove` não pode quebrar a aplicação. É este
   teste que separa módulo de pedaço de projeto.
5. **Instalável nos projetos que já existem:** Laravel 11–13, PHP 8.2+. Dependência pesada
   (Filament, por exemplo) é **opcional**: só as telas do painel exigem a versão nova.
6. **Peça sem teste não está pronta.** Construir e testar no mesmo fôlego.
7. **Módulo concluído gera documentação:** `registro.md` + `tutorial.md`.
8. **Sem estado global.** Nada de `static` guardando dado entre requisições.
9. **Config publicável, nunca editada no pacote.**

**Lições do primeiro módulo, que viraram regra:**

10. **Pacote não usa a build do hospedeiro.** Classe Tailwind do host não é compilada
    para o pacote. Cada módulo traz a própria folha de estilo, escopada.
11. **O que entra num layout não pode derrubar o layout.** Por exemplo, se as migrations
    não rodaram, o módulo degrada em vez de estourar.
12. **Cor herdada não vai dentro de `rgb()`.** No Filament 5, `--primary-600` já é uma cor
    `oklch(...)` completa. Use `var(--primary-600)` e `color-mix()`.
13. **Paridade de chaves de tradução não prova que a chave existe.** Teste a string resolvida.

---

## 3. Idioma

| Camada | Idioma |
|---|---|
| Código: classes, métodos, colunas, tabelas, config | **inglês** |
| Interface: tudo o que o usuário lê | **`en` + `pt_BR`**, via arquivos de tradução |
| `README.md` do pacote | inglês (mais `README.pt-BR.md` em português) |
| `docs/registro.md`, `docs/tutorial.md`, `module.json` | português |

Nenhuma string fica escrita no meio do código: tudo passa por `__('<nome>::...')`.

---

## 4. Convenções

| Item | Regra | Exemplo |
|---|---|---|
| Repositório | um por módulo | `darlisamorim/lore-support` |
| Pacote Composer | `darlisamorim/lore-<nome>` | `darlisamorim/lore-support` |
| Namespace | `Lore\<Nome>\` → `src/` | `Lore\Support\` |
| Tabelas | prefixo `<nome>_`, **configurável** | `support_tickets` |
| Config | `config/<nome>.php` | `config('support.chat.queue')` |
| Traduções | `recursos/lang/{en,pt_BR}/` | `__('support::chat.queue_position')` |
| Eventos | `Lore\<Nome>\Events\<Fato no passado>` | `ChatAccepted` |

**Camadas** (herdadas do Leylis):

| Peça | Papel | Exemplo |
|---|---|---|
| Action | uma regra de negócio, um método público | `StartChatAction` |
| Service | orquestra várias actions | `ChatService` |
| Observer | efeito colateral | `TicketObserver` |
| Event + Listener | único canal entre módulos | `ChatAccepted` |
| Job | processamento assíncrono | `AlertUnansweredJob` |
| Policy · Form Request · API Resource | autorização · validação · saída | `TicketPolicy` |

**Código:** PSR-12 e `declare(strict_types=1)` em todo arquivo. Migration para toda
mudança de schema, factory para todo model, soft delete em entidades com histórico e índice
em todo campo consultado com frequência.

**Cabeçalho obrigatório** em todo `.php` (herdado do WebInclude e verificado por teste):

```php
<?php

declare(strict_types=1);

/**
 * @source     TicketStatus.php
 * @location   src/Enums/TicketStatus.php
 * @summary    Ticket lifecycle states with labels, colours and queue helpers.
 *
 * @author     Darlis A. Amorim <eu@darlisalvesamorim.com>
 * @see        https://github.com/darlisamorim/lore-support
 * @package    Lore\Support
 */
```

**Telas Filament:** a página de configuração salva sozinha ao sair do campo (auto-save
`onBlur`), e o botão Salvar continua funcionando.

---

## 5. Anatomia de um módulo

```
lore-<nome>/
├── module.json          contrato declarado (fonte da verdade)
├── composer.json
├── README.md · README.pt-BR.md · LICENSE
├── src/                 Provider, Models, Enums, Actions, Services, Events, Jobs, Http, Console
├── config/<nome>.php
├── database/            migrations, factories
├── recursos/            filament/ (só carrega se houver Filament), views/, lang/en, lang/pt_BR, rotas/
├── testes/              LigaDesligaTest, ContratoTest, Feature/
└── docs/                registro.md (escrito durante) · tutorial.md (escrito ao concluir)
```

**`module.json`:**
- `requer.hospedeiro` lista o que o módulo assume do app, por exemplo `"tabela:users"`.
  O que não estiver ali, o módulo não pode usar.
- `requer.modulos` **fica vazio.**
- `expoe` lista tabelas, config, rotas, comandos, eventos e telas. O que não está listado
  é interno e pode mudar.

**Eventos:** o nome fica no passado e o payload leva tipos primitivos ou models do próprio
módulo. Quem dispara nunca espera resposta, e um ouvinte ausente não gera erro.

---

## 6. Testes e documentação

| Teste | O que prova |
|---|---|
| `LigaDesligaTest` | o app sobe com o módulo, migra e desmigra limpo, e nada do hospedeiro depende dele |
| `ContratoTest` | o `module.json` é válido e bate com o código; os cabeçalhos estão presentes |
| Feature | o que o `tutorial.md` promete funciona |

Os testes rodam **dentro do laboratório**, com o módulo instalado por repositório
Composer `path`.

| Documento | Quando | Conteúdo |
|---|---|---|
| `registro.md` | durante a construção | decisões, alternativas descartadas e problemas resolvidos |
| `tutorial.md` | ao concluir | instalação, configuração e operação |
| `README` (en + pt-BR) | ao publicar | cartão de visita no Packagist |

---

## 7. Laboratório

Um **Laravel 13 + Filament 5 pelado**, que serve só de bancada para instalar, rodar e
testar os módulos. Não guarda código de negócio, pode ser recriado do zero a qualquer
momento e não é versionado junto dos módulos.

**Identidade visual (fica no laboratório, nunca no módulo):** dark por padrão, primária
`#B20000`, cinza `Neutral`, Inter no corpo e Montserrat 700 nos títulos, navegação
"timeline" e `->spa()`. O módulo não carrega marca: herda a cor de quem o hospeda.

⚠️ O WebInclude é Filament 3 + Tailwind 3, e o laboratório é Filament 5 + Tailwind 4: os
nomes de classe mudaram. Confira no blade do vendor antes de portar CSS.

---

## 8. Stack

| | Laboratório | Faixa suportada pelo módulo |
|---|---|---|
| PHP | 8.4 | ^8.2 |
| Laravel | 13 | ^11 \| ^12 \| ^13 |
| Filament | 5 | ^5, opcional |
| Testes | Pest 4 | — |
| Banco | SQLite | agnóstico |

---

## 9. Módulos previstos

Ordenados pela repetição real nos projetos.

**Onda 1 — fundação**
| Módulo | Onde já existe |
|---|---|
| Configurações | Syndicate, Flag, WebInclude, Leylis, Rogue |
| Usuários, Perfis e Permissões | todos os 6 |
| Notificações | WebInclude, Leylis |
| Log / Auditoria | WebInclude, Leylis |

**Onda 2 — maior duplicação e maior valor**
| Módulo | Onde já existe |
|---|---|
| Suporte (chamado + chat ao vivo) | WebInclude, Leylis, Valhalla |
| Pagamento (Asaas, Stripe, Pix, assinatura, webhook) | WebInclude, Rogue, Leylis, Valhalla |
| Mídia (upload, compressão, WebP, biblioteca) | WebInclude, Leylis, Valhalla |

**Onda 3 — conteúdo e comercial**
| Módulo | Onde já existe |
|---|---|
| Páginas / CMS | WebInclude, Valhalla |
| Blog | WebInclude |
| SEO + Sitemap / Robots / RSS | WebInclude, Valhalla, Cerberus |
| CRM / E-mail marketing | WebInclude, Valhalla, Syndicate |
| Contratos + Assinatura digital | WebInclude (gov.br) |
| Cupom / Desconto | Valhalla, Rogue |

**Fora do escopo:** multi-tenancy e e-commerce (são arquitetura de base, não se plugam) e
regras de domínio específico (leilão, currículo, prospecção).

---

## 10. O primeiro módulo: `lore-support` (chat ao vivo + chamados)

Consolidou as implementações do Leylis, do WebInclude e do Valhalla num pacote só.
O controller de chat do Leylis tinha **798 linhas**, com o mesmo fluxo escrito três vezes;
no módulo virou **um fluxo de ~150 linhas**.

**O que chegou a funcionar (169 testes verdes):**
- **Chat ao vivo:** fila de espera com posição, visitante sem conta, "está digitando",
  limite por atendente e assumir a conversa ao responder.
- **Chamados:** categorias, prioridades, nota interna que nunca chega ao cliente e relógio
  de primeira resposta.
- **Nos dois:** avaliação de uma vez só e alerta de não respondido (comando agendado).
  Os 10 eventos são o único canal para fora.
- **Widget:** uma linha (`@include`), sem build e sem framework. O servidor decide se
  mostra chat, fila, chamado ou fechado.
- **Painel Filament (opcional):** telas de chat ao vivo, chamados e configurações com
  auto-save.
- **Segurança:** limite de requisições por faixa e token em header (nunca na URL).
  CSRF, XSS, IDOR e mass assignment foram conferidos contra um servidor real.
- **Tema** por tokens (presets `default` e `webinclude`).

**Decisões que valem para os próximos módulos:**
- Prefixo de tabela configurável, para não colidir com o `support_tickets` que já existe.
- Nada é adicionado à tabela `users`: a presença do atendente mora na tabela do módulo.
- Presença e "digitando" são **derivados de timestamp**, nunca booleano, porque booleano
  trava em `true` quando o navegador morre.
- Configuração: o arquivo é o padrão e o painel guarda só o que o admin mudou, com uma
  lista branca de chaves editáveis.
- Tempo real por polling, com broadcast opcional.

**Faltava:** upload de anexos, importação dos dados antigos, broadcasting, painel de
indicadores e a publicação (GitHub + Packagist).

---

## 11. Definição de pronto

- [ ] `module.json` completo e batendo com o código
- [ ] instala no laboratório com um `composer require`
- [ ] `LigaDesligaTest` e `ContratoTest` passam
- [ ] testes de feature cobrem o que o tutorial promete
- [ ] traduções `en` e `pt_BR` completas, sem string solta no código
- [ ] `README.md`, `README.pt-BR.md`, `registro.md` e `tutorial.md` escritos
- [ ] `requer.modulos` vazio
- [ ] repositório próprio publicado e registrado no Packagist

---

## 12. Situação

**Recomeçando do zero (07/10/2026).** O repositório foi zerado e este documento ficou como
a base do projeto. O código anterior (laboratório + `lore-support`) nunca foi enviado ao
GitHub.
