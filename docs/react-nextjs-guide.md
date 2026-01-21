# Guide Complet React & Next.js

Ce guide rassemble les informations essentielles pour être performant en React et Next.js, basé sur la documentation officielle.

---

## Table des matières

1. [React - Hooks essentiels](#react---hooks-essentiels)
2. [React - Gestion d'état avancée](#react---gestion-détat-avancée)
3. [React - Optimisation des performances](#react---optimisation-des-performances)
4. [React - Manipulation du DOM](#react---manipulation-du-dom)
5. [Next.js - App Router](#nextjs---app-router)
6. [Next.js - Data Fetching](#nextjs---data-fetching)
7. [Next.js - Server Actions](#nextjs---server-actions)
8. [Next.js - Optimisations](#nextjs---optimisations)
9. [Patterns et bonnes pratiques](#patterns-et-bonnes-pratiques)

---

## React - Hooks essentiels

### useState

Le hook de base pour gérer l'état local d'un composant.

```tsx
const [count, setCount] = useState(0);

// Mise à jour avec la valeur précédente (pattern recommandé)
setCount(prev => prev + 1);
```

### useEffect

Pour les effets de bord (API calls, subscriptions, DOM manipulation).

```tsx
useEffect(() => {
  const connection = createConnection(options);
  connection.connect();
  
  // Cleanup function (important pour éviter les fuites mémoire)
  return () => connection.disconnect();
}, [options]); // Dépendances
```

### useCallback

Cache une fonction pour éviter sa recréation à chaque render.

```tsx
const handleAddTodo = useCallback((text) => {
  const newTodo = { id: nextId++, text };
  // Utiliser l'updater function pour éviter la dépendance sur todos
  setTodos(todos => [...todos, newTodo]);
}, []); // ✅ Pas besoin de todos dans les dépendances
```

### useMemo

Cache le résultat d'un calcul coûteux.

```tsx
const visibleTodos = useMemo(
  () => filterTodos(todos, tab),
  [todos, tab]
);
```

**Différence useMemo vs useCallback:**
- `useMemo` cache le **résultat** d'une fonction
- `useCallback` cache la **fonction elle-même**

```tsx
// useMemo: cache le résultat de computeRequirements(product)
const requirements = useMemo(() => computeRequirements(product), [product]);

// useCallback: cache la fonction handleSubmit elle-même
const handleSubmit = useCallback((orderDetails) => {
  post('/product/' + productId + '/buy', { referrer, orderDetails });
}, [productId, referrer]);
```

### useRef

Pour accéder directement aux éléments DOM ou stocker des valeurs mutables.

```tsx
import { useRef } from 'react';

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>Focus the input</button>
    </>
  );
}
```

---

## React - Gestion d'état avancée

### useReducer + Context API

Pattern recommandé pour la gestion d'état complexe à travers plusieurs composants.

**1. Créer le Context et le Provider:**

```tsx
// TasksContext.tsx
import { createContext, useContext, useReducer } from 'react';

const TasksContext = createContext(null);
const TasksDispatchContext = createContext(null);

export function TasksProvider({ children }) {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);

  return (
    <TasksContext value={tasks}>
      <TasksDispatchContext value={dispatch}>
        {children}
      </TasksDispatchContext>
    </TasksContext>
  );
}

// Custom hooks pour consommer le context
export function useTasks() {
  return useContext(TasksContext);
}

export function useTasksDispatch() {
  return useContext(TasksDispatchContext);
}

// Reducer
function tasksReducer(tasks, action) {
  switch (action.type) {
    case 'added':
      return [...tasks, { id: action.id, text: action.text, done: false }];
    case 'changed':
      return tasks.map(t => t.id === action.task.id ? action.task : t);
    case 'deleted':
      return tasks.filter(t => t.id !== action.id);
    default:
      throw Error('Unknown action: ' + action.type);
  }
}
```

**2. Utiliser dans les composants:**

```tsx
// App.tsx
import { TasksProvider } from './TasksContext';

export default function TaskApp() {
  return (
    <TasksProvider>
      <h1>Mes tâches</h1>
      <AddTask />
      <TaskList />
    </TasksProvider>
  );
}

// AddTask.tsx
import { useState } from 'react';
import { useTasksDispatch } from './TasksContext';

export default function AddTask() {
  const [text, setText] = useState('');
  const dispatch = useTasksDispatch();
  
  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={() => {
        setText('');
        dispatch({ type: 'added', id: nextId++, text });
      }}>
        Ajouter
      </button>
    </>
  );
}
```

### useImperativeHandle

Expose une API limitée à un composant parent via ref.

```tsx
import { useRef, useImperativeHandle, forwardRef } from 'react';

const MyInput = forwardRef(function MyInput(props, ref) {
  const realInputRef = useRef(null);
  
  useImperativeHandle(ref, () => ({
    // Expose uniquement focus, rien d'autre
    focus() {
      realInputRef.current.focus();
    },
    scrollIntoView() {
      realInputRef.current.scrollIntoView();
    },
  }), []);

  return <input {...props} ref={realInputRef} />;
});
```

---

## React - Optimisation des performances

### Quand utiliser useMemo/useCallback?

1. **Calculs coûteux** - Filtrage/tri de grandes listes
2. **Stabilisation de dépendances** - Pour éviter les re-renders de useEffect
3. **Props vers composants mémoïsés** - Avec `React.memo()`

### Exemple: Stabiliser un objet pour useEffect

```tsx
function ChatRoom({ roomId }) {
  const [message, setMessage] = useState('');

  // ✅ L'objet ne change que quand roomId change
  const options = useMemo(() => ({
    serverUrl: 'https://localhost:1234',
    roomId: roomId
  }), [roomId]);

  useEffect(() => {
    const connection = createConnection(options);
    connection.connect();
    return () => connection.disconnect();
  }, [options]); // ✅ Ne se déclenche que quand options change réellement
}
```

### Pattern Updater Function

Évite les dépendances inutiles dans useCallback:

```tsx
// ❌ Mauvais: todos dans les dépendances
const handleAddTodo = useCallback((text) => {
  setTodos([...todos, { id: nextId++, text }]);
}, [todos]);

// ✅ Bon: pas de dépendance sur todos
const handleAddTodo = useCallback((text) => {
  setTodos(todos => [...todos, { id: nextId++, text }]);
}, []);
```

---

## React - Manipulation du DOM

### Accéder à un élément DOM

```tsx
import { useRef } from 'react';

export default function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>Focus</button>
    </>
  );
}
```

### Exposer une API limitée avec forwardRef

```tsx
import { forwardRef, useRef, useImperativeHandle } from 'react';

const MyInput = forwardRef(function MyInput(props, ref) {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus() {
      inputRef.current.focus();
    },
    scrollIntoView() {
      inputRef.current.scrollIntoView();
    },
  }), []);

  return <input {...props} ref={inputRef} />;
});
```

---

## Next.js - App Router

### Structure des fichiers

```
app/
├── layout.tsx       # Layout racine
├── page.tsx         # Page d'accueil (/)
├── loading.tsx      # UI de chargement
├── error.tsx        # Gestion des erreurs
├── not-found.tsx    # Page 404
├── blog/
│   ├── page.tsx     # /blog
│   └── [slug]/
│       └── page.tsx # /blog/:slug (route dynamique)
```

### Server Components vs Client Components

**Server Components (par défaut):**
- Peuvent faire du data fetching directement
- Pas d'accès aux hooks React (useState, useEffect)
- Pas d'accès aux APIs browser

**Client Components:**
- Ajoutez `'use client'` en haut du fichier
- Accès aux hooks et APIs browser
- Peuvent recevoir des props de Server Components

```tsx
// Server Component (par défaut)
export default async function Page() {
  const data = await fetch('https://api.example.com/data');
  return <div>{/* ... */}</div>;
}

// Client Component
'use client'
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

### Routes dynamiques avec generateStaticParams

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

export default async function Page({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);
  return <article>{post.content}</article>;
}
```

---

## Next.js - Data Fetching

### Stratégies de cache avec fetch()

```tsx
export default async function Page() {
  // 1. Données statiques (cache par défaut)
  // Similaire à getStaticProps
  const staticData = await fetch('https://...', { cache: 'force-cache' });

  // 2. Données dynamiques (pas de cache)
  // Similaire à getServerSideProps
  const dynamicData = await fetch('https://...', { cache: 'no-store' });

  // 3. Données revalidées (ISR)
  // Revalidation toutes les 10 secondes
  const revalidatedData = await fetch('https://...', {
    next: { revalidate: 10 },
  });

  return <div>...</div>;
}
```

### Data fetching dans les layouts

```tsx
// app/dashboard/layout.tsx
import { getUser } from '@/lib/data';

export default async function Layout({ children }) {
  const user = await getUser('1');

  return (
    <>
      <nav>
        <UserName user={user.name} />
      </nav>
      {children}
    </>
  );
}
```

**Note:** Next.js déduplique automatiquement les requêtes fetch identiques.

### Pattern Server Component + Client Component

```tsx
// page.tsx (Server Component)
import HomePage from './home-page';

async function getPosts() {
  const res = await fetch('https://...');
  return res.json();
}

export default async function Page() {
  const recentPosts = await getPosts();
  // Passe les données au Client Component
  return <HomePage recentPosts={recentPosts} />;
}

// home-page.tsx (Client Component)
'use client'

export default function HomePage({ recentPosts }) {
  // Peut utiliser useState, useEffect, etc.
  return <div>{/* ... */}</div>;
}
```

---

## Next.js - Server Actions

### Créer une Server Action

```tsx
// app/actions.ts
'use server'

import { z } from 'zod';
import { revalidatePath } from 'next/cache';

const schema = z.object({
  email: z.string().email('Email invalide'),
  title: z.string().min(1, 'Titre requis'),
});

export async function createPost(formData: FormData) {
  // Validation avec Zod
  const validatedFields = schema.safeParse({
    email: formData.get('email'),
    title: formData.get('title'),
  });

  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors,
      message: 'Validation échouée',
    };
  }

  // Mutation des données
  await saveToDatabase(validatedFields.data);

  // Revalidation du cache
  revalidatePath('/posts');
  
  return { message: 'Post créé avec succès' };
}
```

### Utiliser avec useActionState (React 19)

```tsx
'use client'

import { useActionState } from 'react';
import { createPost } from '@/app/actions';

const initialState = { message: '' };

export function CreatePostForm() {
  const [state, formAction, pending] = useActionState(createPost, initialState);

  return (
    <form action={formAction}>
      <label htmlFor="title">Titre</label>
      <input type="text" id="title" name="title" required />
      
      <label htmlFor="content">Contenu</label>
      <textarea id="content" name="content" required />
      
      {state?.message && (
        <p aria-live="polite">{state.message}</p>
      )}
      
      <button disabled={pending}>
        {pending ? 'Création...' : 'Créer'}
      </button>
    </form>
  );
}
```

### Revalidation du cache

```tsx
import { revalidatePath, revalidateTag } from 'next/cache';

export async function updatePost(formData: FormData) {
  'use server'
  
  // Mise à jour des données
  await updateInDatabase(formData);

  // Option 1: Revalidate par chemin
  revalidatePath('/posts');
  
  // Option 2: Revalidate par tag
  revalidateTag('posts');
}
```

---

## Next.js - Optimisations

### Composant Image

Optimisation automatique des images avec lazy loading.

```tsx
import Image from 'next/image';
import ProfileImage from './profile.png';

export default function Page() {
  return (
    <Image
      src={ProfileImage}
      alt="Photo de profil"
      // width et height automatiquement fournis pour les imports locaux
      placeholder="blur" // Effet blur pendant le chargement
    />
  );
}

// Pour les images distantes
<Image
  src="https://example.com/image.jpg"
  alt="Description"
  width={500}
  height={300}
  loading="lazy" // Par défaut
/>
```

### Optimisation des polices

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap', // Évite le FOIT (Flash of Invisible Text)
});

export default function RootLayout({ children }) {
  return (
    <html lang="fr" className={inter.className}>
      <body>{children}</body>
    </html>
  );
}
```

### Metadata pour le SEO

```tsx
// Metadata statique
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Mon Blog',
  description: 'Articles sur le développement web',
  openGraph: {
    title: 'Mon Blog',
    description: 'Articles sur le développement web',
    images: ['/og-image.jpg'],
  },
};

// Metadata dynamique
export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPost(params.slug);
  
  return {
    title: post.title,
    description: post.excerpt,
  };
}
```

### Loading UI avec Suspense

```tsx
// app/posts/loading.tsx
export default function Loading() {
  return <div>Chargement des posts...</div>;
}

// Ou avec Suspense pour un contrôle plus fin
import { Suspense } from 'react';

export default function Page() {
  return (
    <div>
      <h1>Blog</h1>
      <Suspense fallback={<PostsSkeleton />}>
        <Posts />
      </Suspense>
    </div>
  );
}
```

---

## Patterns et bonnes pratiques

### 1. Composition over Inheritance

Privilégier la composition de composants plutôt que l'héritage.

```tsx
// ✅ Bon: Composition
function Card({ children, header }) {
  return (
    <div className="card">
      {header && <div className="card-header">{header}</div>}
      <div className="card-body">{children}</div>
    </div>
  );
}

// Utilisation
<Card header={<h2>Titre</h2>}>
  <p>Contenu</p>
</Card>
```

### 2. Colocation des données

Fetch les données au plus proche de là où elles sont utilisées.

```tsx
// ✅ Bon: Données fetchées dans le composant qui les utilise
async function UserProfile({ userId }) {
  const user = await getUser(userId);
  return <div>{user.name}</div>;
}

// ❌ Éviter: Props drilling sur plusieurs niveaux
```

### 3. Pattern Client Component Wrapper

Pour les librairies qui ne supportent pas le SSR.

```tsx
// components/map-wrapper.tsx
'use client'

import dynamic from 'next/dynamic';

const Map = dynamic(
  () => import('./map').then((mod) => mod.Map),
  { ssr: false, loading: () => <div>Chargement de la carte...</div> }
);

export function MapWrapper(props) {
  return <Map {...props} />;
}
```

### 4. Gestion des erreurs

```tsx
// app/error.tsx
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Une erreur est survenue</h2>
      <button onClick={() => reset()}>Réessayer</button>
    </div>
  );
}
```

### 5. Types utiles TypeScript

```tsx
// Types pour les props de page
type PageProps = {
  params: { slug: string };
  searchParams: { [key: string]: string | string[] | undefined };
};

// Types pour les Server Actions
type ActionState = {
  message: string;
  errors?: Record<string, string[]>;
};

// Type pour les métadonnées
import type { Metadata } from 'next';
```

---

## Checklist de performance

- [ ] Utiliser Server Components par défaut
- [ ] Limiter `'use client'` aux composants interactifs
- [ ] Utiliser `useMemo`/`useCallback` pour les calculs coûteux
- [ ] Optimiser les images avec `next/image`
- [ ] Utiliser les fonts de `next/font`
- [ ] Implémenter les loading states avec `loading.tsx` ou `Suspense`
- [ ] Configurer correctement le cache des fetch
- [ ] Utiliser `generateStaticParams` pour les routes dynamiques
- [ ] Valider les formulaires côté serveur avec Zod
- [ ] Revalider le cache après les mutations

---

## Ressources

- [Documentation React](https://react.dev)
- [Documentation Next.js](https://nextjs.org/docs)
- [Patterns React](https://react.dev/learn)
- [App Router Migration Guide](https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration)
