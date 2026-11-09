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
│   ├── theme/                  # Design tokens (cores, espaçamentos, raios)
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

## 5. Registro de Problemas Encontrados e Soluções Adotadas (Troubleshooting)

Durante as etapas de implantação, configuração do backend e homologação do aplicativo, foram identificados e solucionados os seguintes desafios técnicos:

### Problema 1: Erro de Validação de E-mail no Cadastro
- **Descrição do Erro:** Ao registrar um novo usuário, o aplicativo alertava sobre a necessidade de confirmar o e-mail, porém o link de redirecionamento falhava ou o usuário ficava bloqueado sem conseguir logar.
- **Causa Raiz:** Por padrão, o Supabase Auth vem com a opção de confirmação de e-mail ativada (`Confirm email: true`) apontando para URLs locais web (`http://localhost:3000`), as quais não abrem nativamente no app móvel sem esquema de *Deep Linking* pré-configurado.
- **Solução Implementada:** 
  1. No painel do Supabase, acessou-se **Authentication → Providers → Email** e desativou-se a opção **"Confirm email"** para o ambiente de desenvolvimento.
  2. Para o usuário já criado e bloqueado, utilizou-se a ação manual de **"Confirm Email"** diretamente no painel **Authentication → Users**.

### Problema 2: Ausência das Tabelas no Schema Cache
- **Descrição do Erro:** `Could not find the table 'public.rooms' in the schema cache`.
- **Causa Raiz:** O cliente mobile tentou consumir a API REST/Realtime do Supabase, mas o esquema de banco de dados (`supabase/schema.sql`) ainda não havia sido executado no PostgreSQL do projeto remoto, resultando na ausência das tabelas públicas.
- **Solução Implementada:**
  1. Acessou-se a ferramenta **SQL Editor** no painel do Supabase.
  2. Executou-se na íntegra o script DDL `supabase/schema.sql`, criando as tabelas (`profiles`, `rooms`, `room_members`, `messages`), as funções PL/pgSQL, as permissões e as políticas de RLS.
  3. No aviso de segurança do Supabase (*"Potential issue detected: This query includes destructive operations"*), confirmou-se via **Run query**, pois o script contém instruções defensivas seguras (`drop trigger if exists`, `cascade`).

### Problema 3: Violação de Chave Estrangeira ao Criar Salas
- **Descrição do Erro:** `insert or update on table "rooms" violates foreign key constraint "rooms_created_by_fkey"`.
- **Causa Raiz:** O usuário de teste foi criado no `auth.users` antes da execução do script `schema.sql`. Como o *trigger* `handle_new_user` ainda não existia no momento em que a conta foi registrada, a linha correspondente na tabela `public.profiles` não foi gerada. Ao tentar criar uma sala, a coluna `created_by` (que possui chave estrangeira apontando para `profiles.id`) falhou por integridade referencial.
- **Solução Implementada:**
  1. Executou-se uma rotina de sincronização e migração retroativa via SQL Editor:
     ```sql
     insert into public.profiles (id, username, avatar_color)
     select 
       id, 
       coalesce(nullif(trim(raw_user_meta_data ->> 'full_name'), ''), split_part(email, '@', 1)), 
       '#2563EB'
     from auth.users
     on conflict (id) do nothing;
     ```
  2. A partir dessa execução, o perfil existente foi devidamente populado e os próximos usuários passaram a ser criados automaticamente pelo gatilho ativo.

---

## 6. Garantia da Qualidade e Testes Automatizados

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

## 7. Conclusão

O aplicativo **ChatRoom** atendeu a todos os requisitos funcionais e não funcionais propostos para a disciplina de Programação para Dispositivos Móveis. Demonstrou a viabilidade de uma arquitetura limpa e reativa, unindo a velocidade de desenvolvimento do ecossistema Expo/React Native à robustez e escalabilidade do Supabase para serviços em tempo real. Os incidentes observados durante a integração foram devidamente diagnosticados, documentados e sanados, servindo de base para o amadurecimento técnico do projeto.
