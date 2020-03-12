---
title: From memoization pattern to caching and idempotency 
date: 2019-08-25 12:00:00
---

In computer programming Memoization refers to a technique to speed up program by storing result of expensive function call 
in permanent/persistent storage, and return the cached result when function get called with the same input(s) again. 

Just as when we learn math and do some calculation to produce the result of 12^2 by following algebra steps, next time we can easily rememeber
the result by getting output of 12^2 from memory without doing all step again and again, because it's remembered in our brain. 

Consider the following expensive function, fee configuration will be get from database everytime we need to calculate fee for a special transaction.
```python
def calFee(productId, amount):
    fee = getFee(productId)
    return amount * fee.percent 
```
Memoization pattern can be applied so that, with the same productId, fee configuration only loaded from database once, then can be remembered and return immedately for the next call.

```python
fees = {}
def calFee(productId, amount):
    fee = None
    if productId in fees:
        fee = fees[productId]
    else:
        fee = getFee(productId)
        fees[productId] = fee
    return amount * fee.percent 
```

Computer programmer often implement memoization in much the same way, call it caching. Memoization is added inside function and explicity define how result is set and get. 

Some programming languages and frameworks also designed to provide a way to do this automatically. 
For example in python, we do not necessary set/get result to/from memory inside function implementation. Decorator can help.

In this version, fee configuration function can be pure, it means we only get data from database, without dealing anything with cache get and set. We also remove global caches object, because in Python everything is an object, caches can be attached directly into feeFunc

```python

def memoX(feeFunc):
   def precheck(productId):
      if not hasattr(feeFunc, 'caches'): feeFunc.caches = {}
      if productId in feeFunc.caches:
         return feeFunc.caches[productId]
      feeFunc.caches[productId] = feeFunc(productId)
      return feeFunc.caches[productId]
   return precheck

def calFee(productId, amount):
    getFee(productId) * amount

@memoX
def getFee(productId):
   import time
   time.sleep(1)
   return 0.1

def main():
    calFee(100, 1000)
    calFee(100, 2000)

if __name__ == '__main__':
    main()
```

In Java, Scala we can implement almost the same with Annotation. Framework such as Spring provides @Cacheable annotation, white Play has similar thing named @Cached or @CacheFor.

# Idempotency
So why the title mentioned idempotency? We use memoization to implement cache, the most reason is to speed up our application. 
Nowaday, cache is being used widely, from small website to very big platform, from local to distributed, but still, mostly because of speed. 

In distributed application, software is deployed as multi instances, a request can be processed by any instance of application cluster, and somehow, it can be processed twice or more than twice. 

A cache can be used to prevent processing an event twice, we cache result of previos computation, when we can differentiate between requests, we can prevent processing request multiple time. At first glance, we can think the pattern applied here because of speed, but in some scenario, it's for safety. 

Take an ecommerce application, has a feature to allow user to buy credit by money. The application connects to some payment gateway to do payment for example. At checkout step, user will be redirected to payment gateway to input payment credential information, and if the process went well, payment gateway will send an event to our application to notify the payment transaction status. 

What happen if we received 2 notifications for the same transaction? 

A bad implementation will be end up by giving user 2 x requested credit point. 
A good implementation will check for order status, then ignore or process payment transaction. But this way we will have to read database many time to check and make sure order status is success or not. 

A better implementaion is using memoization pattern. Same API input will always return same output, even if the same input get into our API multiple times, in API development, we name it `idempotent API`.

>
> On a long enough time scale, everything can happen will happen

Network is not reliable, that why almost every network library has retry strategy implementation. For example: [Apache http client](https://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/impl/client/DefaultHttpRequestRetryHandler.html) has DefaultHttpRequestRetryHandler turned on by default, Apache Axis which is commonly used by bank softwares uses Apache common http client as default http client. When working with 3rd party APIs, we also try to use/implement retry strategy to by pass network error. **Retryable operation is good, but sometimes  it can be very bad.**

Network error and retryable operation not also make callee misbehaviour, but also to the caller. 

Case 1

> A.callB(transid = 1) =>
  B.process(transid = 1) => 
  Network error => 
  A.retry( transid = 1 ) => 
  B returns duplicate transaction

Case 2

> A.callB(transid = 1) =>
  B.process(transid = 1) => 
  Network error => 
  A.retry( transid = 1 ) => 
  B.process(transid = 1) again

...and many other case.

# How to implement idempotency 



According to previous example, if we provide an API the bank to send notification to add point into user's account, the api interface will be like this

**Request**
  - Method: *GET*
  - Params: userid, transid & point where userid, transid are string, and point is a number
  - Example: ?transid=1000&point=10&userid=9345

**Response**
  - Format: json contains total point of user after point is added 
  - Example:
```json
{
    "point": 10,
    "userId": "2325345"
}
```

For the first time, implementation will be like this (I use java and jetty to demonstrate this example)

```java
package main.webserver.handler;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import main.webserver.model.Point;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;


public class PointHandler extends HttpServlet {
    // ingnore signature, user exists...and such
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        Gson gson = new GsonBuilder().create();

        String transId = req.getParameter("transid");
        String userId = req.getParameter("userid");
        long value = Long.valueOf(req.getParameter("point"));

        Point point = new Point(userId).add(transId, value);
        resp.getWriter().write(gson.toJson(point));

    }
}
```

With above implemetation, if by some reason we receive notification twice, we will end up giving user 2x point they paid for.

# Apply memoization

The pattern here is yet simple to apply, it's similar to how caching is implemented. We can set/get from cache internally and explicitly inside API handler, or we can also implement decorator/annotation which wrap the API handler, or as a filter, depend on language/framework we're using. 

The important thing we should take care of is: we should not change business code when apply this pattern, know what to change and what to keep. 

Most developers choose the first way, even add more and more abstractions into source code, do not care about what language/framework already support, adding more code to current implementation: 


```java
// WebServer.java
package main.webserver;

import main.webserver.handler.PointHandler;
import org.eclipse.jetty.server.Connector;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.ServerConnector;
import org.eclipse.jetty.servlet.FilterHolder;
import org.eclipse.jetty.servlet.FilterMapping;
import org.eclipse.jetty.servlet.ServletHandler;

public class WebServer {

    private Server server = new Server();
    final String pathSpec = "/discount";

    private void start() throws Exception {
        ServletHandler servletHandler = new ServletHandler();
        servletHandler.addServletWithMapping(PointHandler.class, pathSpec);
        ServerConnector serverConnector = new ServerConnector(server);
        serverConnector.setPort(8000);
        server.setConnectors(new Connector[]{
                serverConnector
        });
        server.setHandler(servletHandler);
        server.start();
    }
    public static void main(String[] args) throws Exception {
        new WebServer().start();
    }
}

```

```java
// PointHander.java
package main.webserver.handler;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import main.webserver.model.Point;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

public class PointHandler extends HttpServlet {
    // adding a cache storage
    private static Map<String, Point> caches = null;

    public PointHandler(){
            caches = new HashMap<>();
    }
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        Gson gson = new GsonBuilder().create();
        Point point;

        String transId = req.getParameter("transid");
        String userId = req.getParameter("userid");
        long value = Long.valueOf(req.getParameter("point"));

        // check if cache already exist by the key `transid` and return immediately
        if (caches.containsKey(transId)) {
            point = caches.get(transId);
            resp.getWriter().write(gson.toJson(point));
            return;
        }
        // otherwise add point to user and set item to cache
        point = new Point(userId).add(transId, value);
        caches.put(transId, point);
        resp.getWriter().write(gson.toJson(point));

    }
}
```

While this solution works, it introduces more code the the main business code, changing cache behavior, add more logic to business code will most likely introduce more bug. 

Jetty, as a mature Http server has already supported [Filter](http://www.servlets.com/javadoc/javax/servlet/Filter.html), we can use it to implement cache without touching anything in main logic source code (logic code remains as original version), Filter is easy to implement, and flexible enough to support multi API operation, endpoint: 

```java
// CacheFilter.java
package main.webserver.filter;

import com.google.gson.Gson;
import main.webserver.model.Point;

import javax.servlet.*;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpServletResponseWrapper;
import java.io.CharArrayWriter;
import java.io.IOException;
import java.io.PrintWriter;
import java.util.HashMap;
import java.util.Map;

public class CacheFilter implements javax.servlet.Filter{
    private static Map<String, Point>  caches = new HashMap<>();
    FilterConfig filterConfig = null;
    class CharResponseWrapper extends HttpServletResponseWrapper {
        private CharArrayWriter output;

        public String toString() {
            return output.toString();
        }

        public CharResponseWrapper(HttpServletResponse response) {
            super(response);
            output = new CharArrayWriter();
        }

        public PrintWriter getWriter() {
            return new PrintWriter(output);
        }
    }
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        this.filterConfig = filterConfig;
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        Point point;
        Gson gson = new Gson();
        PrintWriter writer = servletResponse.getWriter();
        final String transId = servletRequest.getParameter("transid");
        final String userId = servletRequest.getParameter("userid");
        // adding parameter check, so the main handler does not have to care
        if (transId == null || userId == null) {
            servletResponse.getWriter().write("bad request");
            return;
        }
        point = caches.get(transId);
        if (point != null) {
            writer.write(gson.toJson(point));
            return;
        }
        CharResponseWrapper responseWrapper = new CharResponseWrapper((HttpServletResponse) servletResponse);
        filterChain.doFilter(servletRequest, responseWrapper);
        point = gson.fromJson(responseWrapper.output.toString(), Point.class);
        caches.put(transId, point);
        writer.write(gson.toJson(point));

    }

    @Override
    public void destroy() {
        this.filterConfig = null;

    }
}
```

Install the filter: 

```java
package main.webserver;

import main.webserver.filter.CacheFilter;
import main.webserver.handler.PointHandler;
import org.eclipse.jetty.server.Connector;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.server.ServerConnector;
import org.eclipse.jetty.servlet.FilterHolder;
import org.eclipse.jetty.servlet.FilterMapping;
import org.eclipse.jetty.servlet.ServletHandler;

public class WebServer {

    private Server server = new Server();
    final String pathSpec = "/discount";

    private void start() throws Exception {
        ServletHandler servletHandler = new ServletHandler();
        servletHandler.addServletWithMapping(PointHandler.class, pathSpec);
        ServerConnector serverConnector = new ServerConnector(server);
        serverConnector.setPort(8000);
        server.setConnectors(new Connector[]{
                serverConnector
        });
        // install filter
        CacheFilter cacheFilter = new CacheFilter();
        FilterHolder filterHolder = new FilterHolder(cacheFilter);
        filterHolder.setName("discountCacheFilter");
        servletHandler.addFilter(filterHolder, this.createFilterMapping(pathSpec, filterHolder));
        // end
        server.setHandler(servletHandler);
        server.start();
    }

    private FilterMapping createFilterMapping(String pathSpec, FilterHolder filterHolder) {
        FilterMapping filterMapping = new FilterMapping();
        filterMapping.setPathSpec(pathSpec);
        filterMapping.setFilterName(filterHolder.getName());
        return filterMapping;
    }

    public static void main(String[] args) throws Exception {
        new WebServer().start();
    }
}

```

Writing Jetty filter, we have to write more code, but cache can be disabled/enabled without changing a single line of main business code.


# Conclusion

Caching is very popular solution, many people use it, but sometimes we dont know it can solve many other problem.

Idempotent API is very important, it can save computation resource, also prevent processing request multiple times, behave exactly the same if a single request get called multiple time. 

Language and framework was born many years ago, what we want to do, maybe someone already add to the framework we use, don't try to re-invent the wheel. 


