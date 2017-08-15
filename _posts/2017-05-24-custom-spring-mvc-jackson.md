---
layout: post
title: Custom Jackson Config for Spring Boot
comments: true
---
Spring Boot comes with some really great and usually sensible defaults for creating a Spring Application, but sometimes these defaults need to be tweaked slightly. I recently found myself needing to change some of the Jackson settings, but only for Web Requests. We wanted to use `snake_case` for the JSON fields in our request and response bodies.

The simplest way to solve this is to just create a custom `ObjectMapper` bean which would get picked up automatically by Spring Boot. This can be accomplished by doing something like the following:
```java
@Configuration
public class JacksonConfig {

  @Autowired
  public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder) {
    ObjectMapper objectMapper = builder.createXmlMapper(false).build();

    // Some other custom configuration for supporting Java 8 features
    objectMapper.registerModule(new Jdk8Module());
    objectMapper.registerModule(new JavaTimeModule());

    // Use property
    objectMapper.setPropertyNamingStrategy(PropertyNamingStrategy.SNAKE_CASE);

    return objectMapper;
  }
}
```

The issue with this approach is that it will change the default `ObjectMapper` for the rest of the application. We only want to change it for Spring MVC. To make a change, we will need to extend the `WebMvcConfigurerAdapter`:

```java
@EnableWebMvc
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {

  @Autowired
  private ObjectMapper objectMapper;

  @Override
  public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    ObjectMapper webObjectMapper = objectMapper.copy();
    webObjectMapper.setPropertyNamingStrategy(PropertyNamingStrategy.SNAKE_CASE);
    converters.add(new MappingJackson2HttpMessageConverter(webObjectMapper));
  }
}
```
Here we have injected the existing `ObjectMapper` and created a copy. On that copy we are able to set the desired property naming strategy (and any other configuration) and then added a custom message converter with it.

If you have not defined your own `ObjectMapper` bean, then the default one will be injected (see [JacksonAutoConfiguration](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/jackson/JacksonAutoConfiguration.java))
