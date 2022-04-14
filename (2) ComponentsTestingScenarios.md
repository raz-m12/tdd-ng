Testing Scenarios
=================

## Component binding:
```typescript
@Component({
  selector: 'app-banner',
  template: '<h1>{{title}}</h1>',
  styles: ['h1 { color: green; font-size: 350%}']
})
export class BannerComponent {
  title = 'Test Tour of Heroes';
}

let component: BannerComponent;
let fixture: ComponentFixture<BannerComponent>;
let h1: HTMLElement;

beforeEach(() => {
  TestBed.configureTestingModule({
      declarations: [ BannerComponent ],
  });
  // createComponent() does not bind data
  fixture = TestBed.createComponent(BannerComponent);
  component = fixture.componentInstance; // BannerComponent test instance
  h1 = fixture.nativeElement.querySelector('h1');
});


// You must tell the TestBed to perform data binding by calling fixture.
// detectChanges(). Only then does the <h1> have the expected title.
it('should display original title after detectChanges()', () => {
  fixture.detectChanges(); // binds data and calls the life cycle hooks.
  expect(h1.textContent).toContain(component.title);
});
```

## Automatic [change detection](https://angular.io/guide/testing-components-scenarios#automatic-change-detection):

```typescript
TestBed.configureTestingModule({
  declarations: [ BannerComponent ],
  providers: [
    { provide: ComponentFixtureAutoDetect, useValue: true }
  ]
});

it('should still see original title after comp.title change', () => {
  const oldTitle = comp.title;
  comp.title = 'Test Title';
  // Displayed title is old because Angular didn't hear the change :(
  expect(h1.textContent).toContain(oldTitle);
});

it('should display updated title after detectChanges', () => {
  comp.title = 'Test Title';
  fixture.detectChanges(); // detect changes explicitly
  expect(h1.textContent).toContain(comp.title);
});
```

> The Angular testing environment does not know that the test changed the component's title. The ComponentFixtureAutoDetect service responds to asynchronous activities such as promise resolution, timers, and DOM events. But a direct, synchronous update of the component property is invisible. The test must call fixture.detectChanges() manually to trigger another cycle of change detection.

## Change an [input value](https://angular.io/guide/testing-components-scenarios#change-an-input-value-with-dispatchevent) with dispatchEvent()

```typescript
it('should convert hero name to Title Case', () => {
  // get the name's input and display elements from the DOM
  const hostElement: HTMLElement = fixture.nativeElement;
  const nameInput: HTMLInputElement = hostElement.querySelector('input')!;
  const nameDisplay: HTMLElement = hostElement.querySelector('span')!;

  // simulate user entering a new name into the input box
  nameInput.value = 'quick BROWN  fOx';

  // Dispatch a DOM event so that Angular learns of input value change.
  nameInput.dispatchEvent(new Event('input'));

  // Tell Angular to update the display binding through the title pipe
  fixture.detectChanges();

  expect(nameDisplay.textContent).toBe('Quick Brown  Fox');
});
```

## Calling [compile components](https://angular.io/guide/testing-components-scenarios#calling-compilecomponents):

Recall that the application hasn't been compiled. So when you call createComponent(), the TestBed compiles implicitly.

```typescript
beforeEach(async () => {
  await TestBed.configureTestingModule({
    declarations: [ BannerComponent ],
  }).compileComponents();
  fixture = TestBed.createComponent(BannerComponent);
  component = fixture.componentInstance;
  h1 = fixture.nativeElement.querySelector('h1');
});
```

> Ignore this section if you only run tests with the CLI ng test command because the CLI compiles the application before running the tests.

## Component with a dependency:

```typescript
TestBed.configureTestingModule({
   declarations: [ WelcomeComponent ],
// providers: [ UserService ],  // NO! Don't provide the real service!
                                // Provide a test-double instead 
                                // (stubs, fakes, spies, or mocks))
   providers: [ { provide: UserService, useValue: userServiceStub } ],
});

let userServiceStub: Partial<UserService>;
userServiceStub = {
  isLoggedIn: true,
  user: { name: 'Test User' },
};
```

The purpose of the spec is to test the component, not the service, and real services can be trouble.

The safest way to get the injected service, the way that always works, is to get it from the injector of the component-under-test. The component injector is a property of the fixture's DebugElement.

```typescript
// UserService actually injected into the component
userService = fixture.debugElement.injector.get(UserService);
```

> For a use case in which TestBed.inject() does not work, see the Override component providers section that explains when and why you must get the service from the component's injector instead.


```typescript
let userServiceStub: Partial<UserService>;

beforeEach(() => {
  // stub UserService for test purposes
  userServiceStub = {
    isLoggedIn: true,
    user: { name: 'Test User' },
  };

  TestBed.configureTestingModule({
     declarations: [ WelcomeComponent ],
     providers: [ { provide: UserService, useValue: userServiceStub } ],
  });

  fixture = TestBed.createComponent(WelcomeComponent);
  comp    = fixture.componentInstance;

  // UserService from the root injector
  userService = TestBed.inject(UserService);

  //  get the "welcome" element by CSS selector (e.g., by class name)
  el = fixture.nativeElement.querySelector('.welcome');
});

it('should welcome the user', () => {
  fixture.detectChanges();
  const content = el.textContent;
  expect(content)
    .withContext('"Welcome ..."')
    .toContain('Welcome');
  expect(content)
    .withContext('expected name')
    .toContain('Test User');
});

it('should welcome "Bubba"', () => {
  userService.user.name = 'Bubba'; // welcome message hasn't been shown yet
  fixture.detectChanges();
  expect(el.textContent).toContain('Bubba');
});

it('should request login if not logged in', () => {
  userService.isLoggedIn = false; // welcome message hasn't been shown yet
  fixture.detectChanges();
  const content = el.textContent;
  expect(content)
    .withContext('not welcomed')
    .not.toContain('Welcome');
  expect(content)
    .withContext('"log in"')
    .toMatch(/log in/i);
});
```


## (Particularly useful) Component with an [async service](https://angular.io/guide/testing-components-scenarios#component-with-async-service):

```html
template: `
  <p class="twain"><i>{{quote | async}}</i></p>
  <button type="button" (click)="getQuote()">Next quote</button>
  <p class="error" *ngIf="errorMessage">{{ errorMessage }}</p>`,
```

```typescript
getQuote() {
  this.errorMessage = '';
  this.quote = this.twainService.getQuote().pipe(
    startWith('...'),
    catchError( (err: any) => {
      // Wait a turn because errorMessage already set once this turn
      setTimeout(() => this.errorMessage = err.message || err.toString());
      return of('...'); // reset message to placeholder
    })
  );
```

### Testing with a spy:
```typescript
beforeEach(() => {
  testQuote = 'Test Quote';

  // Create a fake TwainService object with a `getQuote()` spy
  const twainService = jasmine.createSpyObj('TwainService', ['getQuote']);
  // Make the spy return a synchronous Observable with the test data
  getQuoteSpy = twainService.getQuote.and.returnValue(of(testQuote));

  TestBed.configureTestingModule({
    declarations: [TwainComponent],
    providers: [{provide: TwainService, useValue: twainService}]
  });

  fixture = TestBed.createComponent(TwainComponent);
  component = fixture.componentInstance;
  quoteEl = fixture.nativeElement.querySelector('.twain');
});
```
You can write many useful tests with this spy, even though its Observable is synchronous (of(testQuote)).


## TODO [Syncronous Tests](https://angular.io/guide/testing-components-scenarios#synchronous-tests):