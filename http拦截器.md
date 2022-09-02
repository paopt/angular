拦截器是`HttpClient`的依赖服务，所以`HTTPClient`要保持单例，在根模块导入`HttpClientModule`。拦截器可以对请求、相应进行拦截。调用顺序为：
请求 =>
                          <= 响应
拦截器1  =>拦截器2 ... => 拦截器n

例子：
```typescript

@Injectable()
export class CustomJsonInterceptor implements HttpInterceptor {
  constructor(private jsonParser: JsonParser) {}
  intercept(httpRequest: HttpRequest<any>, next: HttpHandler) {
    // 拦截请求
    return next.handle(httpRequest).pipe(
      // 拦截响应
    );
  }
}
```

看代码什么都清楚了
```typescript
@Injectable()
export class HttpClient {
  // HttpHandler就是拦截器链，最后一个是HttpBackend，真正发送请求的地方
  constructor(private handler: HttpHandler) {}

  request(first: string|HttpRequest<any>, url?: string, options: {
    body?: any,
    headers?: HttpHeaders|{[header: string]: string | string[]},
    context?: HttpContext,
    observe?: 'body'|'events'|'response',
    params?: HttpParams|
          {[param: string]: string | number | boolean | ReadonlyArray<string|number|boolean>},
    reportProgress?: boolean,
    responseType?: 'arraybuffer'|'blob'|'json'|'text',
    withCredentials?: boolean,
  } = {}): Observable<any> {
    let req: HttpRequest<any>;
    // First, check whether the primary argument is an instance of `HttpRequest`.
    if (first instanceof HttpRequest) {
      // It is. The other arguments must be undefined (per the signatures) and can be
      // ignored.
      req = first;
    } else {
      // It's a string, so it represents a URL. Construct a request based on it,
      // and incorporate the remaining arguments (assuming `GET` unless a method is
      // provided.

      // Figure out the headers.
      let headers: HttpHeaders|undefined = undefined;
      if (options.headers instanceof HttpHeaders) {
        headers = options.headers;
      } else {
        headers = new HttpHeaders(options.headers);
      }

      // Sort out parameters.
      let params: HttpParams|undefined = undefined;
      if (!!options.params) {
        if (options.params instanceof HttpParams) {
          params = options.params;
        } else {
          params = new HttpParams({fromObject: options.params} as HttpParamsOptions);
        }
      }

      // Construct the request.
      req = new HttpRequest(first, url!, (options.body !== undefined ? options.body : null), {
        headers,
        context: options.context,
        params,
        reportProgress: options.reportProgress,
        // By default, JSON is assumed to be returned for all calls.
        responseType: options.responseType || 'json',
        withCredentials: options.withCredentials,
      });
    }

    // Start with an Observable.of() the initial request, and run the handler (which
    // includes all interceptors) inside a concatMap(). This way, the handler runs
    // inside an Observable chain, which causes interceptors to be re-run on every
    // subscription (this also makes retries re-run the handler, including interceptors).
    const events$: Observable<HttpEvent<any>> =
        of(req).pipe(concatMap((req: HttpRequest<any>) => this.handler.handle(req)));

    // If coming via the API signature which accepts a previously constructed HttpRequest,
    // the only option is to get the event stream. Otherwise, return the event stream if
    // that is what was requested.
    if (first instanceof HttpRequest || options.observe === 'events') {
      return events$;
    }

    // The requested stream contains either the full response or the body. In either
    // case, the first step is to filter the event stream to extract a stream of
    // responses(s).
    const res$: Observable<HttpResponse<any>> = <Observable<HttpResponse<any>>>events$.pipe(
        filter((event: HttpEvent<any>) => event instanceof HttpResponse));

    // Decide which stream to return.
    switch (options.observe || 'body') {
      case 'body':
        // The requested stream is the body. Map the response stream to the response
        // body. This could be done more simply, but a misbehaving interceptor might
        // transform the response body into a different format and ignore the requested
        // responseType. Guard against this by validating that the response is of the
        // requested type.
        switch (req.responseType) {
          case 'arraybuffer':
            return res$.pipe(map((res: HttpResponse<any>) => {
              // Validate that the body is an ArrayBuffer.
              if (res.body !== null && !(res.body instanceof ArrayBuffer)) {
                throw new Error('Response is not an ArrayBuffer.');
              }
              return res.body;
            }));
          case 'blob':
            return res$.pipe(map((res: HttpResponse<any>) => {
              // Validate that the body is a Blob.
              if (res.body !== null && !(res.body instanceof Blob)) {
                throw new Error('Response is not a Blob.');
              }
              return res.body;
            }));
          case 'text':
            return res$.pipe(map((res: HttpResponse<any>) => {
              // Validate that the body is a string.
              if (res.body !== null && typeof res.body !== 'string') {
                throw new Error('Response is not a string.');
              }
              return res.body;
            }));
          case 'json':
          default:
            // No validation needed for JSON responses, as they can be of any type.
            return res$.pipe(map((res: HttpResponse<any>) => res.body));
        }
      case 'response':
        // The response stream was requested directly, so return it.
        return res$;
      default:
        // Guard against new future observe types being added.
        throw new Error(`Unreachable: unhandled observe type ${options.observe}}`);
    }
  }
}


@NgModule({
  /**
   * Optional configuration for XSRF protection.
   */
  imports: [
    HttpClientXsrfModule.withOptions({
      cookieName: 'XSRF-TOKEN',
      headerName: 'X-XSRF-TOKEN',
    }),
  ],
  /**
   * Configures the [dependency injector](guide/glossary#injector) where it is imported
   * with supporting services for HTTP communications.
   */
  providers: [
    HttpClient,
    {provide: HttpHandler, useClass: HttpInterceptingHandler},
    HttpXhrBackend,
    {provide: HttpBackend, useExisting: HttpXhrBackend}, // 发送请求到后端的服务ajax
  ],
})
export class HttpClientModule {
}


@Injectable()
export class HttpInterceptingHandler implements HttpHandler {
  private chain: HttpHandler|null = null;

  constructor(private backend: HttpBackend, private injector: Injector) {}

  handle(req: HttpRequest<any>): Observable<HttpEvent<any>> {
    // 在这里把拦截器串起来， 最后一个是HttpBackend
    if (this.chain === null) {
      const interceptors = this.injector.get(HTTP_INTERCEPTORS, []);
      this.chain = interceptors.reduceRight(
          (next, interceptor) => new HttpInterceptorHandler(next, interceptor), this.backend);
    }
    return this.chain.handle(req);
  }
}
```

可以像下面手动创建http实例, 这样就不会调用拦截器，相当于直接发送请求到后端
```typescript
constructor(private backend: HttpBackend) {
  this.http = new HttpClient(backend);
}
```

