---
layout: post
title: Analyzing Circular Bean Dependencies in Spring
comments: true
github: https://gist.github.com/gdpotter/55e68308ac191308c294a2cc42e596b2
crosspost_to_medium: true
---
It has been well documented that [circular dependencies are an anti-pattern](https://en.wikipedia.org/wiki/Circular_dependency#Problems_of_circular_dependencies). It results in tight coupling and is a smell that your application might be utilizing the [big ball of mud architecture](https://en.wikipedia.org/wiki/Big_ball_of_mud). However, it is very rarely as simple as `ClassA` depends on `ClassB` and `ClassB` depends on `ClassA`. In medium-large Spring projects, these cycles might be 10+ beans in size and difficult to find.

Moving to constructor injection is seen as a best-practice for many reasons but one is that it doesn't allow for circular dependencies to exist. In field/setter-injection, Spring can first construct all of the beans and then wire them up after. Constructor injection combines those two steps and requires all dependencies to already be instantiated before it can inject/create the bean. And while moving to constructor injection can be great for new projects, it doesn't directly solve the problem of first identifying the circular dependencies in an existing application.

To find the cycles, first we need to identify all of the beans and their dependencies. That can be accomplished by iterating over all the bean definitions in the `ListableBeanFactory`:
```java
public class DepGraph {
    private final Map<String, List<String>> nodes;

    private DepGraph(Map<String, Set<String>> nodes) {
        this.nodes = nodes;
    }

    public static DepGraph buildDependencyGraph(GenericApplicationContext applicationContext) {
        ListableBeanFactory factory = applicationContext.getBeanFactory();
        Map<String, Set<String>> beanDeps = Arrays.stream(factory.getBeanDefinitionNames())
            .filter(beanName -> !factory.getBeanDefinition(beanName).isAbstract())
            .collect(Collectors.toMap(
                Function.identity(),
                beanName -> Arrays.asList(factory.getDependenciesForBean(beanName))
            ));

        return new DepGraph(beanDeps);
    }
}
```

Next, we will perform a depth-first search of the graph. Is this the cleanest, most elegant DFS on the internet? No, but it worked for my use case. If anyone has any suggestions on how I can make stylish, please let me know in the comments below:
```java
public Set<String> calculateCycles() {
    Set<String> visited =  new HashSet<>();
    return nodes.keySet().stream()
        .map(node -> calculateCycles(node, visited, Collections.emptyList()))
        .flatMap(Set::stream)
        .collect(Collectors.toSet());
}

private Set<String> calculateCycles(String node, Set<String> visited, List<String> path) {
    if (visited.contains(node)) {
        return Collections.emptySet();
    }

    List<String> newPath = new LinkedList<>(path);
    newPath.add(node);
    visited.add(node);

    if (path.contains(node)) {
        List<String> cycle = newPath.subList(path.indexOf(node), path.size());
        return Collections.singleton(String.join("->", cycle));
    }
    List<String> deps = nodes.getOrDefault(node, Collections.emptyList());
    return deps.stream()
        .map(dep -> calculateCycles(dep, visited, newPath))
        .flatMap(Set::stream)
        .collect(Collectors.toSet());
}
```

We begin by creating a `HashSet` to keep track of all the bean names that have been already visited in the search. Since the graph likely has cycles, we wouldn't want to get stuck in an endless loop. We then begin iterating through the map invoking the next method for each entry. For each entry, we recursively explore all of its dependencies while keeping track of the path we took to get there. If the new node is already in that path, then we have found a cycle.

I'm just returning a collection of `String` (ie `"A->B->C->D->A"`) but you can adapt that to return a List or real class depending on your use case case.

I found this to be very helpful in identifying problematic portions of my codebase. My team is slowly working on untangling this web of beans
and working to avoid these in the future.
