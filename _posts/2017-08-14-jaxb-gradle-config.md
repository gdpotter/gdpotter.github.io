---
layout: post
title: A Simple Gradle JAXB Configuration
comments: true
github: https://github.com/gdpotter/gradle-jaxb-example
crosspost_to_medium: true
---
Working with any XML schema is often a daunting task and when using Java it is common to use a library such as JAXB to turn that schema into Java classes and then marshal (or unmarshal) the XML using those classes. The [XJC](http://docs.oracle.com/javase/6/docs/technotes/tools/share/xjc.html) tool will convert XML schemas into the Java classes but unfortunately due to the age of the project, integrating with gradle is not clear. However, there is an Ant plugin that we can invoke from Gradle to make generating these classes easy.

Because this is just a Gradle task, it can be included in any project. However, I've found placing the generated classes in its own jar to work well on any medium to large project. This way, the classes don't have to be constantly regenerated each time `gradle clean` is run and it allows any consumers to be consistently versioned to the correct schema.

For this example, we will be using the [ISO 20022 schemas](https://www.iso20022.org/full_catalogue.page) to generate Java classes. These schemas are especially complex with a single XSD file including hundreds of classes.

If you don't want to follow the entire tutorial, you can jump right to the complete code on [GitHub](https://github.com/gdpotter/gradle-jaxb-example).

## Getting Started
To get started, we are going to want to add the jaxb dependency and a basic task with our details:
```gradle
configurations {
    jaxb
}
dependencies {
    jaxb "com.sun.xml.bind:jaxb-xjc:2.1.7"
}
task genJaxb {
  ext.sourcesDir = "${buildDir}/generated-sources/jaxb"
  ext.classesDir = "${buildDir}/classes/jaxb"
  outputs.dir classesDir
}
```

## Working with a Single Schema
We want configure the `genJaxb` task to generate the classes:
```gradle
task genJaxb {
  ext.sourcesDir = "${buildDir}/generated-sources/jaxb"
  ext.classesDir = "${buildDir}/classes/jaxb"
  ext.schema = "${projectDir}/src/main/resources/seev.031.001.07.xsd"
  outputs.dir classesDir

  doLast() {
    project.ant {
      // Create output directories
      mkdir(dir: sourcesDir)
      mkdir(dir: classesDir)

      taskdef name: 'xjc', classname: 'com.sun.tools.xjc.XJCTask', classpath: configurations.jaxb.asPath

      xjc(destdir: sourcesDir, schema: schema, 'package': 'com.gdpotter.sample.iso_20022.seev_031_01_07') {
        produces(dir: sourcesDir, includes: '**/*.java')
      }

      javac(destdir: classesDir, source: 1.8, target: 1.8, debug: true,
            debugLevel: 'lines,vars,source',
            includeantruntime: false,
            classpath: configurations.jaxb.asPath) {
        src(path: sourcesDir)
        include(name: '**/*.java')
        include(name: '*.java')
      }

      copy(todir: classesDir) {
        fileset(dir: sourcesDir, erroronmissingdir: false) {
          exclude(name: '**/*.java')
        }
      }
    }
  }
}
```

In this example, I've placed the schema xsd file in `src/main/resources` but you could have also pointed it to an online address.

Finally, we just need to make sure that the compiled sources are included in the jar and can referenced by the code. We do this by adding the `classesDir` to the `jar` task and also including them as a dependency:

```gradle
dependencies {
  compile(files(genJaxb.classesDir).builtBy(genJaxb))
  jaxb 'com.sun.xml.bind:jaxb-xjc:2.1.7'
}

compileJava.dependsOn 'genJaxb'

jar {
  from genJaxb.classesDir
}
```
To generate the classes, you can run the `jar` task which depends on `genJaxb`:
```
$ ./gradlew jar
```
You can see the results in the `/build/` directory:
```
├── build
│   ├── classes
│   │   └── jaxb
│   │       └── (compiled .class files)
│   ├── generated-sources
│   │   └── jaxb
│   │       └── (generated .java files)
│   ├── libs
│   │   └── gradle-jaxb-example-1.0-SNAPSHOT.jar
...
```

## Adding Multiple Schemas
Let's now take a look at what we would need to do add multiple schemas. We have the seev_031 schema added above, but let's add the seev_039 schema as well and just change it to load all `*.xsd` files:
```gradle
xjc(destdir: sourcesDir, 'package': 'com.gdpotter.sample.iso_20022') {
  schema(dir: "${projectDir}/src/main/resources", includes: '**/*.xsd')
  produces(dir: sourcesDir, includes: '**/*.java')
}
```
However, when trying to run the `genJaxb` Gradle task, we get an errors like the following:
```
[ant:xjc] [ERROR] A class/interface with the same name "com.gdpotter.sample.iso_20022.seev_031_01_07.EventConfirmationStatus1Code" is already in use. Use a class customization to resolve this conflict.
```
This is because the two `xsd` files declare some of the same classes. You'll also notice that both schemas have to be configured for the same package. Even though we can configure multiple schemas, we can only configure a single package. However, by using a `binding.xml` file, we are able to customize the package of each schema.

## The binding.xml file
The [binding file](https://docs.oracle.com/javase/tutorial/jaxb/intro/custom.html) allows for the customization how how the jaxb binding process occurs. We are going to use it to change the package on a per-schema basis as follows:
```xml
<bindings xmlns="http://java.sun.com/xml/ns/jaxb" version="2.1">
  <bindings schemaLocation="seev.031.001.07.xsd">
    <schemaBindings>
      <package name="com.gdpotter.sample.iso_20022.seev_031_01_07" />
    </schemaBindings>
  </bindings>
  <bindings schemaLocation="seev.039.001.07.xsd">
    <schemaBindings>
      <package name="com.gdpotter.sample.iso_20022.seev_039_01_07" />
    </schemaBindings>
  </bindings>
</bindings>
```
And then we can just configure our `genJaxb` task to use that file:
```gradle
xjc(destdir: sourcesDir, binding: "${projectDir}/src/main/resources/binding.xml") {
  schema(dir: "${projectDir}/src/main/resources", includes: '**/*.xsd')
  produces(dir: sourcesDir, includes: '**/*.java')
}
```
