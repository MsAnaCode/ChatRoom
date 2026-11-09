# ChatRoom

App de mensagens em tempo real: entre em salas públicas, converse e veja quem está online e quem está digitando — tudo atualizando sozinho, sem precisar puxar a tela para atualizar.

## Funcionalidades

- Cadastro e login (e-mail e senha)
- **Perfil automático**: nome e cor do avatar são criados sozinhos no cadastro via Trigger no PostgreSQL
- Lista de salas públicas, com contagem de participantes em uma só query
- Criar uma sala (com emoji e descrição) e entrar/sair de salas
- **Mensagens em tempo real** (Supabase Realtime): aparecem em todos os aparelhos abertos na mesma sala via `postgres_changes`
- Envio **otimista**: a mensagem aparece na hora e é confirmada depois
- **Presença**: contador de quem está online na sala agora (Supabase Presence)
- **"Fulano está digitando..."**: aviso efêmero que não grava nada no banco (Supabase Broadcast)
- Apagar a própria mensagem com long-press; editar o nome exibido no Perfil

## Tecnologias

React Native · Expo · Expo Router · TypeScript · Supabase (Auth, Postgres, RLS, **Realtime**: `postgres_changes`, Presence e Broadcast) · TanStack Query · React Hook Form · Zod · Jest · PGlite (testes de banco em memória)

## Como executar

1. Instale as dependências: `npm install`
2. Crie um projeto no [Supabase](https://supabase.com) e execute o script `supabase/schema.sql` no SQL Editor.
3. Em **Database → Replication**, confirme se a tabela `messages` está com o Realtime ativado.
4. Em **Authentication → Providers → Email**, desative a opção "Confirm email" (para permitir cadastro e login imediato sem envio de link por e-mail).
5. Copie `.env.example` para `.env` e preencha sua URL e anon/public key do Supabase:
   ```env
   EXPO_PUBLIC_SUPABASE_URL=https://SEU-PROJETO.supabase.co
   EXPO_PUBLIC_SUPABASE_KEY=SUA-CHAVE-ANON
   ```
6. Inicie o app: `npx expo start -c` e escaneie o QR Code no app **Expo Go** no celular.

## Testes

| Comando | O que testa |
| --- | --- |
| `npm test` | Todos os testes: app (Jest) e banco (PGlite) |
| `npm run test:app` | Agrupamento de mensagens, avatar e indicador de digitação (Jest) |
| `npm run test:db` | RLS, gatilho de perfil, isolamento de mensagens e salas (PGlite) |

O teste do banco roda com PGlite (PostgreSQL real executado dentro do Node.js), validando todas as regras de segurança sem requisições externas ou necessidade de Docker.

## Arquitetura

```
app/         rotas e layouts (Expo Router)
src/
  components/  interface: balão de mensagem, avatar, cartão de sala...
  context/     sessão e autenticação do usuário
  hooks/       salas, mensagens (Realtime), presença, digitação
  services/    integração com Supabase
  theme/       tokens de estilo (cores, espaçamentos, raios)
  types/       modelos de domínio
  utils/       agrupamento de mensagens, avatar, indicador, erros
supabase/
  schema.sql   banco, RLS, trigger de perfil e função is_room_member
  tests/       testes automatizados do banco via PGlite
```

## Autor

Projeto acadêmico desenvolvido para a disciplina de Programação para Dispositivos Móveis - IESB.
Orientação: Prof. Me. Bruno Assunção Dias.
