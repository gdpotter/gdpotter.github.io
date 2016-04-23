---
layout: post
title: Spring Filter Ordering
---

The internet is littered with examples of how to add [Cross Origin Resource Sharing](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) (CORS) support to a Spring Boot application. Almost all of these examples use the `SimpleCORSFilter` which looks something like this:

{% highlight java %}
@Component
public class SimpleCORSFilter implements Filter {

   	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
       	HttpServletResponse response = (HttpServletResponse) res;
        response.setHeader("Access-Control-Allow-Origin", "*");
   	    response.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE");
       	response.setHeader("Access-Control-Max-Age", "3600");
        response.setHeader("Access-Control-Allow-Headers", "Authorization, Origin, x-requested-with, Content-Type, Accept");
   	    chain.doFilter(req, res);
    }
   	public void init(FilterConfig filterConfig) {}
    public void destroy() {}
}
{% endhighlight %}

When working on an AngularJS project, I got stuck trying to determine why my requests were failing the CORS check but my filter was never being called.

After some investigation I found that Spring Security had its own filter that ended the `FilterChain` when authentication failed. I wasn't passing the authentication header in the `OPTIONS` request and so when Spring Security couldn't authorize the request, the rest of the chain stopped and thus never got to my filter.

As of Spring 2.5, fixing this issue is quite simple. Just add the `@Order` annotation to the filter like so:

{% highlight java %}
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class SimpleCORSFilter implements Filter {
{% endhighlight %}

The JavaDoc for `@Order` states:
> The default value is Ordered.LOWEST_PRECEDENCE, indicating lowest priority (losing to any other specified order value).

Unless you pass the `Oredered.HIGHEST_PRECEDENCE` (or some other integer), the filter will be towards the end of the chain.
