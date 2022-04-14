CLI commands
============

## Code coverage

Generate a [code coverage report](https://angular.io/guide/testing-code-coverage).
```javascript
ng test --no-watch --code-coverage

// the following CLI command generates a BannerComponent in the app/banner
ng generate component banner --inline-template --inline-style --module app
```

If you want to create code-coverage reports every time you test, set the following option in the CLI configuration file, angular.json:
```json
"test": {
  "options": {
    "codeCoverage": true
  }
}
```

# Testing [Services](https://angular.io/guide/testing-services)

+ See src/app/demo/demo.spec.ts for the [simplest kind of tests](https://angular.io/guide/testing-services#testing-services).

> Prefer spies as they are usually the best way to mock services.

## Angular Testbed:


The [TestBed](https://angular.io/guide/testing-services#angular-testbed) creates a dynamically-constructed Angular test module that emulates an Angular @NgModule.

```typescript
let service: ValueService;

beforeEach(() => {
  TestBed.configureTestingModule({ providers: [ValueService] });
});

// To use:
beforeEach(() => {
  TestBed.configureTestingModule({ providers: [ValueService] });
  service = TestBed.inject(ValueService);
});
```

## Using Spies:
```typescript
let masterService: MasterService;
let valueServiceSpy: jasmine.SpyObj<ValueService>;

beforeEach(() => {
  const spy = jasmine.createSpyObj('ValueService', ['getValue']);

  TestBed.configureTestingModule({
    // Provide both the service-to-test and its (spy) dependency
    providers: [
      MasterService,
      { provide: ValueService, useValue: spy }
    ]
  });
  // Inject both the service-to-test and its (spy) dependency
  masterService = TestBed.inject(MasterService);
  valueServiceSpy = TestBed.inject(ValueService) as jasmine.SpyObj<ValueService>;
});

// To consume the spy:
it('#getValue should return stubbed value from a spy', () => {
  const stubValue = 'stub value';
  valueServiceSpy.getValue.and.returnValue(stubValue);

  expect(masterService.getValue())
    .withContext('service returned stub value')
    .toBe(stubValue);
  expect(valueServiceSpy.getValue.calls.count())
    .withContext('spy method was called once')
    .toBe(1);
  expect(valueServiceSpy.getValue.calls.mostRecent().returnValue)
    .toBe(stubValue);
});
```

## Testing HTTP services
>The subscribe() method takes a success (next) and fail (error) callback.
>Make sure you provide both callbacks so that you capture errors. 
>Neglecting to do so produces an asynchronous uncaught observable error 
>that the test runner will likely attribute to a completely different test.

```typescript
let httpClientSpy: jasmine.SpyObj<HttpClient>;
let heroService: HeroService;

beforeEach(() => {
  // TODO: spy on other methods too
  httpClientSpy = jasmine.createSpyObj('HttpClient', ['get']);
  heroService = new HeroService(httpClientSpy);
});

it('should return expected heroes (HttpClient called once)', (done: DoneFn) => {
  const expectedHeroes: Hero[] =
    [{ id: 1, name: 'A' }, { id: 2, name: 'B' }];

  httpClientSpy.get.and.returnValue(asyncData(expectedHeroes));

  heroService.getHeroes().subscribe({
    next: heroes => {
      expect(heroes)
        .withContext('expected heroes')
        .toEqual(expectedHeroes);
      done();
    },
    error: done.fail
  });
  expect(httpClientSpy.get.calls.count())
    .withContext('one call')
    .toBe(1);
});

it('should return an error when the server returns a 404', (done: DoneFn) => {
  const errorResponse = new HttpErrorResponse({
    error: 'test 404 error',
    status: 404, statusText: 'Not Found'
  });

  httpClientSpy.get.and.returnValue(asyncError(errorResponse));

  heroService.getHeroes().subscribe({
    next: heroes => done.fail('expected an error, not heroes'),
    error: error  => {
      expect(error.message).toContain('test 404 error');
      done();
    }
  });
});
```

For in-depth instructions see [Testing HTTP Requests](https://angular.io/guide/http#testing-http-requests).

# Basics of [Testing Components](https://angular.io/guide/testing-components-basics#basics-of-testing-components).

+ For testing ONLY the component see the reference "'LightswitchComp'" in app/demo/demo.spec.ts.

```typescript
export class DashboardHeroComponent {
  @Input() hero!: Hero;
  @Output() selected = new EventEmitter<Hero>();
  click() { this.selected.emit(this.hero); }
}

it('raises the selected event when clicked', () => {
  const comp = new DashboardHeroComponent();
  const hero: Hero = {id: 42, name: 'Test'};
  comp.hero = hero;

  comp.selected.pipe(first()).subscribe((selectedHero: Hero) => expect(selectedHero).toBe(hero));
  comp.click();
});
```

+ component with dependencies:
```typescript
class MockUserService {
  isLoggedIn = true;
  user = { name: 'Test User'};
}

beforeEach(() => {
  TestBed.configureTestingModule({
    // provide the component-under-test and dependent service
    providers: [
      WelcomeComponent,
      { provide: UserService, useClass: MockUserService }
    ]
  });
  // inject both the component and the dependent service.
  comp = TestBed.inject(WelcomeComponent);
  userService = TestBed.inject(UserService);
});

it('should not have welcome message after construction', () => {
  expect(comp.welcome).toBe('');
});

it('should welcome logged in user after Angular calls ngOnInit', () => {
  comp.ngOnInit();
  expect(comp.welcome).toContain(userService.user.name);
});

it('should ask user to log in if not logged in after ngOnInit', () => {
  userService.isLoggedIn = false;
  comp.ngOnInit();
  expect(comp.welcome).not.toContain(userService.user.name);
  expect(comp.welcome).toContain('log in');
});
```

## Components [DOM Testing](https://angular.io/guide/testing-components-basics#component-dom-testing):

```typescript
describe('BannerComponent (with beforeEach)', () => {
  let component: BannerComponent;
  let fixture: ComponentFixture<BannerComponent>;

  beforeEach(() => {
    TestBed.configureTestingModule({declarations: [BannerComponent]});
    fixture = TestBed.createComponent(BannerComponent);
    component = fixture.componentInstance;
  });

  it('should create', () => {
    expect(component).toBeDefined();
  });
  it('should contain "banner works!"', () => {
    const bannerElement: HTMLElement = fixture.nativeElement;
    expect(bannerElement.textContent).toContain('banner works!');
  });
  it('should have <p> with "banner works!"', () => {
    // these two lines are equivalent when running in the browser
    const bannerDe: DebugElement = fixture.debugElement.nativeElement;
    const bannerElement: HTMLElement = fixture.nativeElement;
    const p = bannerElement.querySelector('p')!;
    expect(p.textContent).toEqual('banner works!');
  });
});
```

+ native element:
  - tests that run in the browser always have the nativeElement value as HTMLElement.
  - use the standard HTML querySelector to dive deeper into the element tree.
  - By.css('p') instructions [here](https://angular.io/guide/testing-components-basics#bycss).
  