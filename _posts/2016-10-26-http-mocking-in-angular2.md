---
layout:	post
title:	"HTTP Mocking in Angular 2"
date:	2016-10-26 09:36:00 +0200
categories:	Testing
tags:	[Angular2, testing, http]
---

# Introduction

Mocking out HTTP calls in your Angular 2 apps is something you might want to consider in your Angular 2 unit tests. The API calls are essentially a stability point and you can emulate these calls as part of your testing.

The big problem however, is the amount of work needed to put some of these tests in place using Angular 2, Jasmine and Karma.

# Mocking your HTTP

For an in depth explanation, check out this [blog from Ken Rimple](http://chariotsolutions.com/blog/post/testing-angular-2-0-x-services-http-jasmine-karma/) about how to mock out your HTTP.

Here's some mocking of HTTP calls.

Lets assume you have a service that looks something like this:

{% highlight typescript %}
@Injectable()
export class StatusService {
    constructor(private http:Http){}

    getStatus(){
        return this.http.get('http://svc/status').map((res:Response) => res.json());
    }
}
{% endhighlight %}

Your test might look something like this:

{% highlight typescript %}
describe('Status Service', () => {
  let mockBackend: MockBackend;

  beforeEach(async(() => {
    TestBed.configureTestingModule({
      providers: [
        StatusService,
        MockBackend,
        BaseRequestOptions,
        {
          provide: Http,
          deps: [MockBackend, BaseRequestOptions],
          useFactory:
            (backend: XHRBackend, defaultOptions: BaseRequestOptions) => {
              return new Http(backend, defaultOptions);
            }
       }
      ],
      imports: [
        HttpModule
      ]
    });
    mockBackend = getTestBed().get(MockBackend);
  }));
  
  it('Get the status', (done) => {
    let statusService: StatusService;

    getTestBed().compileComponents().then(() => {
      mockBackend.connections.subscribe(
        (connection: MockConnection) => {
            connection.mockRespond(new Response(
              new ResponseOptions({
                  body: {status: "submitted"}
            })));
        });

        statusService = getTestBed().get(StatusService);
        expect(statusService).toBeDefined();

        statusService.getStatus().subscribe( (data:any) => {
              expect(data).toBeDefined();
              expect(data.status).toBe("submitted");
              done();
        });
      });
  });
});
{% endhighlight %}

You'll of course also need the various imports.

As a very quick break down, we do the following:
-  Create a mock backend
-  Create a mock Response
-  Subscribe to the service response and check it

That's a lot of code!

# Making the HTTP mocking more manageable

Clearly there must be a better way. Fortunately the pattern is quite clear: Theres a big chuck of code for injecting the backend, and theres a big chunck of code for injecting the responses.

I've made some library methods that will add the injections for me, resulting in a test that now looks like this:
{% highlight typescript %}
describe('Status Service', () => {
  let mockBackend: MockBackend;

  beforeEach(async(() => {
    TestBed.configureTestingModule(HttpMockResponse.testModuleMetaDataWith({
      providers: [StatusService]
    }));

    mockBackend = getTestBed().get(MockBackend);
  }));
  
  it('Get the status', (done) => {
    let statusService: StatusService;

    getTestBed().compileComponents().then(() => {
          HttpMockResponse.mockHttpCalls(mockBackend, [
            HttpMockResponse.withBody({status:"submitted"})
        ]);

        statusService = getTestBed().get(StatusService);
        expect(statusService).toBeDefined();

        statusService.getStatus().subscribe( (data:any) => {
          expect(data).toBeDefined();
          expect(data.status).toBe("submitted");
          done();
        });
      });
  });
});
{% endhighlight %}

So what's changed? Well, mostly that theres no longer a wall of dependencies and that the responses are better encapsulated.

##  No more wall of dependencies 

I don't need to inject a wall of dependencies and custom factories. My current config settings are augmented by the call to `HttpMockResponse.testModuleMetaDataWith`.

This method basically appends the HTTP mocking dependencies to the meta data you pass in.

## My mock responses are encapsulated

My call to `HttpMockResponse.mockHttpCalls` plugs in a bunch of plumbing around mocking an HTTP response. Notably, you actually pass in an array, not just a single response

Also not demonstrated is that you can apply a filter to the responses - the mocking helper will return the first matching response evaulated in order of appearence. So you can do stuff like this:
{% highlight typescript %}
HttpMockResponse.mockHttpCalls(mockBackend, [ 
  HttpMockResponse.withBody({svcurl:"http://svcs/"}, connection => /\/config$/.test(connection.request.url)),
  HttpMockResponse.withBody({status:"submitted"}, connection => /\/status$/.test(connection.request.url))
]);
{% endhighlight %}

Lets assume my `StatusService` always pulls a config from another API call (Maybe it should be a a config service, but this is illustrative), my MockBackend will return the config body for calls ending in `/config` and the status for calls ending in `/status`.

To re-iterate, responses are evaulated in order of appearence and return the first match.

## Fewer imports on the tests

Another advantage of the above approach is the need for fewer imports. It's a little arb since it's mostly from one library, but it's nice I guess. This could be encapsulated by the helper library.

# Inside HttpMockResponse

I would suggest checking out my [github](https://github.com/geoffles/http-mock-response) to checkout the source for the helper. It's just an encapsulation of the injections, but here are some snippets:

{% highlight typescript %}
public static testModuleMetaDataWith(metaData:TestModuleMetadata):TestModuleMetadata{
    if (!metaData.providers){
        metaData.declarations = new Array<any>();
    }
    if (!metaData.imports){
        metaData.imports = new Array<any>();
    }
    
    metaData.providers.push(MockBackend);
    metaData.providers.push(BaseRequestOptions);
    metaData.providers.push({
      provide: Http,
      deps: [MockBackend, BaseRequestOptions],
      useFactory:
        (backend: XHRBackend, defaultOptions: BaseRequestOptions) => {
          return new Http(backend, defaultOptions);
        }
    });

    metaData.imports.push(HttpModule);
    
    return metaData;
}

public static mockHttpCalls(mockBackend:MockBackend, responses:Array<ResponseContainer>){
mockBackend
  .connections
  .subscribe((connection: MockConnection) => {
      let isMatch = responses.find(container => container.isMatch(connection));
      if (isMatch)
      {
          connection.mockRespond(isMatch.response);
      }
  });
}
{% endhighlight %}

`testModuleMetaDataWith` needs to check if the providers and imports already exist on the object passed in and create them if missing. Next, it adds the providers and imports.

`mockHttpCalls` finds the first matching `ResponseContainer`, which is a container class with a reponse, and a predicate to test for match. If there are no matches, there will be no responses - instead of throwing an exception, I'd rather let the test handle that.

# Conclusion

I've wrapped the HTTP mocking backend for Angular 2 into a helper class that simplifies adding HTTP mocking to my tests. This improves the readability of the test and reduces the ceremony around adding HTTP mocking to something a little easier to remember.
