# Internal Developer Guide for Meteor 3.x + Angular Apps

This documentation provides guidance for working on Meteor-based applications using MeteorJS version 3.x and Angular as the front-end rendering framework.

---

## Project Basics

* JavaScript dependencies are managed using **npm**.
* The codebase uses **MeteorJS (v3.2.2 or higher)**: [https://www.meteor.com/](https://www.meteor.com/)
* Angular is used for rendering UI components: [https://angular.io/](https://angular.io/)
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

## Angular + Meteor Integration

### Data Subscriptions with Services

```typescript
// tasks.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { Meteor } from 'meteor/meteor';
import { TasksCollection } from '../api/TasksCollection';

export interface Task {
  _id: string;
  title: string;
  completed: boolean;
  createdAt: Date;
  userId: string;
}

@Injectable({
  providedIn: 'root'
})
export class TasksService {
  private tasksSubject = new BehaviorSubject<Task[]>([]);
  private loadingSubject = new BehaviorSubject<boolean>(false);
  private subscription: any;

  public tasks$ = this.tasksSubject.asObservable();
  public loading$ = this.loadingSubject.asObservable();

  constructor() {}

  loadTasks(): void {
    this.loadingSubject.next(true);
    
    this.subscription = Meteor.subscribe('tasks', {
      onReady: () => {
        const tasks = TasksCollection.find({}).fetch() as Task[];
        this.tasksSubject.next(tasks);
        this.loadingSubject.next(false);
      },
      onError: (error: any) => {
        console.error('Subscription error:', error);
        this.loadingSubject.next(false);
      }
    });
  }

  async addTask(title: string): Promise<void> {
    try {
      await Meteor.callAsync('tasks.insert', title);
      this.refreshTasks();
    } catch (error) {
      throw error;
    }
  }

  async updateTask(taskId: string, updates: Partial<Task>): Promise<void> {
    try {
      await Meteor.callAsync('tasks.update', taskId, updates);
      this.refreshTasks();
    } catch (error) {
      throw error;
    }
  }

  private refreshTasks(): void {
    if (this.subscription && this.subscription.ready()) {
      const tasks = TasksCollection.find({}).fetch() as Task[];
      this.tasksSubject.next(tasks);
    }
  }

  destroy(): void {
    if (this.subscription) {
      this.subscription.stop();
    }
  }
}
```

### Component with Meteor Data

```typescript
// task-list.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Observable } from 'rxjs';
import { TasksService, Task } from '../services/tasks.service';

@Component({
  selector: 'app-task-list',
  template: `
    <div class="task-list">
      <div *ngIf="loading$ | async" class="loading">
        Loading...
      </div>
      
      <div *ngIf="!(loading$ | async)">
        <ul>
          <li *ngFor="let task of tasks$ | async; trackBy: trackByTaskId">
            {{ task.title }}
            <button (click)="toggleComplete(task)">
              {{ task.completed ? 'Undo' : 'Complete' }}
            </button>
          </li>
        </ul>
      </div>
    </div>
  `,
  styleUrls: ['./task-list.component.scss']
})
export class TaskListComponent implements OnInit, OnDestroy {
  tasks$: Observable<Task[]>;
  loading$: Observable<boolean>;

  constructor(private tasksService: TasksService) {
    this.tasks$ = this.tasksService.tasks$;
    this.loading$ = this.tasksService.loading$;
  }

  ngOnInit(): void {
    this.tasksService.loadTasks();
  }

  ngOnDestroy(): void {
    this.tasksService.destroy();
  }

  trackByTaskId(index: number, task: Task): string {
    return task._id;
  }

  async toggleComplete(task: Task): Promise<void> {
    try {
      await this.tasksService.updateTask(task._id, {
        completed: !task.completed
      });
    } catch (error) {
      console.error('Error updating task:', error);
    }
  }
}
```

### Form Component with Reactive Forms

```typescript
// add-task.component.ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { TasksService } from '../services/tasks.service';

@Component({
  selector: 'app-add-task',
  template: `
    <form [formGroup]="taskForm" (ngSubmit)="onSubmit()">
      <div class="form-group">
        <input
          type="text"
          formControlName="title"
          placeholder="Task title"
          [disabled]="isSubmitting"
          class="form-control"
          [class.is-invalid]="taskForm.get('title')?.invalid && taskForm.get('title')?.touched"
        />
        <div *ngIf="taskForm.get('title')?.invalid && taskForm.get('title')?.touched" 
             class="invalid-feedback">
          Title is required
        </div>
      </div>
      
      <button type="submit" 
              [disabled]="taskForm.invalid || isSubmitting"
              class="btn btn-primary">
        {{ isSubmitting ? 'Adding...' : 'Add Task' }}
      </button>
    </form>
  `,
  styleUrls: ['./add-task.component.scss']
})
export class AddTaskComponent {
  taskForm: FormGroup;
  isSubmitting = false;

  constructor(
    private fb: FormBuilder,
    private tasksService: TasksService
  ) {
    this.taskForm = this.fb.group({
      title: ['', [Validators.required, Validators.minLength(1)]]
    });
  }

  async onSubmit(): Promise<void> {
    if (this.taskForm.invalid) return;

    this.isSubmitting = true;

    try {
      const title = this.taskForm.get('title')?.value;
      await this.tasksService.addTask(title);
      this.taskForm.reset();
    } catch (error) {
      console.error('Error adding task:', error);
    } finally {
      this.isSubmitting = false;
    }
  }
}
```

---

## TypeScript Code Style

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
  * `PascalCase` for Angular components, services, and class names
  * `UPPER_CASE` for constants
  * `kebab-case` for component selectors and file names
* Use optional chaining (`?.`) and nullish coalescing (`??`) where appropriate.
* Always use explicit types in TypeScript.

---

## Angular Component Guidelines

### Component Structure

```typescript
import { Component, Input, Output, EventEmitter, OnInit, OnDestroy } from '@angular/core';
import { Observable, Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({
  selector: 'app-my-component',
  templateUrl: './my-component.component.html',
  styleUrls: ['./my-component.component.scss']
})
export class MyComponentComponent implements OnInit, OnDestroy {
  // 1. Input properties
  @Input() title: string = '';
  @Input() items: any[] = [];

  // 2. Output events
  @Output() itemSelected = new EventEmitter<any>();
  @Output() itemDeleted = new EventEmitter<string>();

  // 3. Public properties
  isLoading = false;
  error: string | null = null;

  // 4. Private properties
  private destroy$ = new Subject<void>();

  constructor(
    // 5. Dependency injection
    private myService: MyService
  ) {}

  ngOnInit(): void {
    // 6. Initialization logic
    this.loadData();
  }

  ngOnDestroy(): void {
    // 7. Cleanup
    this.destroy$.next();
    this.destroy$.complete();
  }

  // 8. Public methods
  async loadData(): Promise<void> {
    this.isLoading = true;
    this.error = null;

    try {
      // Async operations
      const data = await this.myService.getData();
      // Handle data
    } catch (error) {
      this.error = 'Failed to load data';
      console.error(error);
    } finally {
      this.isLoading = false;
    }
  }

  onItemClick(item: any): void {
    this.itemSelected.emit(item);
  }

  // 9. Private methods
  private handleError(error: any): void {
    console.error('Component error:', error);
  }
}
```

### State Management with Services

```typescript
// app-state.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { Meteor } from 'meteor/meteor';

export interface AppState {
  user: any;
  isLoggedIn: boolean;
  loading: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class AppStateService {
  private stateSubject = new BehaviorSubject<AppState>({
    user: null,
    isLoggedIn: false,
    loading: true
  });

  public state$ = this.stateSubject.asObservable();

  constructor() {
    this.initializeState();
  }

  private async initializeState(): Promise<void> {
    try {
      // Check if user is logged in
      const user = Meteor.user();
      this.stateSubject.next({
        user,
        isLoggedIn: !!user,
        loading: false
      });
    } catch (error) {
      console.error('Error initializing state:', error);
      this.stateSubject.next({
        user: null,
        isLoggedIn: false,
        loading: false
      });
    }
  }

  async login(email: string, password: string): Promise<void> {
    try {
      await Meteor.loginWithPassword(email, password);
      const user = Meteor.user();
      this.stateSubject.next({
        user,
        isLoggedIn: true,
        loading: false
      });
    } catch (error) {
      throw error;
    }
  }

  async logout(): Promise<void> {
    try {
      await Meteor.logout();
      this.stateSubject.next({
        user: null,
        isLoggedIn: false,
        loading: false
      });
    } catch (error) {
      throw error;
    }
  }
}
```

---

## Dependency Injection & Modules

### Feature Module Structure

```typescript
// tasks.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule } from '@angular/forms';

import { TasksRoutingModule } from './tasks-routing.module';
import { TaskListComponent } from './components/task-list.component';
import { AddTaskComponent } from './components/add-task.component';
import { TasksService } from './services/tasks.service';

@NgModule({
  declarations: [
    TaskListComponent,
    AddTaskComponent
  ],
  imports: [
    CommonModule,
    ReactiveFormsModule,
    TasksRoutingModule
  ],
  providers: [
    TasksService
  ]
})
export class TasksModule { }
```

### Guards for Route Protection

```typescript
// auth.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { Meteor } from 'meteor/meteor';
import { AppStateService } from '../services/app-state.service';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {
  constructor(
    private appState: AppStateService,
    private router: Router
  ) {}

  canActivate(): Observable<boolean> {
    return this.appState.state$.pipe(
      map(state => {
        if (state.isLoggedIn) {
          return true;
        } else {
          this.router.navigate(['/login']);
          return false;
        }
      })
    );
  }
}
```

---

## Testing with Angular

### Component Testing

```typescript
// task-list.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { of } from 'rxjs';
import { TaskListComponent } from './task-list.component';
import { TasksService } from '../services/tasks.service';

describe('TaskListComponent', () => {
  let component: TaskListComponent;
  let fixture: ComponentFixture<TaskListComponent>;
  let mockTasksService: jasmine.SpyObj<TasksService>;

  const mockTasks = [
    { _id: '1', title: 'Test Task 1', completed: false, createdAt: new Date(), userId: 'user1' },
    { _id: '2', title: 'Test Task 2', completed: true, createdAt: new Date(), userId: 'user1' }
  ];

  beforeEach(async () => {
    const spy = jasmine.createSpyObj('TasksService', ['loadTasks', 'updateTask', 'destroy'], {
      tasks$: of(mockTasks),
      loading$: of(false)
    });

    await TestBed.configureTestingModule({
      declarations: [TaskListComponent],
      providers: [
        { provide: TasksService, useValue: spy }
      ]
    }).compileComponents();

    mockTasksService = TestBed.inject(TasksService) as jasmine.SpyObj<TasksService>;
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(TaskListComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should load tasks on init', () => {
    expect(mockTasksService.loadTasks).toHaveBeenCalled();
  });

  it('should display tasks', () => {
    const compiled = fixture.nativeElement;
    expect(compiled.textContent).toContain('Test Task 1');
    expect(compiled.textContent).toContain('Test Task 2');
  });

  it('should toggle task completion', async () => {
    mockTasksService.updateTask.and.returnValue(Promise.resolve());
    
    await component.toggleComplete(mockTasks[0]);
    
    expect(mockTasksService.updateTask).toHaveBeenCalledWith('1', { completed: true });
  });
});
```

### Service Testing

```typescript
// tasks.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { TasksService } from './tasks.service';

// Mock Meteor
const mockMeteor = {
  subscribe: jasmine.createSpy('subscribe'),
  callAsync: jasmine.createSpy('callAsync')
};

(global as any).Meteor = mockMeteor;

describe('TasksService', () => {
  let service: TasksService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(TasksService);
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });

  it('should load tasks', () => {
    const mockSubscription = {
      ready: jasmine.createSpy('ready').and.returnValue(true),
      onReady: jasmine.createSpy('onReady'),
      onError: jasmine.createSpy('onError')
    };
    
    mockMeteor.subscribe.and.returnValue(mockSubscription);
    
    service.loadTasks();
    
    expect(mockMeteor.subscribe).toHaveBeenCalledWith('tasks', jasmine.any(Object));
  });

  it('should add task', async () => {
    mockMeteor.callAsync.and.returnValue(Promise.resolve());
    
    await service.addTask('Test Task');
    
    expect(mockMeteor.callAsync).toHaveBeenCalledWith('tasks.insert', 'Test Task');
  });
});
```

---

## Common Patterns

### Error Handling

```typescript
// error-handler.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

export interface AppError {
  message: string;
  type: 'error' | 'warning' | 'info';
  timestamp: Date;
}

@Injectable({
  providedIn: 'root'
})
export class ErrorHandlerService {
  private errorsSubject = new BehaviorSubject<AppError[]>([]);
  public errors$ = this.errorsSubject.asObservable();

  addError(message: string, type: 'error' | 'warning' | 'info' = 'error'): void {
    const error: AppError = {
      message,
      type,
      timestamp: new Date()
    };

    const currentErrors = this.errorsSubject.value;
    this.errorsSubject.next([...currentErrors, error]);

    // Auto-remove after 5 seconds
    setTimeout(() => {
      this.removeError(error);
    }, 5000);
  }

  removeError(errorToRemove: AppError): void {
    const currentErrors = this.errorsSubject.value;
    const filteredErrors = currentErrors.filter(error => error !== errorToRemove);
    this.errorsSubject.next(filteredErrors);
  }

  clearErrors(): void {
    this.errorsSubject.next([]);
  }
}
```

### HTTP Interceptor for Meteor Calls

```typescript
// meteor.interceptor.ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from '@angular/common/http';
import { Observable, from } from 'rxjs';
import { switchMap } from 'rxjs/operators';
import { Meteor } from 'meteor/meteor';

@Injectable()
export class MeteorInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Add Meteor user token to requests if available
    const user = Meteor.user();
    
    if (user && req.url.includes('/api/')) {
      const modifiedReq = req.clone({
        setHeaders: {
          'X-User-Id': user._id,
          'X-Auth-Token': user.services?.resume?.loginTokens?.[0]?.hashedToken || ''
        }
      });
      
      return next.handle(modifiedReq);
    }
    
    return next.handle(req);
  }
}
```