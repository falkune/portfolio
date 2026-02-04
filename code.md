# TP React : CRUD avec API et JWT

Ce TP montre comment créer une application React pour gérer des tâches (`Task`) via un backend API avec JWT.  
Les étudiants apprendront à :  

- Faire des requêtes HTTP avec `fetch` et token JWT  
- Afficher une liste d’éléments  
- Ajouter et supprimer des tâches  
- Utiliser React Router 19 pour naviguer entre les pages  

---

## 1️⃣ Structure du projet

app/
├─ api.ts

├─ components/

│ ├─ TaskList.tsx

│ └─ TaskForm.tsx

├─ pages/

│ └─ TasksPage.tsx

├─ routes/

│ ├─ home.tsx

│ └─ tasks.tsx

├─ routes.ts

└─ root.tsx


---

## 2️⃣ `api.ts` : communication avec l’API

```ts
const API_URL = "http://localhost:3000";

// Token en dur pour le TP
const TOKEN = "<VOTRE_TOKEN_JWT>";

function getHeaders() {
  return {
    "Content-Type": "application/json",
    Authorization: `Bearer ${TOKEN}`,
  };
}

export async function getTasks() {
  const res = await fetch(`${API_URL}/api/tasks`, {
    headers: getHeaders(),
  });
  return res.json();
}

export async function createTask(task: { title: string; description?: string; status?: string; dueDate?: string }) {
  const res = await fetch(`${API_URL}/api/tasks`, {
    method: "POST",
    headers: getHeaders(),
    body: JSON.stringify(task),
  });
  return res.json();
}

export async function deleteTask(id: number) {
  await fetch(`${API_URL}/api/tasks/${id}`, {
    method: "DELETE",
    headers: getHeaders(),
  });
}
```

### TaskList.tsx : afficher et supprimer des tâches

```ts
type Task = {
  id: number;
  title: string;
  description: string;
  status: string;
  dueDate?: string;
};

type Props = {
  tasks: Task[];
  onDelete: (id: number) => void;
};

export function TaskList({ tasks = [], onDelete }: Props) {
  if (!tasks.length) return <p>Aucune tâche</p>;

  return (
    <ul>
      {tasks.map((task) => (
        <li key={task.id}>
          <strong>{task.title}</strong> - {task.status}
          <button onClick={() => onDelete(task.id)}>Supprimer</button>
        </li>
      ))}
    </ul>
  );
}
```

#### TaskList.tsx : afficher et supprimer des tâches

```ts
type Task = {
  id: number;
  title: string;
  description: string;
  status: string;
  dueDate?: string;
};

type Props = {
  tasks: Task[];
  onDelete: (id: number) => void;
};

export function TaskList({ tasks = [], onDelete }: Props) {
  if (!tasks.length) return <p>Aucune tâche</p>;

  return (
    <ul>
      {tasks.map((task) => (
        <li key={task.id}>
          <strong>{task.title}</strong> - {task.status}
          <button onClick={() => onDelete(task.id)}>Supprimer</button>
        </li>
      ))}
    </ul>
  );
}
```

#### TaskForm.tsx : ajouter une tâche

```ts
import { useState } from "react";
import { createTask } from "../api";

type Props = {
  onTaskAdded: () => void;
};

export function TaskForm({ onTaskAdded }: Props) {
  const [title, setTitle] = useState("");

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!title) return;

    try {
      await createTask({ title, status: "pending" });
      setTitle("");
      onTaskAdded();
    } catch (err) {
      console.error(err);
      alert("Impossible de créer la tâche");
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Titre de la tâche"
        required
      />
      <button type="submit">Ajouter</button>
    </form>
  );
}
```

#### TasksPage.tsx : page principale des tâches

```ts
import { useEffect, useState } from "react";
import { getTasks, deleteTask } from "../api";
import { TaskList } from "../components/TaskList";
import { TaskForm } from "../components/TaskForm";

export function TasksPage() {
  const [tasks, setTasks] = useState<any[]>([]);

  const loadTasks = async () => {
    const response = await getTasks();
    setTasks(response.data); // On récupère le tableau "data" de l'API
  };

  const handleDelete = async (id: number) => {
    await deleteTask(id);
    loadTasks();
  };

  useEffect(() => {
    loadTasks();
  }, []);

  return (
    <div>
      <h1>Mes tâches</h1>
      <TaskForm onTaskAdded={loadTasks} />
      <TaskList tasks={tasks} onDelete={handleDelete} />
    </div>
  );
}

```

### routes/tasks.tsx
```ts
import { TasksPage } from "../pages/TasksPage";

export default function TasksRoute() {
  return <TasksPage />;
}

```

#### routes.ts : configurer les routes
```ts
import { type RouteConfig, index, route } from "@react-router/dev/routes";

export default [
  index("routes/home.tsx"),
  route("tasks", "routes/tasks.tsx"),
] satisfies RouteConfig;

```

### modification de Tasklist
```ts
type Task = {
  id: number;
  title: string;
  description: string;
  status: string;
  dueDate?: string;
};

type Props = {
  tasks: Task[];
  onDelete: (id: number) => void;
};

import { Task } from "./Task";

export function TaskList({ tasks, onDelete }: Props) {
  if (!tasks.length) {
    return (
      <p className="text-gray-500 italic mt-4">
        Aucune tâche
      </p>
    );
  }

  return (
    <ul className="mt-4 space-y-2">
      {tasks.map((task) => (
        <Task
          key={task.id}
          task={task}
          onDelete={onDelete}
        />
      ))}
    </ul>
  );
}
```

# creation de Task
```ts
type Task = {
  id: number;
  title: string;
  description: string;
  status: string;
  dueDate?: string;
};

type TaskProps = {
  task: Task;
  onDelete: (id: number) => void;
};

export function Task({ task, onDelete }: TaskProps) {
  return (
    <li className="flex items-center justify-between p-4 bg-white rounded-md shadow-sm border border-gray-200">
      <div>
        <strong className="block text-lg">{task.title}</strong>
        <span className="text-gray-600 text-sm">{task.status}</span>

        {task.description && (
          <p className="text-gray-700 text-sm mt-1">
            {task.description}
          </p>
        )}
      </div>

      <button
        onClick={() => onDelete(task.id)}
        className="ml-4 px-3 py-1 bg-red-500 text-white rounded-md hover:bg-red-600 transition"
      >
        Supprimer
      </button>
    </li>
  );
}
```

