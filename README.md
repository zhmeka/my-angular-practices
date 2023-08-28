# Angular | Хорошие практики

## Использование функции *inject()* для внедрения зависисимостей в компонент
Код выглядит более декларативно и чисто. Работать с наследованием становится легче.

**Конструктор**

```ts
export class AppComponent {
  constructor(
    @Inject(FOO) private foo: string,
    @Optional() @Inject(BAR) private bar: string | null,
    private http: HttpClient,
    private todosService: TodosService
  ) { }
}
```

**Функция Inject()**

```ts
export class AppComponent {
  private foo = inject(FOO)
  private bar = inject(BAR, { optional: true })
  private http = inject(HttpClient);
  private todosService = inject(TodosService);
}
```
------------

**Внедрение зависимостей в конструкторе и наследование**
```ts
export class Foo {
  constructor(private readonly userService: UserService) {}
}

export class Bar extends Foo {
  constructor(private readonly userService: UserService) {
    super(userService)
  }
}
```

**Внедрение зависимостей с использованием функции Inject() и наследование**

```ts
export class Foo {
  private  userService = inject(UserService);
}

export class Bar extends Foo {
  constructor() {
    super()
  }
}
```

## Использование пайпа async везде, где это возможно
Этот подход значительно сокращает количество кода и повышает читаемость. Так же отпадает нужда в контроле отписок, лишних состояниях и заботе о change detection.

**Явное использование subscribe()**

```ts
@Component({
  selector: 'app-form',
  template: `<div *ngFor="let filter of filters">{{ filter }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FilterListComponent implements OnInit, OnDestroy {
  private readonly destroy$ = new Subject<void>();

  public filters: IFilter[] = [];

   constructor(
    private readonly filterService: FilterService,
    private readonly changeDetectorRef: ChangeDetectorRef
  ) {}

  public ngOnInit(): void {
    this.filterService.getFilters()
    .pipe(takeUntil(this._destroy$))
    .subscribe((filters: IFilter[]): void => {
      this.filters = filters;
      this.changeDetectorRef.detectChanges();
    })
  }

  public ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Пайп async**

```ts
@Component({
  selector: 'app-form',
  template: `<div *ngFor="let filter of filters$ | async">{{ filter }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class FilterListComponent {
  public filters$ = inject(FilterService).getFilters();
}
```

## Использование *Template Driven Forms* вместо *Reactive Forms*
 Единственное преимущество *Reactive Forms*, по моему мнению, заключается в более удобном юнит-тестировании. В остальном формы *Reactive Forms* представляют собой огромное количество лишнего кода и ненужных усложнений. Под капотом оба эти подхода к формам имеют один и тот же API, но *Template Driven Forms* гораздо проще использовать и поддерживать.

**Reactive Forms**

```ts
@Component({
  selector: 'app-form',
  template: `<form [formGroup]="form" (ngSubmit)="sumbit()">
    <input type="text" formControlName="name" />
    <input type="email" formControlName="email" />
    <input type="checkbox" formControlName="hasPhone" />
    <input type="tel" formControlName="phone" />

    <button type="submit" [disabled]="form.invalid">Submit</button>
  </form>`,
})
export class FormComponent implements OnInit, OnDestroy {
  private readonly destroy$ = new Subject<void>();

  public form: FormGroup = this.fb.group({
    name: ['', Validators.required],
    email: ['', Validators.required, Validators.email],
    hasPhone: [false],
    phone: [''],
  });

  constructor(private readonly fb: FormBuilder) {}

  public ngOnInit(): void {
    this.form
      .get('hasPhone')
      .valueChanges.pipe(takeUntil(this._destroy$), startWith(this.form.get('hasPhone').value))
      .subscribe((hasPhone: boolean): void => {
        const phoneControl: AbstractControl = this.form.get('phone');

        hasPhone ? phoneControl.enable() : phoneControl.disable();
      });
  }

  public ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  public submit(): void {
    console.log(this.form.value);
  }
}
```

**Template Driven Forms**
```ts
@Component({
  selector: 'app-form',
  template: `<form #form="ngForm" (ngSubmit)="sumbit(form.value)">
    <input type="text" ngModel name="name" required />
    <input type="email" ngModel name="email" required email />
    <input type="checkbox" ngModel name="hasPhone" #hasPhoneControl="ngModel" />
    <input type="tel" ngModel name="phone" [disabled]="hasPhoneControl.value" />

    <button type="submit" [disabled]="form.invalid">Submit</button>
  </form>`,
})
export class FormComponent {
  public submit(value: any): void {
    console.log(value);
  }
}
```

## Инжектирование констант в компоненты
Помимо очевидного преймщуства с тестированием, этот подход так же является более гибким.

**Константа**

```ts
// Файл с константой
export const COUNTRIES: ICountry[] = [];

// Файл компонента
private readonly _countries: ICountry[] = COUNTRIES;
```

**Injection Token**

```ts
// Файл с токеном
export const COUNTRIES = new InjectionToken('COUNTRIES', {
  factory: (): ICountry[] => [],
});

// Файл компонента
private readonly _countries: ICountry[] = inject(COUNTRIES);
```

## Использование standalone-компонентов и разбивание всех фич приложения на  независимые друг от друга модули
Для каждой фичи нужен свой отдельный модуль.

## Отказ от использования методов для получения значений в шаблонах
Это сильно может сказаться на производительности на страницах с большой интерактивностью или тяжелыми вычислительными операциями т.к. проверка обнаружения изменений будет вызывать эти методы каждый раз при любом действии пользователя.

Альтернативы: pure-пайпы, вычисление значений напрямую, декомпозиция больших компонентов и т.д..


**Вызов метода в шаблоне**

```html
<div *ngFor="let filter of filters">
  <div>{{ getFilterValue(filter) }}</div>
</div>
```

**Использование pure-pipe**
```html
<div *ngFor="let filter of filters">
  <div>{{ filter | filterValue }}</div>
</div>
```
