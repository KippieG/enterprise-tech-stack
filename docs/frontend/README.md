# Frontend — Angular

## Overzicht

Angular is het enterprise frontend framework van Google — gebouwd met TypeScript, opinionated en schaalbaar. Het is de standaardkeuze voor grote teams en complexe bedrijfsapplicaties waar structuur en onderhoud prioriteit hebben boven snelheid van prototyping.

---

## Projectstructuur (Feature-based)

```
src/
├── app/
│   ├── core/                  # Singletons: auth, interceptors, guards
│   │   ├── auth/
│   │   ├── interceptors/
│   │   └── guards/
│   ├── shared/                # Herbruikbare components, pipes, directives
│   │   ├── components/
│   │   ├── pipes/
│   │   └── directives/
│   ├── features/              # Feature modules (lazy loaded)
│   │   ├── orders/
│   │   │   ├── components/
│   │   │   ├── services/
│   │   │   ├── models/
│   │   │   └── orders.routes.ts
│   │   └── warehouse/
│   ├── app.config.ts
│   ├── app.routes.ts
│   └── app.component.ts
└── environments/
    ├── environment.ts
    └── environment.prod.ts
```

---

## Standalone Components (Angular 17+)

```typescript
// order-list.component.ts
@Component({
  selector: 'app-order-list',
  standalone: true,
  imports: [CommonModule, RouterModule, OrderCardComponent],
  template: `
    <div class="order-grid">
      @for (order of orders(); track order.id) {
        <app-order-card [order]="order" (selected)="onSelect($event)" />
      }
      @empty {
        <p>Geen orders gevonden.</p>
      }
    </div>
  `
})
export class OrderListComponent implements OnInit {
  private orderService = inject(OrderService);

  orders = signal<Order[]>([]);

  ngOnInit() {
    this.orderService.getAll().subscribe(orders => this.orders.set(orders));
  }

  onSelect(order: Order) {
    // navigeer naar detail
  }
}
```

---

## Services & Dependency Injection

```typescript
@Injectable({ providedIn: 'root' })
export class OrderService {
  private http = inject(HttpClient);
  private apiUrl = inject(API_BASE_URL);

  getAll(): Observable<Order[]> {
    return this.http.get<Order[]>(`${this.apiUrl}/orders`);
  }

  getById(id: number): Observable<Order> {
    return this.http.get<Order>(`${this.apiUrl}/orders/${id}`);
  }

  create(command: CreateOrderCommand): Observable<number> {
    return this.http.post<number>(`${this.apiUrl}/orders`, command);
  }
}
```

---

## RxJS — Essentiële Operators

```typescript
// Combineer meerdere calls
combineLatest([
  this.orderService.getAll(),
  this.customerService.getAll()
]).pipe(
  map(([orders, customers]) => this.mergeData(orders, customers))
).subscribe(data => this.data.set(data));

// Zoekbalk met debounce
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.orderService.search(term)),
  takeUntilDestroyed(this.destroyRef)
).subscribe(results => this.results.set(results));

// Foutafhandeling
this.orderService.getById(id).pipe(
  catchError(err => {
    this.error.set('Order niet gevonden');
    return EMPTY;
  })
).subscribe(order => this.order.set(order));
```

---

## Routing met Guards

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
  {
    path: 'orders',
    loadChildren: () => import('./features/orders/orders.routes'),
    canActivate: [authGuard]
  },
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.routes'),
    canActivate: [authGuard, adminGuard]
  }
];

// auth.guard.ts
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isLoggedIn()) return true;

  router.navigate(['/login'], { queryParams: { returnUrl: state.url } });
  return false;
};
```

---

## HTTP Interceptor (Auth + Error handling)

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const token = auth.getToken();

  const authReq = token
    ? req.clone({ headers: req.headers.set('Authorization', `Bearer ${token}`) })
    : req;

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        auth.logout();
      }
      return throwError(() => error);
    })
  );
};
```

---

## Reactive Forms

```typescript
@Component({ /* ... */ })
export class CreateOrderFormComponent {
  private fb = inject(FormBuilder);
  private orderService = inject(OrderService);

  form = this.fb.group({
    orderNumber: ['', [Validators.required, Validators.maxLength(50)]],
    customerId: [null as number | null, Validators.required],
    deliveryDate: ['', Validators.required],
    lines: this.fb.array([])
  });

  get lines() {
    return this.form.get('lines') as FormArray;
  }

  addLine() {
    this.lines.push(this.fb.group({
      productCode: ['', Validators.required],
      quantity: [1, [Validators.required, Validators.min(1)]]
    }));
  }

  submit() {
    if (this.form.invalid) return;
    this.orderService.create(this.form.value as CreateOrderCommand).subscribe();
  }
}
```

---

## State Management met Signals

```typescript
// order.store.ts (zonder NgRx, met signals)
@Injectable({ providedIn: 'root' })
export class OrderStore {
  private orderService = inject(OrderService);

  private _orders = signal<Order[]>([]);
  private _loading = signal(false);
  private _error = signal<string | null>(null);

  // Public readonly
  readonly orders = this._orders.asReadonly();
  readonly loading = this._loading.asReadonly();
  readonly error = this._error.asReadonly();

  // Computed
  readonly openOrders = computed(() =>
    this._orders().filter(o => o.status === 'open')
  );

  loadAll() {
    this._loading.set(true);
    this.orderService.getAll().subscribe({
      next: orders => {
        this._orders.set(orders);
        this._loading.set(false);
      },
      error: err => {
        this._error.set(err.message);
        this._loading.set(false);
      }
    });
  }
}
```

---

## Best Practices

- Gebruik **standalone components** en **lazy loading** voor elk feature module
- **Signals** voor lokale UI-state, RxJS voor async data streams
- Nooit logica in templates — gebruik computed signals of pipes
- Gebruik `takeUntilDestroyed(destroyRef)` i.p.v. `ngOnDestroy` + Subject
- **Environment files** voor API base URLs — nooit hardcoden
- Linting via ESLint + `@angular-eslint/recommended`
- E2E testen met **Playwright** (moderne keuze boven Cypress)

---

## Nuttige packages

| Package | Gebruik |
|---------|---------|
| `@angular/material` | UI componenten |
| `@ngrx/store` | Redux-stijl state management |
| `@ngrx/signals` | Signal-based store |
| `ngx-translate` | Internationalisatie |
| `date-fns` | Datum manipulatie |
| `ng-select` | Dropdown met zoeken |
| `primeng` | Rich UI componenten |

---

*[← Terug naar overzicht](../../README.md)*
