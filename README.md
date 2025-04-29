# ReactJS :: Aula 04 

**Prof. Ricardo Maroquio** 

> **Fluxo completo de conta**  ― registro, login, recuperação/troca de senha e edição de perfil (com foto);

> **Rotas protegidas**  que redirecionam quem não está logado;

> **Painel de Administração**  em que **apenas usuários‐admin**  gerenciam outros perfis.

---

## 0. Pré-requisitos revistos 

| Já feito | Conceito | 
| --- | --- | 
| Aula 01 | Componentização, JSX, Vite+Bootstrap | 
| Aula 02 | useState, eventos, React-Router | 
| Aula 03 | CRUD de produtos (Supabase + React-Query) | 

Para hoje você **já deve ter:** 

```bash
npm i @supabase/supabase-js        # aula 03
npm i @tanstack/react-query        # aula 03
npm i react-router-dom@6
```

---

## 1. Entendendo o Supabase Auth 

| Termo | Descrição Rápida | 
| --- | --- | 
| provedores | Email + senha, OAuth (Google, GitHub etc.), magic-link, telefone | 
| Session | Objeto que contém o access_token JWT e o usuário logado | 
| Claims | Dados extras inseridos no JWT (is_admin, por exemplo) | 
| Row-Level Security | Políticas SQL que conferem se o usuário (claim uid) pode ler/escrever linhas | 

---

## 2. Preparação no painel do Supabase 

### 2.1 Ativar o e-mail + links de reset 
 
2. **Authentication ▸ Settings ▸ Email** 
 
  - *Confirm email:* **ON**
 
  - *Password recovery:* **ON**
 
  - *Site URL:* `http://localhost:5173`
 
  - *Redirect URLs:* `http://localhost:5173/update-password`
 
4. **Storage**  → **Create bucket** 
 
  - Nome: `avatars`
 
  - Público? **Não**  (usaremos *signed URLs*).

2.2 Tabela **profiles** 

```sql
create table profiles (
  id          uuid primary key references auth.users on delete cascade,
  full_name   text,
  avatar_url  text,
  is_admin    boolean default false,
  updated_at  timestamp with time zone default now()
);
```

#### Políticas RLS 

```sql
-- Leitura: o usuário só vê o próprio perfil
create policy "Profiles are viewable by owner"
  on profiles for select
  using ( auth.uid() = id );

-- Update: o usuário edita o próprio perfil
create policy "Users can edit their own profile"
  on profiles for update
  using ( auth.uid() = id );

-- Admins podem ver / editar tudo
create policy "Admins can manage all profiles"
  on profiles
  for all
  using ( exists (
      select 1 from profiles p
      where p.id = auth.uid() and p.is_admin
  ));
```

> ⚠️ **Habilite RLS**  depois de criar as policies.

---

## 3. Configurando o cliente de autenticação 

3.1 `src/services/supabase.js` (atualize se já existir)

```js
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

export const supabase = createClient(supabaseUrl, supabaseKey, {
  auth: {
    persistSession: true,
    detectSessionInUrl: true,   // permite capturar o token do link de reset
  },
});
```

Crie/atualize `.env`:

```ini
VITE_SUPABASE_URL=...
VITE_SUPABASE_ANON_KEY=...
```

### 3.2 Providenciando estado global de usuário 

`src/contexts/AuthContext.jsx`

```jsx
import { createContext, useContext, useEffect, useState } from 'react';
import { supabase } from '../services/supabase';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [session, setSession] = useState(supabase.auth.getSession()?.data?.session || null);
  const user = session?.user || null;
  const isAdmin = user?.user_metadata?.is_admin; // veremos na seção 6

  useEffect(() => {
    const { data: listener } = supabase.auth.onAuthStateChange((_event, sess) => {
      setSession(sess);
    });
    return () => listener.subscription.unsubscribe();
  }, []);

  const value = { session, user, isAdmin };
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};

export const useAuth = () => useContext(AuthContext);
```

Envolva o `<App />` em `main.jsx`:

```jsx
createRoot(document.getElementById('root')).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <App />
      </AuthProvider>
    </QueryClientProvider>
  </StrictMode>
);
```

---

## 4. Rotas protegidas 

`src/components/ProtectedRoute.jsx`

```jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../contexts/AuthContext';

export const ProtectedRoute = ({ children }) => {
  const { user } = useAuth();
  return user ? children : <Navigate to="/login" replace />;
};
```

`src/components/AdminRoute.jsx`

```jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../contexts/AuthContext';

export const AdminRoute = ({ children }) => {
  const { user, isAdmin } = useAuth();
  if (!user) return <Navigate to="/login" replace />;
  return isAdmin ? children : <Navigate to="/" replace />;
};
```

---

## 5. Páginas de conta 

> Coloque todas dentro de `src/pages/auth/`

5.1 **LoginPage.jsx** 

```jsx
import { useState } from 'react';
import { supabase } from '../../services/supabase';
import { useNavigate, Link } from 'react-router-dom';

export default function LoginPage() {
  const nav = useNavigate();
  const [form, set] = useState({ email: '', password: '' });
  const [error, setError] = useState('');

  const handleSubmit = async e => {
    e.preventDefault();
    const { error } = await supabase.auth.signInWithPassword(form);
    if (error) return setError(error.message);
    nav('/');       // logged!
  };

  return (
    <div className="row justify-content-center">
      <div className="col-md-4">
        <h2>Entrar</h2>
        <form onSubmit={handleSubmit}>
          <input className="form-control mb-2"
                 placeholder="E-mail"
                 onChange={e => set({ ...form, email: e.target.value })}
          />
          <input className="form-control mb-2"
                 type="password"
                 placeholder="Senha"
                 onChange={e => set({ ...form, password: e.target.value })}
          />
          {error && <div className="alert alert-danger">{error}</div>}
          <button className="btn btn-primary w-100">Entrar</button>
        </form>
        <div className="text-center mt-2">
          <Link to="/register">Criar conta</Link> |{' '}
          <Link to="/forgot-password">Esqueci a senha</Link>
        </div>
      </div>
    </div>
  );
}
```

5.2 **RegisterPage.jsx** 

```jsx
import { useState } from 'react';
import { supabase } from '../../services/supabase';
import { useNavigate } from 'react-router-dom';

export default function RegisterPage() {
  const nav = useNavigate();
  const [form, set] = useState({ email: '', password: '', full_name: '' });
  const [error, setError] = useState('');

  const handleSubmit = async e => {
    e.preventDefault();
    const { data, error } = await supabase.auth.signUp({
      email: form.email,
      password: form.password,
      options: {
        data: { full_name: form.full_name, is_admin: false },
      },
    });
    if (error) return setError(error.message);

    // Cria registro em profiles
    await supabase.from('profiles').insert({
      id: data.user.id,
      full_name: form.full_name,
    });

    alert('Verifique seu e-mail para confirmar a conta.');
    nav('/login');
  };

  /* … inputs muito parecidos ao login … */
}
```

5.3 **ForgotPasswordPage.jsx** 

```jsx
const handle = async () => {
  const { error } = await supabase.auth.resetPasswordForEmail(email, {
    redirectTo: 'http://localhost:5173/update-password',
  });
  …
};
```

5.4 **UpdatePasswordPage.jsx** 
Quando o usuário clicar no link recebido, o Supabase redireciona com um `access_token` na URL; `supabase-js` captura e já deixa o usuário logado. Então basta exibir um formulário:

```jsx
const { user } = useAuth();
const onSubmit = async () => {
  await supabase.auth.updateUser({ password: newPassword });
  nav('/');
};
```

---

## 6. Editar Perfil + upload de avatar 

### 6.1 Serviço auxiliar 

`src/services/profileService.js`

```js
import { supabase } from './supabase';

export const getProfile = () =>
  supabase.from('profiles').select('*').single();

export const updateProfile = async ({ full_name, file }) => {
  let avatar_url;
  if (file) {
    const fileExt = file.name.split('.').pop();
    const fileName = `${crypto.randomUUID()}.${fileExt}`;
    const { error: upErr } = await supabase.storage
      .from('avatars')
      .upload(fileName, file);
    if (upErr) throw upErr;
    avatar_url = fileName;
  }

  const updates = { full_name, avatar_url, updated_at: new Date() };
  const { data, error } = await supabase
    .from('profiles')
    .update(updates)
    .eq('id', supabase.auth.user().id)
    .select()
    .single();
  if (error) throw error;
  return data;
};
```

6.2 **ProfilePage.jsx** 

```jsx
import { useQuery, useMutation } from '@tanstack/react-query';
import { getProfile, updateProfile } from '../../services/profileService';

export default function ProfilePage() {
  const { data: profile } = useQuery(['profile'], getProfile);
  const mutation = useMutation(updateProfile);

  const handleSubmit = e => {
    e.preventDefault();
    const file = e.target.avatar.files[0];
    mutation.mutate({ full_name: e.target.full_name.value, file });
  };

  const avatarUrl = profile?.avatar_url
    ? supabase.storage.from('avatars').getPublicUrl(profile.avatar_url).data.publicUrl
    : 'https://via.placeholder.com/150';

  /* form exibindo avatar, nome e botão Salvar */
}
```

---

## 7. Papel de administrador 

### 7.1 Marcar administradores 

*No início do curso, crie manualmente um usuário e, no console SQL, rode:*

```sql
update profiles set is_admin = true where id = 'UUID_DO_ADMIN';
```

> Esse campo será copiado para o *JWT* na próxima sessão do usuário.

**Dica:**  adicione um *trigger* que espelhe `is_admin` para `auth.users.raw_user_meta_data`.

7.2 Painel **AdminUsersPage.jsx** 

Objetivos:
 
2. Listar todos os perfis (`select * from profiles`) – só admins veem pela policy.
 
4. Botões *Tornar admin / Remover admin*.
 
6. Botão **Excluir usuário**  (cascata).

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

const queryClient = useQueryClient();

const { data: users } = useQuery(['users'], () =>
  supabase.from('profiles').select('id, full_name, is_admin, avatar_url')
);

const promote = useMutation(
  ({ id, is_admin }) =>
    supabase.from('profiles').update({ is_admin }).eq('id', id),
  {
    onSuccess: () => queryClient.invalidateQueries(['users']),
  }
);

const remove = useMutation(id =>
  supabase.rpc('delete_user', { user_id: id }) /* função SQL ver abaixo */,
  { onSuccess: () => queryClient.invalidateQueries(['users']) }
);
```

Função SQL (executar uma única vez):

```sql
create or replace function delete_user(user_id uuid)
returns void language plpgsql security definer as $$
begin
  delete from auth.users where id = user_id;
end $$;
```

Assegure‐se de proteger a função com `SECURITY DEFINER` + **policy**  que permita somente admin invocá-la.

### 7.3 Rota 

```jsx
<Route
  path="/admin/users"
  element={
    <AdminRoute>
      <AdminUsersPage />
    </AdminRoute>
  }
/>
```

---

8. Ajustes no `Header`

```jsx
import { useAuth } from '../contexts/AuthContext';

const Header = ({ cartCount }) => {
  const { user, isAdmin } = useAuth();

  /* … links normais … */
  {user ? (
     <div className="dropdown text-end">
        <a className="d-block link-light text-decoration-none dropdown-toggle" data-bs-toggle="dropdown">
          {user.user_metadata.full_name || user.email}
        </a>
        <ul className="dropdown-menu dropdown-menu-end">
          {isAdmin && <li><Link className="dropdown-item" to="/admin/users">Admin</Link></li>}
          <li><Link className="dropdown-item" to="/profile">Perfil</Link></li>
          <li><hr className="dropdown-divider" /></li>
          <li><button onClick={() => supabase.auth.signOut()} className="dropdown-item">Sair</button></li>
        </ul>
     </div>
  ) : (
     <Link to="/login" className="btn btn-outline-light">Entrar</Link>
  )}
```

---

## 9. Teste completo do fluxo 
 
2. **Registrar**  novo usuário → confirmar e-mail.
 
4. **Login**  → editar perfil, trocar senha.
 
6. **Esqueci a senha**  → link de reset → define nova senha.
 
8. **Promover a admin**  via SQL → login novamente → acesso ao painel.
 
10. **Admin panel** : listar usuários, promover/outros, remover.

---

## 10. Exercício proposto 

> Adicione login OAuth (Google).

Implemente *soft delete* no painel em vez de deleção permanente.

No CRUD de produtos, exiba o nome do usuário que criou/atualizou cada item (use coluna `owner` + RLS).

---

## 11. Conclusão 

Nesta aula você:

- Configurou **Supabase Auth** , RLS e *claims*;
 
- Criou **Contexto de Autenticação**  em React;
 
- Montou **rotas protegidas**  e **painel de administração**  controlado por claims;
 
- Implementou **todos os fluxos de conta**  de uma aplicação moderna.
