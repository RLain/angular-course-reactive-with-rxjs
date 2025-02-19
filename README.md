# Angular course: Reactive angular with RxJS
🔗 https://angular-university.io/course/reactive-angular-course
🔗 https://rxjs.dev/

Starting point: $ git clone --branch 1-start https://github.com/angular-university/reactive-angular-course.git

The essential concepts in RxJS which solve async event management are:

- Observable: represents the idea of an invokable collection of future values or events.
- Observer: is a collection of callbacks that knows how to listen to values delivered by the Observable.
- Subscription: represents the execution of an Observable, is primarily useful for cancelling the execution.
- Operators: are pure functions that enable a functional programming style of dealing with collections with operations like map, filter, concat, reduce, etc.
- Subject: is equivalent to an EventEmitter, and the only way of multicasting a value or event to multiple Observers.
- Schedulers: are centralized dispatchers to control concurrency, allowing us to coordinate when computation happens on e.g. setTimeout or requestAnimationFrame or others.

# Stateless Observables Service


## Traditional imperative style

To understand how reactive design works, we first need to understand how the traditional imperative style works.

"In programming, imperative style is a paradigm where code explicitly describes how to achieve a specific result through a sequence of instructions or commands. This is in contrast to declarative style, which focuses on describing what the desired result is, leaving the "how" to the underlying system."

"Imperative code provides clear, sequential instructions, making it:
• Easier to follow for simple tasks.
• Well-suited for scenarios where step-by-step control is essential.

However, it can become verbose and harder to maintain as complexity increases, particularly in asynchronous or reactive programming contexts like Angular."

To kick off the understanding, we start by using the httpClient (Observable based service) to hit our backend.

```ts
export class HomeComponent implements OnInit {
  beginnerCourses: Course[];
  advancedCourses: Course[];

  constructor(private http: HttpClient, private dialog: MatDialog) {}

  ngOnInit() {

    this.http.get('/api/courses')
      .subscribe(
        res => {

          const courses: Course[] = res["payload"].sort(sortCoursesBySeqNo);

          this.beginnerCourses = courses.filter(course => course.category == "BEGINNER");

          this.advancedCourses = courses.filter(course => course.category == "ADVANCED");

        });

  }
```

### proxy.json

Small side note: package.json script "start": "ng serve  --proxy-config ./proxy.json" leverages off the proxy.json file e.g.

```json
{
  "/api": {
    "target": "http://localhost:9000",
    "secure": false
  }
}
```

Meaning when we hit the HTTP Client uses /api, it leverages off the defined target in proxy mode ergo invokes localhost:9000.

```ts
this.http.get('/api/courses').subscribe(..etc)
```

### Oberserable

The following creates an observable, which allows us to observe the behaviour:

```ts
 ngOnInit() {
    this.http.get('/api/courses')
      .subscribe(
        res => {
          const courses: Course[] = res["payload"].sort(sortCoursesBySeqNo);

          this.beginnerCourses = courses.filter(course => course.category == "BEGINNER");

          this.advancedCourses = courses.filter(course => course.category == "ADVANCED");

        });

  }
```

An observable may emit many values over its lifetime, it also may emit none. We can receive the events emitted by the observable via the callback aka `rest => {}` which responds to any emissions. 

The properties this.beginnerCourses and this.advancedCourses are directly assigned the filtered arrays. This direct manipulation of state reflects a typical imperative approach.

## Refactoring to a stateless observable service

The first potential issue raised with the imperative style is the potential for [Callback hell](https://callbackhell.com/).

There is also too much logic inside the component. The component knows that data is coming from the backend, it also knows how to do the HTTP call and process the data it receives.

Ideally a component shouldn't know where the data is coming from. And also it should be easy to test the component in isolation without having to use a real HTTP backend. 

Another thing is that keeping data in the following immutable properties is problematic as if there is any change in the data locally in the component, there is no way for the application to know the data has been modified:

```ts
beginnerCourses: Course[];
advancedCourses: Course[];
```

Ideally, code should be flat and placed inside an Angular service for reusability. Here is the new CourseService that we created:

```ts
@Injectable({
  providedIn: 'root'
})
export class CourseService {
  constructor(private http: HttpClient) {
    
  }

  loadAllCourses(): Observable<Course[]> {
    return this.http.get<Course[]>("/api/courses")
      .pipe(
        map(res => res["payload"])
      )
  }
}
```

🔗 https://rxjs.dev/api/operators/map

Important, the data is not stored in the service meaning the service is *stateless*. The data returned is only accessible by the invoking component. The service itself does not have access to the application data nor does it keep it in memory.

Stateless services are a crucial design pattern in Angular (and similar frameworks) because they help maintain scalability, simplicity, and testability. This prevents issues like data inconsistency or synchronization problems across components.

• In a stateless service, each call is independent, so the service can handle multiple simultaneous requests without worrying about internal state conflicts.
• This is especially important in large applications where multiple components might rely on the same service at the same time.
• Stateless services are easier to test because their behavior is solely based on the inputs and outputs of their methods.

Inside the home.component.ts we then update the ngOnInit to now utilise this new service. Note that when a variable is an observable, add a `dollar sign` to the end of it to make it easier to identify. A typical reactive style component will generally only have Onservable variables.

Note that this will _not_ work:
```html
          <mat-card *ngFor="let course of beginnerCourses$" class="course-card mat-elevation-z10">
````

We have to use the async pipe. When the component gets destroyed, the async pipe will unsubscribe from the observable avoiding any potential memory leaks.

```html
          <mat-card *ngFor="let course of (beginnerCourses$ | async)" class="course-card mat-elevation-z10">
```

Here is the overall refactor:

```ts
export class HomeComponent implements OnInit {
  beginnerCourses$: Observable<Course[]>;
  advancedCourses$: Observable<Course[]>;

  constructor(
    private coursesService: CourseService,
    private dialog: MatDialog
  ) {}

  ngOnInit() {
    const courses$ = this.coursesService.loadAllCourses()
      .pipe(
        map(courses => courses.sort(sortCoursesBySeqNo)
      )
    )

    this.beginnerCourses$ = courses$.pipe(
      map((courses) =>
        courses.filter((course) => course.category === "BEGINNER")
      )
    );

    this.advancedCourses$ = courses$.pipe(
      map((courses) =>
        courses.filter((course) => course.category === "ADVANCED")
      )
    );
  }
  ```


### Advantages of the refactoring

1. New service that can be reused throughut the application (CoorsesService)
2. No potential for callback hell. Everything is defined through observables
3. The data in the component is no longer mutuable state variables, but rather obserables. We can't access the data but can subscribe to the obseravble.
4. The async pipe subscribes to the obseravble making the data available to the view and also unsubscribes preventing memory leaks.


### Avoiding duplicate HTTP requests

The refactored approach is still not 100%. The following invokes the server twice, and this is because there are TWO subscriptions. This means we are making a _redundant_ call to the server. This can be seen by loading the page with Dev Tools open, and navigating to Network > Fetch/XHR:

```ts
 ngOnInit() {
    const courses$ = this.coursesService.loadAllCourses()
      .pipe(
        map(courses => courses.sort(sortCoursesBySeqNo)
      )
    )

// First subscription
    this.beginnerCourses$ = courses$.pipe(
      map((courses) =>
        courses.filter((course) => course.category === "BEGINNER")
      )
    );

// Second subscription
    this.advancedCourses$ = courses$.pipe(
      map((courses) =>
        courses.filter((course) => course.category === "ADVANCED")
      )
    );
  }

  //We can also showcase how this happens by adding a third subscription. This will then show three requests on the network tab:
  courses$.subscribe(val => console.log(val))
  ```

  To solve this, we can chain the `shareReplay()` operator onto the loadAllCourses observable:

  ```ts
    loadAllCourses(): Observable<Course[]> {
    return this.http.get<Course[]>("/api/courses")
      .pipe(
        map(res => res["payload"]),
        shareReplay()
      )
  }
  ```

  This will now only do one HTTP request when the service is invoked. No change was required to the ngOnit method on the component.

  This is _almost always_ what we want in practise. We don't want data retrieval or data modification requests being triggered more than once. For most of the time when using the Angular HTTPClient, add shareReplay() to the observable (on the service).

### Angular view: Layer Patterns - Smart vs Presentational Components

This section starts by highlighting how the original home.component.html had two identical mat-card definitions with the only variance being the data: beginningCourse$ vs advancedCourse$.

Due to the identical presentation nature it makes sense to have a single source of truth that can be reused by both scenarios. We created a new component using the following command:

$ ng generate course name-of-component --project name-of-project e.g. ng generate component courses-card-list --project reactive-angular-course will create courses-card-list.ts

Then simply moved the card into this component, then on home.component.html simple pass in the type of courses when we invoke:

```html
    <mat-tab-group>

        <mat-tab label="Beginners">

          <courses-card-list [courses]="beginnerCourses$ | async"></courses-card-list>

        </mat-tab>

        <mat-tab label="Advanced">

          <courses-card-list [courses]="advancedCourses$ | async"></courses-card-list>

        </mat-tab>

    </mat-tab-group>
```

A `presentational component` simply renders information provided to the component. It doesn't know where the data has come from, it simply needs to present that data. 

Versus the home.comoponent.ts which knows about the service layer, knows where the data comes from, and knows how to prepare the observables. However it has very little information about how to display the information.

The presentation component (courses-card-list.ts) doesn't know about the service layer of the application, instead it receives all its data via inputs.

This approach can be very useful, but distinguishing between smart and presentational components can be taken too far.... Don't worry about making every component either smart or pure presentational, rather think of them as high level recipes opposed to strict best practises that must be followed at all costs.

_______________________________________

# Reactive Component Interaction

## Decoupled component communication using a shared Service

In this example, we created the LoadingComponent which is shared across multiple components at different levels of the tree. This component is used to indicate an interaction is being awaited from the backend (e.g. a spinner showing a DB response is required etc).

ℹ️ Note to future self: The advantage of a shared service and decoupled componnent is when you have `multiple components making asynchronous calls, and you want a single loading spinner displayed for all of them`. Ergo `shared state`. For individual components where you don't have multiple calls awaiting a response, using the local state is fine aka having a showSpinner property and direct <mat-spinner> on the component itself.

To create this shared service this we did the following:

1. Defined the loading.component.html to include a mat-spinner
2. Added `<loading></loading>` to the top level app.component.html
3. Created the LoadingService (note this is referred to as an "API for the service") and leaving in its default configuration. 

Unlike the CoursesService we _won't_ be defining the providedIn:'root' on the LoadingService Injectable as this indicates that we want the service to be a global singleton (only one instance of the service on the whole application). This won't be the case for the LoadingService as it might have multiple instances in the application. 

4. Adding the LoadingService to the `providers` array on the top app.component.ts

Providers: Configures the injector of this directive or component with a token that maps to a provider of a dependency.

```ts
@Component({
  selector: "app-root",
  templateUrl: "./app.component.html",
  styleUrls: ["./app.component.css"],
  standalone: false,
  providers: [ LoadingService ], //Here
})
```

5. Adding the private loadingService: LoadingService to the various components. Aka LoadingComponent, HomeComponent & CourseDialogComponent.

```ts
export class HomeComponent implements OnInit {
  ...

  constructor(private coursesService: CourseService,
              private loadingService: LoadingService) {} //Here
```

By adopting a reactive design we make it very simple for different components at different levels of the Angular component tree to easily interact with the loading component in a decoupled way.


## LoadingService Reactive API Design

This is a continuation of the previous decoupled component lecture.

We started by adding the Obseravble to the service

```ts
@Injectable()
export class LoadingService {
  loading$: Observable<boolean>
}
```

We then updated the LoadingComponent constructor from private loadingService to `public loadingService` so it is accessible by the template of the component. We then consume the loadingService in the html using the async pipe:

```html
@if(loadingService.loading$ | async){
<div class="spinner-container">
  <mat-spinner></mat-spinner>
</div>
}
```

Next, on the LoadingService itself, we create three methods that we can then invoke in our various components:

```ts
@Injectable()
export class LoadingService {
  loading$: Observable<boolean>; //This has not yet been assigned a value

  showLoadingUntilCompleted<T>(observable$: Observable<T>): Observable<T>{
      return undefined //Logic to be added
  }

  loadingOn() {} //Logic to be added

  loadingOff() {} //Logic to be added
}
```

## Interaction using custom obserables and Behaviour Subject

The most important part of the reactive design is now assigning the observable a value and emiting this value. To do this we will use a `Subject`. They are very similar to observables in the sense that we can subscribe to it, with the added benefit of being able to `emit the value`. An observable is only a subsription and we can't control the values emitted. With a subject we can define `what` value to emit. 

Note in this lecture, Vasco explains that the RxJS library has two subject types:

- `new Subject()`: A Subject is a special type of Observable that allows values to be multicasted to many Observers. Subjects are like EventEmitters.
- `new BehaviorSubject()`: A variant of Subject that requires an initial value and emits its current value whenever it is subscribed to. - In general this one is recommended, `this is a special type of subject that remembers the last value` submitted by the subject. Better in general of async applications as we don't know the exact timings of the lifecycle of the component.

The following initially emits the value of `false`:

```ts
  private loadingSubject = new BehaviorSubject()<boolean>(false);
```

We want to keep private to prevent other parts of the application changing the value. This needs to be controlled by the LoadingService. Any component outside of the service needs to be able to subscribe to the values, but _only_ the LoadingService must have control to emit the values.

We then updated the loading$ observable 

```ts
  loading$: Observable<boolean> = this.loadingSubject.asObservable();
```

and added the values for emission in the methods:

```ts
  loadingOn() {
    this.loadingSubject.next(true)
  }

  loadingOff() {
    this.loadingSubject.next(false)
  }
```

Next, we invoked the methods on an example component HomeComponent:

```ts
reloadCourses() {
    this.loadingService.loadingOn() //Added this to turn on the loader

    const courses$ = this.coursesService
      .loadAllCourses()
      .pipe(
        map((courses) => courses.sort(sortCoursesBySeqNo)),
        finalize(() =>  this.loadingService.loadingOff()) //Added this to turn off the loader
      );

    //...etc
  }
  ```

`finalize()` = Returns an Observable that mirrors the source Observable, but will call a specified function when the source terminates on complete or error.


👀 Resume section 3 lecture 15: https://www.udemy.com/course/rxjs-reactive-angular-course/learn/lecture/18305358#questions