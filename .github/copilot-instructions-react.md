# Internal Developer Guide for Meteor 3.x + React Apps

This documentation provides guidance for working on Meteor-based applications using MeteorJS version 3.x and React as the front-end rendering framework.

---

## Project Basics

* JavaScript dependencies are managed using **npm**.
* The codebase uses **MeteorJS (v3.2.2 or higher)**: [https://www.meteor.com/](https://www.meteor.com/)
* React is used for rendering UI components: [https://react.dev/](https://react.dev/)
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

## React + Meteor Integration

### Data Subscriptions

Use `useTracker` hook for reactive data subscriptions:

```jsx
import { useTracker } from 'meteor/react-meteor-data';
import { TasksCollection } from '../api/TasksCollection';

function TaskList() {
  const { tasks, isLoading } = useTracker(() => {
    const subscription = Meteor.subscribe('tasks');
    return {
      tasks: TasksCollection.find({}).fetch(),
      isLoading: !subscription.ready()
    };
  }, []);

  if (isLoading) return <div>Loading...</div>;

  return (
    <ul>
      {tasks.map(task => (
        <li key={task._id}>{task.title}</li>
      ))}
    </ul>
  );
}
```

### Async Method Calls in Components

```jsx
import { useState } from 'react';

function AddTask() {
  const [title, setTitle] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    
    try {
      await Meteor.callAsync('tasks.insert', title);
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
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        disabled={isSubmitting}
      />
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Adding...' : 'Add Task'}
      </button>
    </form>
  );
}
```

---

## JavaScript Code Style

Please follow the conventions below to maintain consistency:

* **Strings**: Use single quotes `'like this'`.
* **Indentation**: Use 2 spaces.
* **Variables**: Use `const` and `let`; avoid `var`.
* **String building**: Use template literals `` `Hello ${name}` ``.
* **Functions**: Prefer regular function declarations over arrow functions for components.
* **Asynchronous Code**: Use `async`/`await`.
* **Modules**: Use `import`/`export` (ES6 modules); avoid CommonJS.
* **Naming**:
  * `camelCase` for variables/functions
  * `PascalCase` for React components and class names
  * `UPPER_CASE` for constants
* Use optional chaining (`?.`) and nullish coalescing (`??`) where appropriate.

---

## React Component Guidelines

### Component Structure

```jsx
import React, { useState, useEffect } from 'react';
import { useTracker } from 'meteor/react-meteor-data';

function MyComponent({ propOne, propTwo }) {
  // 1. Hooks first
  const [localState, setLocalState] = useState('');
  
  // 2. useTracker for Meteor data
  const { data, isLoading } = useTracker(() => {
    // subscription logic
  }, []);

  // 3. useEffect for side effects
  useEffect(() => {
    // side effect logic
  }, []);

  // 4. Event handlers
  const handleClick = async () => {
    try {
      await Meteor.callAsync('myMethod');
    } catch (error) {
      console.error(error);
    }
  };

  // 5. Render
  return (
    <div>
      {/* JSX content */}
    </div>
  );
}

export default MyComponent;
```

### State Management

**Local State:**
```jsx
const [count, setCount] = useState(0);
```

**Global State (for complex apps):**
Consider using React Context or a state management library like Zustand or Redux Toolkit.

### Error Boundaries

```jsx
import React from 'react';

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children;
  }
}
```

---

## TypeScript Support (Optional)

For TypeScript projects, add proper type definitions:

```tsx
interface Task {
  _id: string;
  title: string;
  completed: boolean;
  createdAt: Date;
  userId: string;
}

interface TaskListProps {
  showCompleted?: boolean;
}

function TaskList({ showCompleted = true }: TaskListProps) {
  const { tasks, isLoading } = useTracker((): {
    tasks: Task[];
    isLoading: boolean;
  } => {
    const subscription = Meteor.subscribe('tasks');
    return {
      tasks: TasksCollection.find({}).fetch() as Task[],
      isLoading: !subscription.ready()
    };
  }, []);

  // Component logic...
}
```

---

## Testing with React

### Component Testing

```jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { Meteor } from 'meteor/meteor';
import MyComponent from './MyComponent';

// Mock Meteor methods
jest.mock('meteor/meteor', () => ({
  Meteor: {
    callAsync: jest.fn(),
    subscribe: jest.fn(() => ({ ready: () => true }))
  }
}));

test('should handle form submission', async () => {
  Meteor.callAsync.mockResolvedValue(undefined);
  
  render(<MyComponent />);
  
  const input = screen.getByRole('textbox');
  const button = screen.getByRole('button');
  
  fireEvent.change(input, { target: { value: 'Test task' } });
  fireEvent.click(button);
  
  await waitFor(() => {
    expect(Meteor.callAsync).toHaveBeenCalledWith('tasks.insert', 'Test task');
  });
});
```

---

## Common Patterns

### Loading States

```jsx
function DataComponent() {
  const { data, isLoading, error } = useTracker(() => {
    const subscription = Meteor.subscribe('myData');
    return {
      data: MyCollection.find({}).fetch(),
      isLoading: !subscription.ready(),
      error: subscription.error
    };
  }, []);

  if (error) return <div>Error: {error.message}</div>;
  if (isLoading) return <div>Loading...</div>;
  if (!data.length) return <div>No data found</div>;

  return (
    <div>
      {data.map(item => (
        <div key={item._id}>{item.name}</div>
      ))}
    </div>
  );
}
```

### Form Handling

```jsx
function UserForm() {
  const [formData, setFormData] = useState({
    name: '',
    email: ''
  });

  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await Meteor.callAsync('users.create', formData);
      setFormData({ name: '', email: '' });
    } catch (error) {
      console.error('Form submission error:', error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="name"
        value={formData.name}
        onChange={handleChange}
        placeholder="Name"
      />
      <input
        name="email"
        value={formData.email}
        onChange={handleChange}
        placeholder="Email"
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```