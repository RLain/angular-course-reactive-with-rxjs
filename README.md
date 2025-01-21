# Angular course: Reactive angular with RxJS
🔗 https://angular-university.io/course/reactive-angular-course

Starting point: $ git clone --branch 1-start https://github.com/angular-university/reactive-angular-course.git

# Stateless Observables Service


## Traditional imperative style

We are using the httpClient to hit our backend.

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

The following creates an observable:

```ts
this.http.get('/api/courses').subscribe(..etc)
```

