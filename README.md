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

Due to the identical presentation nature it makes sense to have a single source of truth that can be reused by both scenarios.

$ ng generate course name-of-component --project name-of-project e.g. ng generate component courses-card-list --project reactive-angular-course will create courses-card-list.ts

A `presentational component` simply renders information provided to the component. It doesn't know where the data has come from, it simply needs to present that data. 

Versus the home.comoponent.ts which knows about the service layer, knows where the data comes from, and knows how to prepare the observables. However it has very little information about how to display the information.

The presentation component (courses-card-list.ts) doesn't know about the service layer of the application, instead it receives all its data via inputs.

This approach can be very useful and distinctiong between smart and presentational components can be taken too far. Don't worry about making every component either smart or pure presentational....think of them as high level recipes opposed to strict best practises that must be followed at all costs.











