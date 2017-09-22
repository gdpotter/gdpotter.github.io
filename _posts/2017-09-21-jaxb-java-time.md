---
layout: post
title: Using Java 8 Time within JAXB with XJC
comments: true
github: https://github.com/gdpotter/gradle-jaxb-example
---
When we left off on our [Gradle JAXB project]({{ site.baseurl }}{% post_url 2017-08-14-jaxb-gradle-config %}), we were using a simple Gradle configuration to generate Java classes from an XML schema using JAXB and XJC. However, once you start to use the generated classes, you will notice something about date fields that might be difficult to use. Once again, we are using the [ISO 20022 schemas](https://www.iso20022.org/full_catalogue.page) in the sample.

If you don't want to follow the entire tutorial, you can jump right to the complete code on [GitHub](https://github.com/gdpotter/gradle-jaxb-example).

Before we begin, let's take a look at the issue. Notice the `FinancialInstrumentAttributes79` class:
```java
public class FinancialInstrumentAttributes79 {

  @XmlElement(name = "FinInstrmId", required = true)
  protected SecurityIdentification19 finInstrmId;
  @XmlElement(name = "PlcOfListg")
  protected MarketIdentification3Choice plcOfListg;
  @XmlElement(name = "DayCntBsis")
  protected InterestComputationMethodFormat4Choice dayCntBsis;
  @XmlElement(name = "ClssfctnTp")
  protected ClassificationType32Choice clssfctnTp;
  @XmlElement(name = "OptnStyle")
  protected OptionStyle8Choice optnStyle;
  @XmlElement(name = "DnmtnCcy")
  protected String dnmtnCcy;
  @XmlElement(name = "NxtCpnDt")
  protected XMLGregorianCalendar nxtCpnDt;
  @XmlElement(name = "XpryDt")
  protected XMLGregorianCalendar xpryDt;
  @XmlElement(name = "FltgRateFxgDt")
  protected XMLGregorianCalendar fltgRateFxgDt;
  @XmlElement(name = "MtrtyDt")
  protected XMLGregorianCalendar mtrtyDt;
  @XmlElement(name = "IsseDt")
  protected XMLGregorianCalendar isseDt;

  ...

}
```

While some of those fields use simple strings, the default `XMLGregorianCalendar` class is not the easiest to work with. Some of these fields do not even store time information and would be much better suited with the Java 8 `LocalDate` class. Luckily, with a bit of xml configuration in the bindings file, we can tell XJC to use [type adapters](http://docs.oracle.com/javase/8/docs/api/javax/xml/bind/annotation/adapters/XmlAdapter.html) for Java 8's Date and Time API (aka JSR-310).

## Getting Started

At its core, JAXB uses the [`XmlAdapter<ValueType,BoundType>`](http://docs.oracle.com/javase/8/docs/api/javax/xml/bind/annotation/adapters/XmlAdapter.html) class to convert between the XML string and the Java class. We could extend this class ourselves but I found an open source project [jaxb-java-time-adapters](https://github.com/migesok/jaxb-java-time-adapters) that did the job quite well. To start, we will need to add this dependency to gradle so that jaxb can use the classes:

```gradle
dependencies {
  compile(files(genJaxb.classesDir).builtBy(genJaxb))
  jaxb "com.sun.xml.bind:jaxb-xjc:2.1.7"
  jaxb "com.migesok:jaxb-java-time-adapters:1.1.3"
}
```

Next, we will need to add the xjc namespace to the top of the `binding.xml` file:
```xml
<bindings xmlns="http://java.sun.com/xml/ns/jaxb" version="2.1"
          xmlns:xjc="http://java.sun.com/xml/ns/jaxb/xjc">
  ...
</bindings>
```

However, you're not done yet. If you were to try to run `gradle clean genJaxb` right now, you would see an error similar to the following:
```
[ant:xjc] [ERROR] vendor extension bindings (jaxb:extensionBindingPrefixes) are not allowed in the strict mode. Use -extension.
```

The error is pretty self-explanatory but it can be difficult to act on if you don't know where to look. We just need to add one simple line to the xjc ant configuration in the `build.gradle` file:
```gradle
xjc(destdir: sourcesDir, binding: "${projectDir}/src/main/resources/binding.xml") {
  schema(dir: "${projectDir}/src/main/resources", includes: '**/*.xsd')
  arg(value: "-extension")
  produces(dir: sourcesDir, includes: '**/*.java')
}
```

## Adding the Type Adapter

After adding the dependency and enabling extension bindings, we can configure our new type adapters. Just add the following to the top of the `binding.xml` file:

```xml
<bindings xmlns="http://java.sun.com/xml/ns/jaxb" version="2.1"
          xmlns:xjc="http://java.sun.com/xml/ns/jaxb/xjc">

  <globalBindings>
    <xjc:javaType name="java.time.LocalDate" xmlType="xs:date"
      adapter="com.migesok.jaxb.adapter.javatime.LocalDateXmlAdapter" />
    <xjc:javaType name="java.time.LocalDateTime" xmlType="xs:dateTime"
      adapter="com.migesok.jaxb.adapter.javatime.LocalDateTimeXmlAdapter" />
  </globalBindings>

  ...

</bindings>
```

We're just going to configure the adapters for `LocalDate` and `LocalDateTime` (binding them to the `xs:date` and `xs:dateTime` XML types) but you can add as many as you would like.

## Result

Now let's look back at that same `FinancialInstrumentAttributes79` class from above:
```java

public class FinancialInstrumentAttributes79 {

  @XmlElement(name = "FinInstrmId", required = true)
  protected SecurityIdentification19 finInstrmId;
  @XmlElement(name = "PlcOfListg")
  protected MarketIdentification3Choice plcOfListg;
  @XmlElement(name = "DayCntBsis")
  protected InterestComputationMethodFormat4Choice dayCntBsis;
  @XmlElement(name = "ClssfctnTp")
  protected ClassificationType32Choice clssfctnTp;
  @XmlElement(name = "OptnStyle")
  protected OptionStyle8Choice optnStyle;
  @XmlElement(name = "DnmtnCcy")
  protected String dnmtnCcy;
  @XmlElement(name = "NxtCpnDt", type = String.class)
  @XmlJavaTypeAdapter(LocalDateXmlAdapter.class)
  protected LocalDate nxtCpnDt;
  @XmlElement(name = "XpryDt", type = String.class)
  @XmlJavaTypeAdapter(LocalDateXmlAdapter.class)
  protected LocalDate xpryDt;
  @XmlElement(name = "FltgRateFxgDt", type = String.class)
  @XmlJavaTypeAdapter(LocalDateXmlAdapter.class)
  protected LocalDate fltgRateFxgDt;
  @XmlElement(name = "MtrtyDt", type = String.class)
  @XmlJavaTypeAdapter(LocalDateXmlAdapter.class)
  protected LocalDate mtrtyDt;
  @XmlElement(name = "IsseDt", type = String.class)
  @XmlJavaTypeAdapter(LocalDateXmlAdapter.class)
  protected LocalDate isseDt;

  ...

}
```

Now the generated classes are using `LocalDate` instead of `XMLGregorianCalendar` for these dates. Notice the `@XMLJavaType` annotation on the field--that is how the JAXB marshaller knows how to marshal and unmarshal the XML and these generated classes.
