# Dependency Injection in Java, an introduction

## Introduction

In this post I am a going to build a small example of how dependency injection could be built in Java.
It will use Java annotations and be fully working. 

The implementation will be naive and un-optimized, but it should be easy to follow and understand, which is the whole point.

### What is Dependency Injection
Once upon a time came object oriented programming, and Java in particular. 
The introduction of OOP was a real revolution in the programming world, as it, among other, allowed to  
nicely encapsulate functionality and provide a great way to organize your application. 

> One of the major characteristics of an object is that it has to be created (instantiated), and that there's nothing that 
stops you from creating (instantiating) many many objects of the same class. 

At the same time, a specific object can be requested by many different part of the application. For example a database connection object, 
any sort of configuration object, application internal state, ...

Most of the time, only 1 of these objects should be instantiated inside the application.
If you instantiate 2 application states object, which one should you use? In which state is your application, shutting down? 
Or is it receiving requests? No way to know for sure. 

To solve these 2 issues (object available where needed and only 1 instance ), there first was the **singleton** pattern. While it did the job,
it is considered now as an [anti-pattern](https://www.michaelsafyan.com/tech/design/patterns/singleton).

Then came what we now refer to as **dependency injection frameworks**. The basic idea is to delegate the instantiation and management
 of specific object types (In java usually historically referred to as *beans*) to a separate mechanism inside the application.
The framework will instantiate the relevant objects and "inject" then where requested (when another "bean" needs a specific object,
it will look it up and provide it to it).

#### Why this post on Dependency Injection?

The first time you come across one of those big java frameworks, it really feels like magic. 
You annotate a class (with @Bean, @Component, @Controller, whatever ), and then suddenly it just gets created out of the blue, 
dependencies get "injected", without requesting any sort of plumbing ... magic !!

I hope with this small introduction to demystify what dependency injection is, and show that it does not have to feel magical or scary.


## Implementing a Dependency Injection (DI) framework
### What will the framework do
The framework will identify all classes and sub-classes of a base package that are annotated with the **@Bean** annotation.

It will instantiate all these beans and provide them with the requested dependencies. 
Dependencies will be provided through the constructor, which should be unique and be annotated with the **@Inject** annotation.

I will leave the testing aside not to burden this post too much, but tests are available in the github repo 
(which is to be found [here](https://github.com/GaetanGiraud/di-annotations) btw)

### Main components of our framework
What do we need to program our dependency injection framework in Java?
* A registry to store our objects and retrieve them when needed.
* Java annotations, @Bean and @Inject
* A package inspector that identify classes that are annotated with @Bean
* A routine that uses reflection to recursively instantiate classes and their dependencies

### Registry
Our registry will just be a Map that maps a class to an object. 

It is all pretty straightforward, you register an object (with or without specifying the class).
You can check if a class has been instantiated, and reset the registry.

To ensure uniqueness an error is thrown when trying to register an object that already exists.
 
Note that this is a "static" class as the registry needs to be available in the rest
of the application.
```java
public class BeanRegistry {
    private static final Map<Class, Object> registry = new HashMap<>();

    public static void register( Object beanObject){
        register(beanObject.getClass(), beanObject);
    }

    public static void register(Class beanClass, Object beanObject){
        // one can register an object only once
        assert !registry.containsKey(beanClass);

        registry.put(beanClass, beanObject);
    }

    public static Object get(Class beanClass){
        return registry.get(beanClass);
    }

    public static boolean isInstantiated(Class c){
        return registry.containsKey(c);
    }

    public static void reset() {
        registry.clear();
    }
    // Note that there is no "remove" method. Beans are instantiated once at start-up 
    // and then should not be removed
}
```

### Java Annotations

Annotations seem very mysterious at first, you just add them to your classes / methods, and some magic happens!

In practice, they are very easy to set-up (if less to use), here is all we need for our small demonstration:
```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Bean {
}

@Retention(RetentionPolicy.RUNTIME)
public @interface Inject {
}
```
Note the retention policy: for the framework to be able to read the annotations at RUNTIME, this needs to be declared
specifically.


### Package Inspection

A bit more boilerplate here, but this being Java, it should not scare the reader too much.

Behind the hood we use Guava to list classes available in the classpath, and then use reflection to perform the 2 following actions:
* Guava's classpath is used to find the public classes defined inside a package and its sub-packages

```java
ClassPath classpath = ClassPath.from(Thread.currentThread().getContextClassLoader()); 

String packageName = baseClass.getPackage().getName();

Set<ClassPath.ClassInfo> classes = classpath.getTopLevelClassesRecursive(packageName);
```
* Then reflection is used to identify beans ...

```java
    myClass.isAnnotationPresent(Bean.class);
```
* ... and to identify injected dependencies.

Our code relies on there being only 1 constructor available (otherwise which one should we chose?)

```java
   Constructor[] constructors = myClass.getConstructors();

   assert constructors.length == 1; // 1 single constructor should be defined.

   Constructor constructor = constructors[0];

   if(constructor.isAnnotationPresent(Inject.class)) {
       return Arrays.stream(constructor.getParameters())
                    .map(Parameter::getType).collect(Collectors.toList());
   }
```

Full code available on 
[Github](https://github.com/GaetanGiraud/di-annotations/blob/master/src/main/java/com/ggiraud/di/PackageInspector.java).

### Class Instantiator
The instantiator works as follow:
* First it gets the list of the beans that need instantiation from the PackageInspector. If they are not already 
instantiated/registered in the BeanRegistry, it will instantiate them.

As will be explained below, beans will be instantiated recursively to ensure that dependencies can be injected when needed.
As such it is possible that a class has already been instantiated before we actually come across it in our list.
> You could let the instantiation of the class fail, but I find it to be clearer to check specifically for it 
as this is an expected behaviour

```java
Set<Class> classes = PackageInspector.getBeans(baseClass);

for(Class c : classes){
    if(!BeanRegistry.isInstantiated(c)) instantiate(c);
}
```

* The instantiation process of a single class is as follows:
    * We get the number of injected dependencies, 
    * Recover them from the bean registry. 
    * If not instantiated, we instantiate them by calling the instantiate method recursively (see above)
    * Using reflexion we instantiate the class.
    * Finally we store the class in the BeanRegistry

```java
private Object instantiateClass(Class myClass) 
            throws IllegalAccessException, InvocationTargetException, InstantiationException {
    List<Class> injectedClasses = PackageInspector.getInjectedDependencies(myClass);
    
    List<Object> argumentsList = new ArrayList<>();
    
    for(Class c : injectedClasses){
        Object bean = BeanRegistry.get(c);
    
        if(bean == null) {
            bean = instantiate(c);
        }
        argumentsList.add(bean);
    }
    
    Object instantiatedBean = myClass.getConstructors()[0].newInstance(argumentsList.toArray());
    
    BeanRegistry.register(instantiatedBean);
}
```

Full code available on 
[Github](https://github.com/GaetanGiraud/di-annotations/blob/master/src/main/java/com/ggiraud/di/ClassInstantiator.java).

### Wiring it all up together
We now have all the components necessary for our framework. Was't too bad, was it?

In the github repo you will find the [most basic example](https://github.com/GaetanGiraud/di-annotations/tree/master/src/main/java/com/ggiraud/main)

It is composed of 2 classes, one representing a UI, and one representing an Engine that does some calculations.

Both classes are annotated as Beans, and the Engine is injected into the UI by the dependency injection mechanism.

```java
@Bean
public class UI {
    private Engine engine;

    @Inject
    public UI(Engine engine) {
        System.out.println("UI initiated");
        this.engine = engine;
    }

    public void calculate(double x, double y){
        System.out.println("2 by 3 is " + engine.calculate(x, y));
    }
}
```

```java
@Bean
public class Engine {

    public Engine() {
        System.out.println("Engine initiated");
    }

    public double calculate(double x, double y) {
        return x * y;
    }
}
```

If we run the following inside main method (with the UI and Engine class being declared in a subpackage of the main class):
```java
public static void main(String[] args){
    ClassInstantiator instantiator = new ClassInstantiator();
    instantiator.instantiateAll(Application.class);

    UI ui  = (UI) BeanRegistry.get(UI.class);

    ui.calculate(2,3);
}
```

The UI class and its dependency the Engine class are both instantiated automatically and inform us that 2 times 3 equals 6,
confirming that both the UI (doing the print) and the Engine (performing the multiplication) are indeed doing what they should be doing!

You can look at the [tests](https://github.com/GaetanGiraud/di-annotations/tree/master/src/test/java/com/ggiraud/di/tests) as well to get a feeling of the inner workings of the framework.

### The end
That's it for this small intro, I hope it helped demystified the whole concept and that you learnt from it!
