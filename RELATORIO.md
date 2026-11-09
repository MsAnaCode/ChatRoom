# Relatório Técnico de Projeto: Aplicativo ChatRoom

**Instituição:** Instituto de Educação Superior de Brasília (IESB)  
**Curso:** Análise e Desenvolvimento de Sistemas / Ciência da Computação  
**Disciplina:** Programação para Dispositivos Móveis  
**Orientador:** Prof. Me. Bruno Assunção Dias  
**Projeto:** ChatRoom — Sistema de Mensageria Móvel em Tempo Real  

---

## 1. Introdução e Objetivos

O projeto **ChatRoom** consiste em uma aplicação móvel completa desenvolvida para suprir a demanda de comunicação instantânea em grupos (salas de bate-papo temáticas). O objetivo principal é demonstrar na prática o uso dos padrões modernos de desenvolvimento mobile, integrando interface responsiva, navegação orientada a rotas declarativas, gerenciamento de estado síncrono e assíncrono e arquitetura baseada em eventos em tempo real (*Realtime Communication*).

### Objetivos Específicos:
- Construir um cliente móvel com **React Native** e **Expo (SDK 58)** com navegação nativa via **Expo Router**.
- Integrar autenticação e autorização robustas com **Supabase Auth** e segurança em nível de banco de dados (**Row Level Security - RLS**).
- Implementar troca de mensagens instantâneas e notificações sem necessidade de *polling*, utilizando os três pilares do **Supabase Realtime**:
  - `postgres_changes`: Escuta e entrega de novos registros de banco de dados diretamente ao app.
  - `Presence`: Rastreamento de estado de usuários ativos/online na sala.
  - `Broadcast`: Notificações efêmeras e de baixa latência (ex.: indicador *"Fulano está digitando..."*).
- Assegurar a robustez do software por meio de uma suíte abrangente de testes automatizados unitários e de integração de banco de dados em memória (**PGlite**).

---

## 2. Tecnologias e Arquitetura

O sistema foi estruturado seguindo os princípios de separação de responsabilidades (SoC) e *Clean Architecture* adaptada ao ecossistema React Native:

### 2.1. Stack Tecnológica
- **Linguagem:** TypeScript (tipagem estática estrita em todas as camadas).
- **Framework Mobile:** React Native `0.88.1` com Expo `58.0.23` (Expo Router `58.0.30`).
- **Backend-as-a-Service (BaaS):** Supabase (PostgreSQL 15+, Supabase Auth e Supabase Realtime).
- **Gerenciamento de Estado de Rede:** TanStack Query (React Query) v5 para cache, refetching e invalidação seletiva de queries.
- **Validação de Formulários:** React Hook Form e Zod (esquemas de validação declarativos).
- **Testes Automatizados:** Jest (Jest-Expo) para regras do frontend e Node.js Test Runner com `@electric-sql/pglite` para testes de banco PostgreSQL isolados em memória.

### 2.2. Estrutura de Pastas e Componentes
```
├── app/                        # Camada de Apresentação e Rotas (Expo Router)
│   ├── (auth)/                 # Grupo de rotas não autenticadas (login, register)
│   ├── (app)/                  # Grupo de rotas protegidas (guardas de autenticação)
│   │   ├── (tabs)/             # Navegação em abas (salas, perfil)
│   │   └── room/               # Telas dinâmicas da sala ([id].tsx) e criação (new.tsx)
│   └── _layout.tsx             # Raiz da aplicação: Context Providers e Stack global
├── src/
│   ├── components/             # Componentes visuais atômicos e reutilizáveis
│   ├── context/                # Contextos globais (AuthContext)
│   ├── hooks/                  # Hooks customizados (useRooms, useMessages, usePresence, useTypingIndicator)
│   ├── services/               # Clientes e chamadas de API do Supabase
│   ├── schemas/                # Schemas de validação Zod
│   ├── theme/                  # Design tokens (cores, tipografia, espaçamentos)
│   ├── types/                  # Definições de tipos TypeScript do domínio
│   └── utils/                  # Funções puras (agrupamento de balões, avatar, formatação)
├── supabase/
│   ├── schema.sql              # Esquema DDL do PostgreSQL, RLS, índices e triggers
│   └── tests/                  # Testes automatizados do banco com PGlite
```

---

## 3. Modelo de Dados e Segurança (PostgreSQL & RLS)

O banco de dados foi modelado para garantir integridade relacional, auditoria automática e segurança em nível de linha (*Row Level Security*), impedindo qualquer acesso não autorizado a dados privados:

### 3.1. Entidades Principais
1. **`public.profiles`**: Informações do usuário vinculadas diretamente a `auth.users(id)`. 
   - Possui *trigger* de banco `handle_new_user()` que cria o perfil com cor e nome automaticamente no momento do cadastro.
2. **`public.rooms`**: Salas temáticas criadas pelos usuários com nome, descrição, emoji e identificador do criador.
3. **`public.room_members`**: Associação muitos-para-muitos entre usuários e salas. Define explicitamente quem tem permissão para visualizar o histórico de mensagens.
4. **`public.messages`**: Mensagens de texto registradas com chave estrangeira para a sala e para o autor.

### 3.2. Políticas de Segurança (Row Level Security - RLS)
Todas as tabelas possuem RLS habilitado:
- **Salas:** Leitura liberada para usuários autenticados; criação restrita com checagem de propriedade (`created_by = auth.uid()`).
- **Membros:** Usuário só pode se cadastrar ou sair de uma sala em seu próprio nome.
- **Mensagens:** Políticas usam a função `public.is_room_member(room_id)` para assegurar que apenas usuários que entraram na sala consigam ler (`SELECT`) ou enviar (`INSERT`) mensagens. A remoção (`DELETE`) só é autorizada para o próprio autor da mensagem.

---

## 4. Funcionalidades Desenvolvidas

1. **Autenticação Desacoplada e Fluida:**
   - Telas de Login e Cadastro com validação em tempo real de formato de e-mail e força de senha com Zod.
   - Guardas de rota automáticos no layout: usuários não autenticados são impedidos de acessar a aplicação interna.

2. **Gerenciamento de Salas:**
   - Listagem de salas públicas com contagem em tempo real de participantes agregados em uma única query otimizada.
   - Criação de novas salas personalizadas com seletor de emojis e regras de validação.
   - Mecanismo explícito de "Entrar na Sala" e "Sair da Sala".

3. **Bate-papo em Tempo Real (Realtime Engine):**
   - **Atualização Instantânea (`postgres_changes`):** Escuta canal WebSocket do Supabase para injetar novas mensagens sem recarregar a tela.
   - **Envio Otimista (*Optimistic Updates*):** A mensagem do usuário é adicionada na tela no mesmo milissegundo do envio e confirmada em segundo plano.
   - **Indicador de Digitação (*Broadcast*):** Notifica os membros presentes quando alguém está digitando, com temporizador de expiração automática (*debounce* de 2 segundos) sem onerar o banco de dados.
   - **Contador de Presença Ativa (*Presence*):** Mostra dinamicamente quantos usuários estão conectados na sala no exato instante.
   - **Gerenciamento de Mensagens:** Suporte a remoção da mensagem pelo autor através de toque longo (*long-press*).

4. **Perfil e Personalização:**
   - Visualização de avatar com iniciais e cor exclusiva gerada no cadastro.
   - Edição de nome de exibição sincronizada com o banco.

---

## 5. Garantia da Qualidade e Testes Automatizados

O projeto conta com **100% de aprovação** na suíte completa de testes automatizados:

```text
Test Suites: 3 passed, 3 total (Jest)
Tests:       20 passed, 20 total
Time:        17.274 s

Suíte de Banco de Dados (PGlite in-memory):
✔ perfis (gatilho automático)
✔ salas e participação (5 cenários de segurança e isolamento)
✔ mensagens (5 cenários de RLS e validações de tamanho)
Tests:       12 passed, 12 total
```

- **Testes Unitários de Frontend:** Validam lógica pura de agrupamento de balões de mensagens contíguas, formatação de horários, cálculo de contraste e cor de avatares, e concatenação de mensagens de digitação.
- **Testes de Integração de Banco de Dados:** Executados em uma instância do motor PostgreSQL nativo rodando dentro do Node.js (`PGlite`), garantindo que as políticas de RLS e triggers funcionem identicamente ao ambiente de produção.

---

## 6. Conclusão

O aplicativo **ChatRoom** atendeu a todos os requisitos funcionais e não funcionais propostos para a disciplina de Programação para Dispositivos Móveis. Demonstrou a viabilidade de uma arquitetura limpa e reativa, unindo a velocidade de desenvolvimento do ecossistema Expo/React Native à robustez e escalabilidade do Supabase para serviços em tempo real.
