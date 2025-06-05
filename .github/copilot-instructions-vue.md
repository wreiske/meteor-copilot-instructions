# Internal Developer Guide for Meteor 3.x + Vue Apps

This documentation provides guidance for working on Meteor-based applications using MeteorJS version 3.x and Vue.js as the front-end rendering framework.

---

## Project Basics

* JavaScript dependencies are managed using **npm**.
* The codebase uses **MeteorJS (v3.2.2 or higher)**: [https://www.meteor.com/](https://www.meteor.com/)
* Vue.js is used for rendering UI components: [https://vuejs.org/](https://vuejs.org/)
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

## Vue + Meteor Integration

### Data Subscriptions with Composition API

```vue
<template>
  <div>
    <div v-if="isLoading">Loading...</div>
    <ul v-else>
      <li v-for="task in tasks" :key="task._id">
        {{ task.title }}
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import { Meteor } from 'meteor/meteor';
import { TasksCollection } from '../api/TasksCollection';

const tasks = ref([]);
const isLoading = ref(true);
let subscription = null;

const loadTasks = () => {
  subscription = Meteor.subscribe('tasks', {
    onReady() {
      tasks.value = TasksCollection.find({}).fetch();
      isLoading.value = false;
    },
    onError(error) {
      console.error('Subscription error:', error);
      isLoading.value = false;
    }
  });
};

onMounted(() => {
  loadTasks();
});

onUnmounted(() => {
  if (subscription) {
    subscription.stop();
  }
});
</script>
```

### Reactive Data with Vue Tracker

```vue
<template>
  <div>
    <div v-if="isLoading">Loading...</div>
    <ul v-else>
      <li v-for="task in tasks" :key="task._id">
        {{ task.title }}
      </li>
    </ul>
  </div>
</template>

<script>
import { defineComponent } from 'vue';
import { Meteor } from 'meteor/meteor';
import { TasksCollection } from '../api/TasksCollection';

export default defineComponent({
  name: 'TaskList',
  meteor: {
    data() {
      return {
        tasks: () => TasksCollection.find({}).fetch(),
        isLoading: () => !Meteor.subscribe('tasks').ready()
      };
    }
  }
});
</script>
```

### Async Method Calls in Components

```vue
<template>
  <form @submit.prevent="handleSubmit">
    <input
      v-model="title"
      type="text"
      placeholder="Task title"
      :disabled="isSubmitting"
    />
    <button type="submit" :disabled="isSubmitting">
      {{ isSubmitting ? 'Adding...' : 'Add Task' }}
    </button>
  </form>
</template>

<script setup>
import { ref } from 'vue';
import { Meteor } from 'meteor/meteor';

const title = ref('');
const isSubmitting = ref(false);

const handleSubmit = async () => {
  if (!title.value.trim()) return;
  
  isSubmitting.value = true;
  
  try {
    await Meteor.callAsync('tasks.insert', title.value);
    title.value = '';
  } catch (error) {
    console.error('Error adding task:', error);
  } finally {
    isSubmitting.value = false;
  }
};
</script>
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
  * `PascalCase` for Vue components and class names
  * `UPPER_CASE` for constants
* Use optional chaining (`?.`) and nullish coalescing (`??`) where appropriate.

---

## Vue Component Guidelines

### Component Structure (Composition API)

```vue
<template>
  <div class="my-component">
    <!-- Template content -->
  </div>
</template>

<script setup>
// 1. Imports
import { ref, computed, onMounted } from 'vue';
import { Meteor } from 'meteor/meteor';

// 2. Props
const props = defineProps({
  title: String,
  items: Array
});

// 3. Emits
const emit = defineEmits(['update', 'delete']);

// 4. Reactive state
const localState = ref('');
const isLoading = ref(false);

// 5. Computed properties
const filteredItems = computed(() => {
  return props.items?.filter(item => item.active) || [];
});

// 6. Methods
const handleAction = async () => {
  try {
    await Meteor.callAsync('myMethod');
    emit('update');
  } catch (error) {
    console.error(error);
  }
};

// 7. Lifecycle hooks
onMounted(() => {
  // Component mounted logic
});
</script>

<style scoped>
.my-component {
  /* Component styles */
}
</style>
```

### State Management with Pinia (recommended)

```js
// stores/tasks.js
import { defineStore } from 'pinia';
import { Meteor } from 'meteor/meteor';
import { TasksCollection } from '../api/TasksCollection';

export const useTaskStore = defineStore('tasks', {
  state: () => ({
    tasks: [],
    isLoading: false,
    error: null
  }),

  getters: {
    completedTasks: (state) => state.tasks.filter(task => task.completed),
    pendingTasks: (state) => state.tasks.filter(task => !task.completed)
  },

  actions: {
    async loadTasks() {
      this.isLoading = true;
      try {
        const subscription = Meteor.subscribe('tasks');
        await new Promise(resolve => {
          subscription.ready() ? resolve() : subscription.onReady(resolve);
        });
        this.tasks = TasksCollection.find({}).fetch();
      } catch (error) {
        this.error = error;
      } finally {
        this.isLoading = false;
      }
    },

    async addTask(title) {
      try {
        await Meteor.callAsync('tasks.insert', title);
        await this.loadTasks(); // Refresh tasks
      } catch (error) {
        this.error = error;
      }
    }
  }
});
```

Usage in component:

```vue
<template>
  <div>
    <div v-if="taskStore.isLoading">Loading...</div>
    <ul v-else>
      <li v-for="task in taskStore.tasks" :key="task._id">
        {{ task.title }}
      </li>
    </ul>
  </div>
</template>

<script setup>
import { onMounted } from 'vue';
import { useTaskStore } from '../stores/tasks';

const taskStore = useTaskStore();

onMounted(() => {
  taskStore.loadTasks();
});
</script>
```

---

## TypeScript Support (Optional)

For TypeScript projects, add proper type definitions:

```vue
<template>
  <div>
    <ul>
      <li v-for="task in tasks" :key="task._id">
        {{ task.title }}
      </li>
    </ul>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
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

const props = withDefaults(defineProps<Props>(), {
  showCompleted: true
});

const tasks = ref<Task[]>([]);
const isLoading = ref<boolean>(true);

const loadTasks = async (): Promise<void> => {
  try {
    const subscription = Meteor.subscribe('tasks');
    await new Promise<void>(resolve => {
      subscription.ready() ? resolve() : subscription.onReady(resolve);
    });
    tasks.value = TasksCollection.find({}).fetch() as Task[];
  } catch (error) {
    console.error('Error loading tasks:', error);
  } finally {
    isLoading.value = false;
  }
};

onMounted(() => {
  loadTasks();
});
</script>
```

---

## Testing with Vue

### Component Testing

```js
import { mount } from '@vue/test-utils';
import { createPinia, setActivePinia } from 'pinia';
import MyComponent from './MyComponent.vue';

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
  beforeEach(() => {
    setActivePinia(createPinia());
  });

  test('should handle form submission', async () => {
    const mockCallAsync = jest.fn().mockResolvedValue(undefined);
    Meteor.callAsync = mockCallAsync;
    
    const wrapper = mount(MyComponent);
    
    await wrapper.find('input').setValue('Test task');
    await wrapper.find('button').trigger('click');
    
    expect(mockCallAsync).toHaveBeenCalledWith('tasks.insert', 'Test task');
  });
});
```

---

## Common Patterns

### Loading States

```vue
<template>
  <div>
    <div v-if="error" class="error">
      Error: {{ error.message }}
    </div>
    <div v-else-if="isLoading" class="loading">
      Loading...
    </div>
    <div v-else-if="!data.length" class="empty">
      No data found
    </div>
    <div v-else>
      <div v-for="item in data" :key="item._id">
        {{ item.name }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { Meteor } from 'meteor/meteor';

const data = ref([]);
const isLoading = ref(true);
const error = ref(null);

onMounted(async () => {
  try {
    const subscription = Meteor.subscribe('myData');
    await new Promise(resolve => {
      subscription.ready() ? resolve() : subscription.onReady(resolve);
    });
    data.value = MyCollection.find({}).fetch();
  } catch (err) {
    error.value = err;
  } finally {
    isLoading.value = false;
  }
});
</script>
```

### Form Handling

```vue
<template>
  <form @submit.prevent="handleSubmit">
    <div class="form-group">
      <label for="name">Name:</label>
      <input
        id="name"
        v-model="form.name"
        type="text"
        required
      />
    </div>
    <div class="form-group">
      <label for="email">Email:</label>
      <input
        id="email"
        v-model="form.email"
        type="email"
        required
      />
    </div>
    <button type="submit" :disabled="isSubmitting">
      {{ isSubmitting ? 'Submitting...' : 'Submit' }}
    </button>
  </form>
</template>

<script setup>
import { reactive, ref } from 'vue';
import { Meteor } from 'meteor/meteor';

const form = reactive({
  name: '',
  email: ''
});

const isSubmitting = ref(false);

const handleSubmit = async () => {
  isSubmitting.value = true;
  
  try {
    await Meteor.callAsync('users.create', form);
    // Reset form
    Object.assign(form, { name: '', email: '' });
  } catch (error) {
    console.error('Form submission error:', error);
  } finally {
    isSubmitting.value = false;
  }
};
</script>
```

### Watchers for Reactive Updates

```vue
<script setup>
import { ref, watch, onMounted } from 'vue';
import { Meteor } from 'meteor/meteor';

const searchTerm = ref('');
const results = ref([]);

// Watch for search term changes and perform search
watch(searchTerm, async (newTerm) => {
  if (newTerm.length >= 3) {
    try {
      results.value = await Meteor.callAsync('search.perform', newTerm);
    } catch (error) {
      console.error('Search error:', error);
    }
  } else {
    results.value = [];
  }
}, { debounce: 300 }); // Debounce for performance
</script>
```