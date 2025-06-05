# Internal Developer Guide for Meteor 3.x + Solid Apps

This documentation provides guidance for working on Meteor-based applications using MeteorJS version 3.x and SolidJS as the front-end rendering framework.

---

## Project Basics

* JavaScript dependencies are managed using **npm**.
* The codebase uses **MeteorJS (v3.2.2 or higher)**: [https://www.meteor.com/](https://www.meteor.com/)
* SolidJS is used for rendering UI components: [https://www.solidjs.com/](https://www.solidjs.com/)
* **MongoDB** is the primary database.
* **GitHub** is used for source control and tracking work items.
* **GitHub Actions** handles CI/CD pipelines.

---

## Local Development

Use the latest **LTS version of Node.js (>= 22.x < 24)** when developing locally.

### Commands

Run lint checks:

```bash
meteor npm run lint
```

Start the app:

```bash
meteor npm run start
```

---

## Meteor 3.x Key Differences

Meteor 3.x introduces several enhancements:

### Async/Await Support

All server code should use `async/await`. Meteor methods and database operations now support asynchronous versions.

#### Meteor Method Calls

**Old (2.x):**

```js
Meteor.call('methodName', arg1, arg2, callback)
```

**New (3.x):**

```js
await Meteor.callAsync('methodName', arg1, arg2)
```

#### MongoDB Collection Operations

**Old (2.x):**

```js
Collection.find({}).fetch()
Collection.findOne({})
Collection.insert({})
```

**New (3.x):**

```js
await Collection.find({}).fetchAsync()
await Collection.findOneAsync({})
await Collection.insertAsync({})
```

---

## Solid + Meteor Integration

### Data Subscriptions with Signals

```jsx
// TaskList.jsx
import { createSignal, createEffect, onCleanup } from 'solid-js';
import { For, Show } from 'solid-js/web';
import { Meteor } from 'meteor/meteor';
import { TasksCollection } from '../api/TasksCollection';

function TaskList() {
  const [tasks, setTasks] = createSignal([]);
  const [isLoading, setIsLoading] = createSignal(true);
  let subscription;

  createEffect(() => {
    subscription = Meteor.subscribe('tasks', {
      onReady() {
        const taskData = TasksCollection.find({}).fetch();
        setTasks(taskData);
        setIsLoading(false);
      },
      onError(error) {
        console.error('Subscription error:', error);
        setIsLoading(false);
      }
    });
  });

  onCleanup(() => {
    if (subscription) {
      subscription.stop();
    }
  });

  return (
    <div class="task-list">
      <Show when={isLoading()} fallback={
        <ul>
          <For each={tasks()}>
            {(task) => <li>{task.title}</li>}
          </For>
        </ul>
      }>
        <div>Loading...</div>
      </Show>
    </div>
  );
}

export default TaskList;
```

### Reactive Meteor Store

```js
// stores/meteorStore.js
import { createSignal, createEffect } from 'solid-js';
import { Meteor } from 'meteor/meteor';

export function createMeteorStore(collection, subscriptionName, selector = {}, options = {}) {
  const [data, setData] = createSignal([]);
  const [isLoading, setIsLoading] = createSignal(true);
  const [error, setError] = createSignal(null);
  let subscription;

  const subscribe = () => {
    setIsLoading(true);
    setError(null);

    subscription = Meteor.subscribe(subscriptionName, {
      onReady() {
        const result = collection.find(selector, options).fetch();
        setData(result);
        setIsLoading(false);
      },
      onError(err) {
        setError(err);
        setIsLoading(false);
      }
    });
  };

  const stop = () => {
    if (subscription) {
      subscription.stop();
    }
  };

  const refresh = () => {
    if (subscription && subscription.ready()) {
      const result = collection.find(selector, options).fetch();
      setData(result);
    }
  };

  return {
    data,
    isLoading,
    error,
    subscribe,
    stop,
    refresh
  };
}
```

Usage:

```jsx
import { onMount, onCleanup } from 'solid-js';
import { createMeteorStore } from '../stores/meteorStore';
import { TasksCollection } from '../api/TasksCollection';

function TasksComponent() {
  const taskStore = createMeteorStore(TasksCollection, 'tasks');

  onMount(() => {
    taskStore.subscribe();
  });

  onCleanup(() => {
    taskStore.stop();
  });

  return (
    <div>
      <Show when={taskStore.isLoading()} fallback={
        <For each={taskStore.data()}>
          {(task) => <div>{task.title}</div>}
        </For>
      }>
        <div>Loading...</div>
      </Show>
    </div>
  );
}
```

### Async Method Calls in Components

```jsx
// AddTask.jsx
import { createSignal } from 'solid-js';
import { Meteor } from 'meteor/meteor';

function AddTask() {
  const [title, setTitle] = createSignal('');
  const [isSubmitting, setIsSubmitting] = createSignal(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!title().trim()) return;

    setIsSubmitting(true);

    try {
      await Meteor.callAsync('tasks.insert', title());
      setTitle('');
    } catch (error) {
      console.error('Error adding task:', error);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={title()}
        onInput={(e) => setTitle(e.target.value)}
        placeholder="Task title"
        disabled={isSubmitting()}
      />
      <button type="submit" disabled={isSubmitting()}>
        {isSubmitting() ? 'Adding...' : 'Add Task'}
      </button>
    </form>
  );
}

export default AddTask;
```

---

## JavaScript Code Style

Please follow the conventions below to maintain consistency:

* **Strings**: Use single quotes `'like this'`.
* **Indentation**: Use 2 spaces.
* **Variables**: Use `const` and `let`; avoid `var`.
* **String building**: Use template literals `` `Hello ${name}` ``.
* **Functions**: Prefer regular function declarations over arrow functions.
* **Asynchronous Code**: Use `async`/`await`.
* **Modules**: Use `import`/`export` (ES6 modules); avoid CommonJS.
* **Naming**:
  * `camelCase` for variables/functions
  * `PascalCase` for Solid components and class names
  * `UPPER_CASE` for constants
* Use optional chaining (`?.`) and nullish coalescing (`??`) where appropriate.

---

## Solid Component Guidelines

### Component Structure

```jsx
import { createSignal, createEffect, onMount, onCleanup } from 'solid-js';
import { Show, For } from 'solid-js/web';
import { Meteor } from 'meteor/meteor';

function MyComponent(props) {
  // 1. Signals (reactive state)
  const [localState, setLocalState] = createSignal('');
  const [isLoading, setIsLoading] = createSignal(false);

  // 2. Derived values (computed)
  const filteredItems = () => props.items?.filter(item => item.active) || [];
  const itemCount = () => filteredItems().length;

  // 3. Effects for side effects
  createEffect(() => {
    // React to signal changes
    console.log('Local state changed:', localState());
  });

  // 4. Lifecycle hooks
  onMount(() => {
    // Component mounted logic
  });

  onCleanup(() => {
    // Cleanup logic
  });

  // 5. Event handlers
  const handleAction = async () => {
    try {
      await Meteor.callAsync('myMethod');
    } catch (error) {
      console.error(error);
    }
  };

  // 6. JSX render
  return (
    <div class="my-component">
      <h2>{props.title}</h2>
      
      <Show when={itemCount() > 0} fallback={<p>No items found</p>}>
        <For each={filteredItems()}>
          {(item) => <div>{item.name}</div>}
        </For>
      </Show>
      
      <button onClick={handleAction}>
        Action
      </button>
    </div>
  );
}

export default MyComponent;
```

### State Management with Context

```jsx
// contexts/AppContext.jsx
import { createContext, useContext, createSignal } from 'solid-js';
import { Meteor } from 'meteor/meteor';

const AppContext = createContext();

export function AppProvider(props) {
  const [user, setUser] = createSignal(null);
  const [isLoggedIn, setIsLoggedIn] = createSignal(false);
  const [tasks, setTasks] = createSignal([]);

  const login = async (email, password) => {
    try {
      await Meteor.loginWithPassword(email, password);
      const currentUser = Meteor.user();
      setUser(currentUser);
      setIsLoggedIn(true);
    } catch (error) {
      throw error;
    }
  };

  const logout = async () => {
    try {
      await Meteor.logout();
      setUser(null);
      setIsLoggedIn(false);
    } catch (error) {
      throw error;
    }
  };

  const addTask = async (title) => {
    try {
      await Meteor.callAsync('tasks.insert', title);
      // Refresh tasks
      loadTasks();
    } catch (error) {
      throw error;
    }
  };

  const loadTasks = () => {
    const subscription = Meteor.subscribe('tasks', {
      onReady() {
        const taskData = TasksCollection.find({}).fetch();
        setTasks(taskData);
      }
    });
  };

  const contextValue = {
    user,
    isLoggedIn,
    tasks,
    login,
    logout,
    addTask,
    loadTasks
  };

  return (
    <AppContext.Provider value={contextValue}>
      {props.children}
    </AppContext.Provider>
  );
}

export function useApp() {
  const context = useContext(AppContext);
  if (!context) {
    throw new Error('useApp must be used within AppProvider');
  }
  return context;
}
```

Usage in component:

```jsx
import { useApp } from '../contexts/AppContext';

function TasksComponent() {
  const { tasks, addTask, loadTasks } = useApp();

  onMount(() => {
    loadTasks();
  });

  return (
    <div>
      <For each={tasks()}>
        {(task) => <div>{task.title}</div>}
      </For>
    </div>
  );
}
```

---

## TypeScript Support (Optional)

For TypeScript projects, add proper type definitions:

```tsx
// types.ts
export interface Task {
  _id: string;
  title: string;
  completed: boolean;
  createdAt: Date;
  userId: string;
}

export interface TaskListProps {
  showCompleted?: boolean;
}
```

```tsx
// TaskList.tsx
import { createSignal, createEffect, onCleanup, Component } from 'solid-js';
import { For, Show } from 'solid-js/web';
import { Meteor } from 'meteor/meteor';
import { Task, TaskListProps } from '../types';

const TaskList: Component<TaskListProps> = (props) => {
  const [tasks, setTasks] = createSignal<Task[]>([]);
  const [isLoading, setIsLoading] = createSignal<boolean>(true);
  let subscription: any;

  createEffect(() => {
    subscription = Meteor.subscribe('tasks', {
      onReady() {
        const taskData = TasksCollection.find({}).fetch() as Task[];
        setTasks(taskData);
        setIsLoading(false);
      },
      onError(error: any) {
        console.error('Subscription error:', error);
        setIsLoading(false);
      }
    });
  });

  onCleanup(() => {
    if (subscription) {
      subscription.stop();
    }
  });

  const filteredTasks = (): Task[] => {
    if (props.showCompleted === false) {
      return tasks().filter(task => !task.completed);
    }
    return tasks();
  };

  return (
    <div class="task-list">
      <Show when={isLoading()} fallback={
        <ul>
          <For each={filteredTasks()}>
            {(task: Task) => <li>{task.title}</li>}
          </For>
        </ul>
      }>
        <div>Loading...</div>
      </Show>
    </div>
  );
};

export default TaskList;
```

---

## Testing with Solid

### Component Testing

```js
// TaskList.test.jsx
import { render, screen, fireEvent, waitFor } from '@solidjs/testing-library';
import { vi } from 'vitest';
import TaskList from './TaskList';

// Mock Meteor
const mockMeteor = {
  subscribe: vi.fn(),
  callAsync: vi.fn()
};

vi.mock('meteor/meteor', () => ({
  Meteor: mockMeteor
}));

describe('TaskList', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  test('should display loading state initially', () => {
    mockMeteor.subscribe.mockReturnValue({
      ready: () => false,
      onReady: vi.fn(),
      onError: vi.fn()
    });

    render(() => <TaskList />);
    
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  test('should display tasks when loaded', async () => {
    const mockTasks = [
      { _id: '1', title: 'Test Task 1', completed: false },
      { _id: '2', title: 'Test Task 2', completed: true }
    ];

    mockMeteor.subscribe.mockImplementation((name, callbacks) => {
      // Simulate ready callback
      setTimeout(() => callbacks.onReady(), 0);
      return { ready: () => true, stop: vi.fn() };
    });

    // Mock collection
    global.TasksCollection = {
      find: () => ({
        fetch: () => mockTasks
      })
    };

    render(() => <TaskList />);

    await waitFor(() => {
      expect(screen.getByText('Test Task 1')).toBeInTheDocument();
      expect(screen.getByText('Test Task 2')).toBeInTheDocument();
    });
  });
});
```

---

## Common Patterns

### Loading States with Resource

```jsx
import { createResource, Suspense } from 'solid-js';
import { Meteor } from 'meteor/meteor';

function DataComponent() {
  const [data] = createResource(async () => {
    const subscription = Meteor.subscribe('myData');
    await new Promise(resolve => {
      subscription.ready() ? resolve() : subscription.onReady(resolve);
    });
    return MyCollection.find({}).fetch();
  });

  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <Show when={data.error} fallback={
          <For each={data()}>
            {(item) => <div>{item.name}</div>}
          </For>
        }>
          <div>Error: {data.error.message}</div>
        </Show>
      </Suspense>
    </div>
  );
}
```

### Form Handling with Validation

```jsx
import { createSignal, createMemo } from 'solid-js';
import { Meteor } from 'meteor/meteor';

function UserForm() {
  const [formData, setFormData] = createSignal({
    name: '',
    email: ''
  });
  const [errors, setErrors] = createSignal({});
  const [isSubmitting, setIsSubmitting] = createSignal(false);

  const isValid = createMemo(() => {
    const data = formData();
    const newErrors = {};

    if (!data.name.trim()) {
      newErrors.name = 'Name is required';
    }

    if (!data.email.trim()) {
      newErrors.email = 'Email is required';
    } else if (!/\S+@\S+\.\S+/.test(data.email)) {
      newErrors.email = 'Email is invalid';
    }

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  });

  const handleChange = (field) => (e) => {
    setFormData(prev => ({
      ...prev,
      [field]: e.target.value
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!isValid()) return;

    setIsSubmitting(true);

    try {
      await Meteor.callAsync('users.create', formData());
      setFormData({ name: '', email: '' });
      setErrors({});
    } catch (error) {
      console.error('Form submission error:', error);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div class="form-group">
        <label for="name">Name:</label>
        <input
          id="name"
          type="text"
          value={formData().name}
          onInput={handleChange('name')}
          classList={{ error: !!errors().name }}
        />
        <Show when={errors().name}>
          <span class="error-message">{errors().name}</span>
        </Show>
      </div>

      <div class="form-group">
        <label for="email">Email:</label>
        <input
          id="email"
          type="email"
          value={formData().email}
          onInput={handleChange('email')}
          classList={{ error: !!errors().email }}
        />
        <Show when={errors().email}>
          <span class="error-message">{errors().email}</span>
        </Show>
      </div>

      <button type="submit" disabled={!isValid() || isSubmitting()}>
        {isSubmitting() ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

### Reactive Search with Debouncing

```jsx
import { createSignal, createEffect, createMemo } from 'solid-js';
import { debounce } from '@solid-primitives/scheduled';
import { Meteor } from 'meteor/meteor';

function SearchComponent() {
  const [searchTerm, setSearchTerm] = createSignal('');
  const [results, setResults] = createSignal([]);
  const [isSearching, setIsSearching] = createSignal(false);

  // Debounced search term
  const debouncedSearchTerm = debounce(searchTerm, 300);

  // Effect for performing search
  createEffect(async () => {
    const term = debouncedSearchTerm();
    
    if (term.length >= 3) {
      setIsSearching(true);
      try {
        const searchResults = await Meteor.callAsync('search.perform', term);
        setResults(searchResults);
      } catch (error) {
        console.error('Search error:', error);
        setResults([]);
      } finally {
        setIsSearching(false);
      }
    } else {
      setResults([]);
    }
  });

  return (
    <div class="search-component">
      <input
        type="text"
        value={searchTerm()}
        onInput={(e) => setSearchTerm(e.target.value)}
        placeholder="Search..."
      />
      
      <Show when={isSearching()}>
        <div>Searching...</div>
      </Show>
      
      <Show when={results().length > 0}>
        <ul class="search-results">
          <For each={results()}>
            {(result) => (
              <li>
                <strong>{result.title}</strong>
                <p>{result.description}</p>
              </li>
            )}
          </For>
        </ul>
      </Show>
    </div>
  );
}
```