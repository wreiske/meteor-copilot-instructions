# Internal Developer Guide for Meteor 3.x + Svelte Apps

This documentation provides guidance for working on Meteor-based applications using MeteorJS version 3.x and Svelte as the front-end rendering framework.

---

## Project Basics

* JavaScript dependencies are managed using **npm**.
* The codebase uses **MeteorJS (v3.2.2 or higher)**: [https://www.meteor.com/](https://www.meteor.com/)
* Svelte is used for rendering UI components: [https://svelte.dev/](https://svelte.dev/)
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

## Svelte + Meteor Integration

### Data Subscriptions with Reactive Stores

```svelte
<!-- TaskList.svelte -->
<script>
  import { onMount, onDestroy } from 'svelte';
  import { writable } from 'svelte/store';
  import { Meteor } from 'meteor/meteor';
  import { TasksCollection } from '../api/TasksCollection';

  export let showCompleted = true;

  const tasks = writable([]);
  const isLoading = writable(true);
  let subscription;

  onMount(() => {
    subscription = Meteor.subscribe('tasks', {
      onReady() {
        tasks.set(TasksCollection.find({}).fetch());
        isLoading.set(false);
      },
      onError(error) {
        console.error('Subscription error:', error);
        isLoading.set(false);
      }
    });
  });

  onDestroy(() => {
    if (subscription) {
      subscription.stop();
    }
  });
</script>

{#if $isLoading}
  <div>Loading...</div>
{:else}
  <ul>
    {#each $tasks as task (task._id)}
      <li>{task.title}</li>
    {/each}
  </ul>
{/if}
```

### Reactive Meteor Data with Custom Store

```js
// stores/meteorStore.js
import { writable } from 'svelte/store';
import { Meteor } from 'meteor/meteor';

export function createMeteorStore(collection, selector = {}, options = {}) {
  const { subscribe, set, update } = writable([]);
  let subscription;

  const meteorSubscribe = (subscriptionName) => {
    subscription = Meteor.subscribe(subscriptionName, {
      onReady() {
        const data = collection.find(selector, options).fetch();
        set(data);
      },
      onError(error) {
        console.error('Meteor subscription error:', error);
      }
    });
  };

  const stop = () => {
    if (subscription) {
      subscription.stop();
    }
  };

  return {
    subscribe,
    meteorSubscribe,
    stop,
    refresh() {
      if (subscription && subscription.ready()) {
        const data = collection.find(selector, options).fetch();
        set(data);
      }
    }
  };
}
```

Usage:

```svelte
<script>
  import { onMount, onDestroy } from 'svelte';
  import { createMeteorStore } from '../stores/meteorStore';
  import { TasksCollection } from '../api/TasksCollection';

  const taskStore = createMeteorStore(TasksCollection);

  onMount(() => {
    taskStore.meteorSubscribe('tasks');
  });

  onDestroy(() => {
    taskStore.stop();
  });
</script>

{#each $taskStore as task (task._id)}
  <div>{task.title}</div>
{/each}
```

### Async Method Calls in Components

```svelte
<!-- AddTask.svelte -->
<script>
  import { Meteor } from 'meteor/meteor';

  let title = '';
  let isSubmitting = false;

  async function handleSubmit() {
    if (!title.trim()) return;
    
    isSubmitting = true;
    
    try {
      await Meteor.callAsync('tasks.insert', title);
      title = '';
    } catch (error) {
      console.error('Error adding task:', error);
    } finally {
      isSubmitting = false;
    }
  }
</script>

<form on:submit|preventDefault={handleSubmit}>
  <input
    bind:value={title}
    type="text"
    placeholder="Task title"
    disabled={isSubmitting}
  />
  <button type="submit" disabled={isSubmitting}>
    {isSubmitting ? 'Adding...' : 'Add Task'}
  </button>
</form>
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
  * `PascalCase` for Svelte components and class names
  * `UPPER_CASE` for constants
* Use optional chaining (`?.`) and nullish coalescing (`??`) where appropriate.

---

## Svelte Component Guidelines

### Component Structure

```svelte
<!-- MyComponent.svelte -->
<script>
  // 1. Imports
  import { onMount, createEventDispatcher } from 'svelte';
  import { writable } from 'svelte/store';
  import { Meteor } from 'meteor/meteor';

  // 2. Props
  export let title = '';
  export let items = [];

  // 3. Event dispatcher
  const dispatch = createEventDispatcher();

  // 4. Local state
  let localState = '';
  let isLoading = false;

  // 5. Reactive statements
  $: filteredItems = items.filter(item => item.active);
  $: hasItems = filteredItems.length > 0;

  // 6. Functions
  async function handleAction() {
    try {
      await Meteor.callAsync('myMethod');
      dispatch('update', { success: true });
    } catch (error) {
      console.error(error);
      dispatch('error', { error });
    }
  }

  // 7. Lifecycle
  onMount(() => {
    // Component mounted logic
  });
</script>

<!-- Template -->
<div class="my-component">
  <h2>{title}</h2>
  
  {#if hasItems}
    {#each filteredItems as item (item.id)}
      <div>{item.name}</div>
    {/each}
  {:else}
    <p>No items found</p>
  {/if}
  
  <button on:click={handleAction}>
    Action
  </button>
</div>

<style>
  .my-component {
    /* Component styles */
  }
</style>
```

### State Management with Custom Stores

```js
// stores/appStore.js
import { writable, derived } from 'svelte/store';
import { Meteor } from 'meteor/meteor';
import { TasksCollection } from '../api/TasksCollection';

// Create stores
export const tasks = writable([]);
export const isLoading = writable(false);
export const error = writable(null);

// Derived stores
export const completedTasks = derived(
  tasks,
  $tasks => $tasks.filter(task => task.completed)
);

export const pendingTasks = derived(
  tasks,
  $tasks => $tasks.filter(task => !task.completed)
);

// Actions
export const taskActions = {
  async loadTasks() {
    isLoading.set(true);
    error.set(null);
    
    try {
      const subscription = Meteor.subscribe('tasks');
      await new Promise(resolve => {
        subscription.ready() ? resolve() : subscription.onReady(resolve);
      });
      
      const taskData = TasksCollection.find({}).fetch();
      tasks.set(taskData);
    } catch (err) {
      error.set(err);
    } finally {
      isLoading.set(false);
    }
  },

  async addTask(title) {
    try {
      await Meteor.callAsync('tasks.insert', title);
      // Refresh tasks
      await this.loadTasks();
    } catch (err) {
      error.set(err);
    }
  },

  async updateTask(taskId, updates) {
    try {
      await Meteor.callAsync('tasks.update', taskId, updates);
      await this.loadTasks();
    } catch (err) {
      error.set(err);
    }
  }
};
```

Usage in component:

```svelte
<script>
  import { onMount } from 'svelte';
  import { tasks, isLoading, taskActions } from '../stores/appStore';

  onMount(() => {
    taskActions.loadTasks();
  });
</script>

{#if $isLoading}
  <div>Loading...</div>
{:else}
  <ul>
    {#each $tasks as task (task._id)}
      <li>{task.title}</li>
    {/each}
  </ul>
{/if}
```

---

## TypeScript Support (Optional)

For TypeScript projects, add proper type definitions:

```svelte
<!-- TaskList.svelte -->
<script lang="ts">
  import { onMount } from 'svelte';
  import type { Writable } from 'svelte/store';
  import { writable } from 'svelte/store';
  import { Meteor } from 'meteor/meteor';

  interface Task {
    _id: string;
    title: string;
    completed: boolean;
    createdAt: Date;
    userId: string;
  }

  interface Props {
    showCompleted?: boolean;
  }

  export let showCompleted: boolean = true;

  const tasks: Writable<Task[]> = writable([]);
  const isLoading: Writable<boolean> = writable(true);

  async function loadTasks(): Promise<void> {
    try {
      const subscription = Meteor.subscribe('tasks');
      await new Promise<void>(resolve => {
        subscription.ready() ? resolve() : subscription.onReady(resolve);
      });
      
      const taskData = TasksCollection.find({}).fetch() as Task[];
      tasks.set(taskData);
    } catch (error) {
      console.error('Error loading tasks:', error);
    } finally {
      isLoading.set(false);
    }
  }

  onMount(() => {
    loadTasks();
  });
</script>

{#if $isLoading}
  <div>Loading...</div>
{:else}
  <ul>
    {#each $tasks as task (task._id)}
      <li>{task.title}</li>
    {/each}
  </ul>
{/if}
```

---

## Testing with Svelte

### Component Testing

```js
import { render, fireEvent, waitFor } from '@testing-library/svelte';
import MyComponent from './MyComponent.svelte';

// Mock Meteor
jest.mock('meteor/meteor', () => ({
  Meteor: {
    callAsync: jest.fn(),
    subscribe: jest.fn(() => ({ 
      ready: () => true,
      onReady: jest.fn()
    }))
  }
}));

describe('MyComponent', () => {
  test('should handle form submission', async () => {
    const mockCallAsync = jest.fn().mockResolvedValue(undefined);
    Meteor.callAsync = mockCallAsync;
    
    const { getByRole } = render(MyComponent);
    
    const input = getByRole('textbox');
    const button = getByRole('button');
    
    await fireEvent.input(input, { target: { value: 'Test task' } });
    await fireEvent.click(button);
    
    await waitFor(() => {
      expect(mockCallAsync).toHaveBeenCalledWith('tasks.insert', 'Test task');
    });
  });

  test('should handle store updates', async () => {
    const { component } = render(MyComponent);
    
    // Test reactive updates
    component.$set({ items: [{ id: 1, name: 'Test' }] });
    
    await waitFor(() => {
      expect(document.body).toContainHTML('Test');
    });
  });
});
```

---

## Common Patterns

### Loading States

```svelte
<script>
  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';
  import { Meteor } from 'meteor/meteor';

  const data = writable([]);
  const isLoading = writable(true);
  const error = writable(null);

  onMount(async () => {
    try {
      const subscription = Meteor.subscribe('myData');
      await new Promise(resolve => {
        subscription.ready() ? resolve() : subscription.onReady(resolve);
      });
      
      data.set(MyCollection.find({}).fetch());
    } catch (err) {
      error.set(err);
    } finally {
      isLoading.set(false);
    }
  });
</script>

{#if $error}
  <div class="error">Error: {$error.message}</div>
{:else if $isLoading}
  <div class="loading">Loading...</div>
{:else if $data.length === 0}
  <div class="empty">No data found</div>
{:else}
  {#each $data as item (item._id)}
    <div>{item.name}</div>
  {/each}
{/if}
```

### Form Handling with Validation

```svelte
<script>
  import { Meteor } from 'meteor/meteor';

  let form = {
    name: '',
    email: ''
  };

  let errors = {};
  let isSubmitting = false;

  function validateForm() {
    errors = {};
    
    if (!form.name.trim()) {
      errors.name = 'Name is required';
    }
    
    if (!form.email.trim()) {
      errors.email = 'Email is required';
    } else if (!/\S+@\S+\.\S+/.test(form.email)) {
      errors.email = 'Email is invalid';
    }
    
    return Object.keys(errors).length === 0;
  }

  async function handleSubmit() {
    if (!validateForm()) return;
    
    isSubmitting = true;
    
    try {
      await Meteor.callAsync('users.create', form);
      // Reset form
      form = { name: '', email: '' };
      errors = {};
    } catch (error) {
      console.error('Form submission error:', error);
    } finally {
      isSubmitting = false;
    }
  }
</script>

<form on:submit|preventDefault={handleSubmit}>
  <div class="form-group">
    <label for="name">Name:</label>
    <input
      id="name"
      bind:value={form.name}
      type="text"
      class:error={errors.name}
    />
    {#if errors.name}
      <span class="error-message">{errors.name}</span>
    {/if}
  </div>

  <div class="form-group">
    <label for="email">Email:</label>
    <input
      id="email"
      bind:value={form.email}
      type="email"
      class:error={errors.email}
    />
    {#if errors.email}
      <span class="error-message">{errors.email}</span>
    {/if}
  </div>

  <button type="submit" disabled={isSubmitting}>
    {isSubmitting ? 'Submitting...' : 'Submit'}
  </button>
</form>

<style>
  .error {
    border-color: red;
  }
  
  .error-message {
    color: red;
    font-size: 0.8em;
  }
</style>
```

### Reactive Statements for Complex Logic

```svelte
<script>
  import { writable } from 'svelte/store';

  let searchTerm = '';
  let items = [];
  let sortBy = 'name';
  let sortOrder = 'asc';

  // Reactive statements for filtering and sorting
  $: filteredItems = items.filter(item => 
    item.name.toLowerCase().includes(searchTerm.toLowerCase())
  );

  $: sortedItems = [...filteredItems].sort((a, b) => {
    const valueA = a[sortBy];
    const valueB = b[sortBy];
    
    if (sortOrder === 'asc') {
      return valueA > valueB ? 1 : -1;
    } else {
      return valueA < valueB ? 1 : -1;
    }
  });

  $: itemCount = sortedItems.length;
  $: hasResults = itemCount > 0;
</script>

<input bind:value={searchTerm} placeholder="Search..." />

<select bind:value={sortBy}>
  <option value="name">Sort by Name</option>
  <option value="date">Sort by Date</option>
</select>

<select bind:value={sortOrder}>
  <option value="asc">Ascending</option>
  <option value="desc">Descending</option>
</select>

<p>Found {itemCount} results</p>

{#if hasResults}
  {#each sortedItems as item (item._id)}
    <div>{item.name}</div>
  {/each}
{:else}
  <p>No items match your search</p>
{/if}
```