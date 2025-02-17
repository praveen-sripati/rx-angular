## Resources

**Example applications:**
A demo application is available on [GitHub](https://github.com/BioPhoton/rx-angular-state-actions).

## Motivation

Signals, or a more commonly used name actions, are a common part of state management and reactive systems in general.
Even if `@rx-angular/state` provides `set` method, sometimes you need to add behaviour to your user input or incoming events.

Subjects are normally used to implement this feature. This leads, especially in bigger applications, to a messy code that is bloated with Subjects.

Let's have a look at this piece of code:

```typescript
@Component({
  template: `
    <input (input)="searchInput($event?.target?.value)" /> Search for:
    {{ search$ | async }}<br />
    <button (click)="submitBtn()">
      Submit<button>
        <br />
        <ul>
          <li *ngFor="let item of list$ | async as list">{{ item }}</li>
        </ul>
      </button>
    </button>
  `,
})
class Component {
  private _submitBtn = new Subject<void>();
  private _searchInput = new Subject<string>();

  set submitBtn() {
    _submitBtn.next();
  }
  get submitBtn$() {
    return _searchInput.asObservable();
  }
  set search(search: string) {
    _searchInput.next(search);
  }
  get search$() {
    return _searchInput.asObservable();
  }

  list$ = this.submitBtn$.pipe(
    withLatestFrom(this.search$),
    switchMap(([_, searchString]) => this.api.query(searchString))
  );

  constructor(api: API) {}
}
```

Just look at the amount of code we need to write for only 2 observable ui events.

Downsides:

- Boilerplate for setter and getter
- Transformations spreaded over class and template
- Manual typing ov subjects and streams
- No central place for UI trigger

Imagine we could have configurable functions that return all UI logic typed under one object.

```typescript
import { RxActionFactory } from '@rx-angular/state/actions';
interface UiActions {
  submitBtn: void;
  searchInput: string;
}

@Component({
  template: `
    <input (input)="ui.searchInput($event)" /> Search for:
    {{ ui.search$ | async }}<br />
    <button (click)="ui.submitBtn()">
      Submit<button>
        <br />
        <ul>
          <li *ngFor="let item of list$ | async as list">{{ item }}</li>
        </ul>
      </button>
    </button>
  `,
  providers: [RxActionFactory],
})
class Component {
  ui = this.factory.create({ searchInput: (e) => e.target.value });

  list$ = this.ui.submitBtn$.pipe(
    withLatestFrom(this.ui.search$),
    switchMap(([_, searchString]) => this.api.query(searchString))
  );

  constructor(api: API, private factory: RxActionFactory<UiActions>) {}
}
```

## RxAngular Signals

This package helps to reduce code used to create composable action streams.
It mostly is used in combination with state management libs to handle user interaction and backend communication.

### Setup

The coalescing features can be used directly from the `cdk` package or indirectly through the `state` package.
To do so, install the `cdk` package and, if needed, the packages depending on it:

1. Install `@rx-angular/state`

```bash
npm i @rx-angular/state
# or
yarn add @rx-angular/state
```

### Basic usage

By using RxAngular Actions we can reduce the boilerplate significantly, to do so we can start by thinking about the specific section their events and event payload types:

```typescript
interface Commands {
  refreshUser: string | number;
  refreshList: string | number;
  refreshGenres: string | number;
}
```

Next we can use the typing to create the action object:

```typescript
commands = getActions<Commands>();
```

The object can now be used to emit signals over setters:

```typescript
commands.refreshUser(value);
commands.refreshList(value);
commands.refreshGenres(value);
```

The emitted signals can be received over observable properties:

```typescript
refreshUser$ = commands.refreshUser$;
refreshList$ = commands.refreshList$;
refreshGenres$ = commands.refreshGenres$;
```

You can also emit multiple signals at once:

```typescript
commands({ refreshUser: true, refreshList: true });
```

If there is the need to make a combined signal you can also select multiple signals and get their emissions in one stream:

```typescript
refreshUserOrList$ = commands.$(['refreshUser', 'refreshList']);
```

### Signals in components

In components/templates we can use signals to map user interaction as well as programmatic to effects or state changes.
This reduces the component class code as well as template.

In addition, we can use it as a shorthand in the template and directly connect to action dispatching in the class.

```typescript
interface UiActions {
  submitBtn: void;
  searchInput: string;
}

@Component({
  template: `
    <input (input)="ui.searchInput($event)" /> Search for:
    {{ ui.search$ | async }}<br />
    <button (click)="ui.submitBtn()">
      Submit<button>
        <br />
        <ul>
          <li *ngFor="let item of list$ | async as list">{{ item }}</li>
        </ul>
      </button>
    </button>
  `,
  providers: [RxState, RxActionFactory],
})
class Component {
  ui = this.factory.create({ searchInput: (e) => e?.target?.value });
  list$ = this.state.select('list');
  submittedSearchQuery$ = this.ui.submitBtn$.pipe(
    withLatestFrom(this.ui.search$),
    map(([_, search]) => search),
    debounceTime(1500)
  );

  constructor(
    private state: RxState<State>,
    private factory: RxActionFactory<UiActions>,
    globalState: StateService
  ) {
    super();
    this.connect('list', this.globalState.refreshGenres$);

    this.state.hold(this.submittedSearchQuery$, this.globalState.refreshGenres);
    // Optional reactively:
    // this.globalState.connectRefreshGenres(this.submittedSearchQuery$);
  }
}
```

#### Using transforms

Often we process `Events` from the template and occasionally also trigger those channels in the class programmatically.

This leads to a cluttered codebase as we have to consider first the value in the event which leads to un necessary and repetitive code in the template.
This is also true for the programmatic usage in the component class or a service.

To ease this pain we can manage this login with `transforms`.

You can write you own transforms or use the predefined functions:

- preventDefault
- stopPropagation
- preventDefaultStopPropagation
- eventValue

```typescript
interface UiActions {
  submitBtn: void;
  searchInput: string;
}

@Component({
  template: `
    <input (input)="ui.searchInput($event)" /> Search for:
    {{ ui.search$ | async }}<br />
    <button (click)="ui.submitBtn()">
      Submit<button>
        <br />
        <ul>
          <li *ngFor="let item of list$ | async as list">{{ item }}</li>
        </ul>
      </button>
    </button>
  `,
  providers: [RxState, RxActionFactory],
})
class Component {
  //                                (e) => e.target ? e.target.value : e
  ui = this.factory.create({ searchInput: eventValue });
  list$ = this.state.select('list');
  submittedSearchQuery$ = this.ui.submitBtn$.pipe(
    withLatestFrom(this.ui.search$),
    map(([_, search]) => search),
    debounceTime(1500)
  );

  constructor(
    private state: RxState<State>,
    private factory: RxActionFactory<UiActions>,
    globalState: StateService
  ) {
    super();
    this.connect('list', this.globalState.refreshGenres$);

    this.state.hold(this.submittedSearchQuery$, this.globalState.refreshGenres);
    // Optional reactively:
    // this.globalState.connectRefreshGenres(this.submittedSearchQuery$);
  }
}
```

### Usage for global services

In services, it comes in handy to have a minimal typed action system.
This helps to have them composable for further optimizations.
Furthermore, we can still expose setters to trigger actions the imperative way.

```typescript
interface State {
  genres: MovieGenreModel[];
}

interface Commands {
  refreshGenres: string | number;
}

@Injectable({
  providedIn: 'root',
})
export class StateService extends RxState<State> {
  private commands = new RxActionFactory<Commands>().create();

  genres$ = this.select('genres');

  constructor(private tmdb2Service: Tmdb2Service) {
    super();

    this.connect(
      'genres',
      this.commands.fetchGenres$.pipe(exhaustMap(this.tmdb2Service.getGenres))
    );
  }

  refreshGenres(genre: string): void {
    this.commands.fetchGenres(genre);
  }

  // Optionally the reactive way
  connectRefreshGenres(genre$: Observable<string>): void {
    this.connect(
      'genres',
      this.commands.fetchGenres$.pipe(exhaustMap(this.tmdb2Service.getGenres))
    );
  }
}
```
