---
title: Custom Annotations in Android
author: Igor Brishkoski
date: 2017-04-17 14:10:00 +0800
categories: [Tutorial, Android, Java]
tags: [Android, Annotations]
---

## Annotations in Java 1.5

Before java annotations, program metadata was available through java comments or by javadoc, but annotations offer more than that.

Java Annotations are introduced in Java 1.5, and now they are heavily used in Java frameworks including Android.
Java Annotations are metadata about the program embedded in the program itself,
but annotations have no direct effect on the operation of the code they annotate (i.e., it does not affect the execution of the program).
They can provide compile-time instructions to the compiler that can be further used by software build tools for generating code, XML files, etc.
They can be associated with classes, methods, fields, parameters and even other annotations.

Java 1.5 offers 3 built-in annotations.

`@Override` which is used to tell the compiler that we’re overriding a method, it makes the code readable and maintainable,
it also helps with avoiding issues when the signature of the method that we’re overriding is changed.

```java
public class MyParentClass {
    public void justaMethod() {
        System.out.println("Parent class method");
    }
}

public class MyChildClass extends MyParentClass {
    @Override
    public void justaMethod() {
        System.out.println("Child class method");
    }
}
```

`@Deprecated` annotation marks the annotated element (class, field, method) as deprecated and indicates that it should not be used.
The compiler then issues a warning when that element is used.
Good practice is to document the deprecated element with javadoc on the reason of why it was deprecated.

```java
/**
 * @deprecated
 * reason for why it was deprecated
 */
@Deprecated
public void anyMethodHere(){
    // Do something
}
```

`@SuppressWarnings` is used to silence the compiler whenever it gives us those annoying deprecated or other warnings.

```java
@SuppressWarnings("deprecation")
public void myMethod() {
    myObject.deprecatedMethod();
}
```
Java 1.7 and 1.8 bring a few more built-in annotations.

## Android Support Annotations Library

BIn addition to the already built-in java annotations, Android, through the
android support library provides additional annotations.
If you’re interested in more, visit the [library page](https://developer.android.com/studio/write/annotations.html).

## Creating a Custom Annotation

We learned about the annotations that are built in java, and we saw that we can include 3rd party annotations in our project,
now let’s see how we can build our own custom annotation.

Java annotations are just interfaces prefixed with `@` sign. Easy enough, let’s create our first custom annotation.
We’re going to call it `MethodInfo` and it will provide some basic info for the method the developers are building.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface MethodInfo {
    String author() default "Igor Brishkoski";
    int revision() default 1;
    String comments();
}
```

That’s pretty much it. Now let’s go over the code line by line and see what’s going on.

As we can see, we have several annotations on our interface.
`@Target` Specifies the elements on which this annotation can be used.
In this case, we have `ElementType.METHOD` since we’re going to be annotating only methods.
Other elements that can be annotated are:

```java
ElementType.METHOD
ElementType.PACKAGE
ElementType.PARAMETER
ElementType.TYPE
ElementType.ANNOTATION_TYPE
ElementType.CONSTRUCTOR
ElementType.LOCAL_VARIABLE
ElementType.FIELD
```


`@Retention` annotation tells the compiler when will it be needed.

`RetentionPolicy.RUNTIME` The annotation should be available at runtime, for inspection via java reflection.
`RetentionPolicy.CLASS` The annotation would be in the .class file, but it would not be available at runtime.
`RetentionPolicy.SOURCE` The annotation would be available in the source code of the program, it would neither be in the .class file nor be available at the runtime.

`@Documented` annotation indicates that elements using this annotation will be documented by JavaDoc. When we’re creating the docs, the annotation will be included in the docs.

`@Inherited` annotation tells the compiler that all subclasses of the annotated class will inherit the annotation.

Great! We now have our own custom annotation, let’s see it in action.

```java
@MethodInfo(author = "John Snow", revision = 2, comments = "Hey!")
public void awesomeMethod() {
    Method method = getClass().getMethod("awesomeMethod");
    MethodInfo methodInfo = method.getAnnotation(MethodInfo.class);

    Log.d("MethodInfo", methodInfo.author());
    Log.d("MethodInfo", methodInfo.revision());
    Log.d("MethodInfo", methodInfo.comments());
}
```

We have annotated the `awesomeMethod()` with our custom annotation, and since we set the retention policy to be at runtime,
we can access the values of the annotation.
This is great, but the true power of annotations comes from compile time code generation.

## Annotation Processor

Let’s see how we can create our own processor and include that processor in our app.

You can check code on GitHub [https://github.com/igor-brishkoski/AwesomeApp/](https://github.com/igor-brishkoski/AwesomeApp/)

### Our Problem
Let’s say we’re developing a lot of android apps. We’re using logcat to debug our apps and to see what’s going on during development,
so we want to make logging models in our projects easier and more standardized.
Our new app AwesomeApp has a model called `User` which has several fields like `firstName`, `lastName` and `city`
and if we want to log the values of the class to `Logcat`, our code would look something like this.

```java
//AwesomeApp
class User {
    String firstName;
    String lastName;
    String city;
}
...
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        User g = new User("Gandalf", "The White", "Maia");

        Log.d("User", "firstName = "+g.firstName+ " lastname = "+g.lastName+" city = "+g.city);
    }
}
```

And by adding more fields in our model, the log statement will only get bigger.

For our solution, we’re going to annotate our model class with our custom annotation and create a logger, using our annotation processor, based on the fields in our model.
### Our Custom Annotation

For our custom annotation, we’re going to create a separate module in our project and add the module as a dependency to our app.
Go to `File -> New -> New Module...` in Android Studio.
Select `Java Library` and name the module `annotation`

You’ll notice a new folder in your project structure with its own `build.gradle` file.
You will also notice that our new module has been added to our settings.gradle file,
but in order for that module to be available in our app, we need to include it as a dependency in our app `build.gradle` file.

```groovy
//our app build.gradle file
dependencies {
    ...

    provided project(':annotation')
    ...
}
```

Great! Let’s create our custom annotation `AwesomeLogger` in the new module that we just created.
We will use it to annotate our models for which we want to create loggers for.

```java
// annotation module
package com.example.annotation;

@Target(ElementType.TYPE) //class level
@Retention(RetentionPolicy.SOURCE) //we only need it at compile time
public @interface AwesomeLogger {}
```

We’re setting the target to be `ElementType.TYPE` which represents a class or interface program element,
and we’re setting the retention policy to be `RententionPolicy.SOURCE` meaning we will only need the annotations during compile time.

Now we can annotate the User model in our app.

### Our Annotation Processor

What is Annotation Processor?

Annotation processor is a tool built in javac for scanning and processing annotations at compile time.
This means that we can register our custom annotation `AwesomeLogger` to be picked up by the processor,
and the processor can generate `.java` files for us containing the code necessary for our logger helper class.

For our Annotation Processor, we’re going to create a separate module in our project and add the module as a dependency to our app.

Go to `File -> New -> New Module...` in Android Studio. Select `Java Library` and name the module `processor`

Create a class `AwesomeLoggerProcessor` and have that class extend the `AbstractProcessor` class.
Every processor has to extend from the `AbstractProcessor`

Let’s look at some of the APIs we’re going to use.

```java
public class AwesomeLoggerProcessor extends AbstractProcessor {
    @Override
    public synchronized void init(ProcessingEnvironment env){ }

    @Override
    public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }

    @Override
    public Set<String> getSupportedAnnotationTypes() { }

    @Override
    public SourceVersion getSupportedSourceVersion() { }
}
```
- `init` — The processor has to have an empty constructor. The init method is provided to us to help us instantiate everything
that we need to get the party started. We’re being passed an instance of `ProcessingEnvironment` that provides a lot of
useful utility classes that we’re going to use such as the `Filer` for generating files, the `Messager` for helping us with error handling and more.
- `process` — Is where the magic happens. Here we’re going to process all annotations that we care about, like the `AwesomeLogger`
- `getSupportedAnnotationsTypes` — Is where we specify the annotations that this processor will be handling in the process method, in our case that’s the `AwesomeLogger` class.
- `getSupportedSourceVersion` — Is where we specify the Java version. Most of the time we go with the latest supported.

### Registering Our Processor

The cool thing to know about the processor is that it runs in a separate **JVM** instance.
This means that the `javac` starts a whole new process just for our processor,
but in order for our processor to be detected by the `javac` we need to register it with the `ServiceLoader`.

To register our processor, we’re going to use Google’s `AutoService` annotation. Whaaa!?!?! Mind... Blown.
We can use annotations in our annotation processor? Yes, we sure can.
We should treat our annotation processor as we treat our apps, that means that we should architect it properly,
we can include dependencies to help us out with development, we should test it, etc…

Let’s include the dependency in our processor’s module build.gradle file.

```groovy
//processor module
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile project(':annotation')

    compile 'com.google.auto.service:auto-service:1.0-rc2'
}
```
Now we can annotate our processor with `AutoService` and that will do all the heavy lifting for us.
Here we have a little preview of our processor so far.

```java
@AutoService(Processor.class) // 1
public class AwesomeLoggerProcessor extends AbstractProcessor {
  private static final String KEY_PARAM_NAME = "args";
  private static final String METHOD_LOG = "log";
  private static final String CLASS_SUFFIX = "_Log";
  private Messager messager;
  private Filer filer;

  @Override
  public synchronized void init(ProcessingEnvironment processingEnvironment) {
      super.init(processingEnvironment);
      messager = processingEnvironment.getMessager(); // 2
      filer = processingEnvironment.getFiler();
  }
  @Override
  public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
  ...
  }

  @Override
  public SourceVersion getSupportedSourceVersion() {
      return SourceVersion.latestSupported(); // 3
  }

  @Override
  public Set<String> getSupportedAnnotationTypes() {
      Set<String> annotations = new HashSet<>();
      annotations.add(AwesomeLogger.class.getCanonicalName()); // 4

      return annotations;
  }
}
```

1. We registered it with the `AutoService` annotation to be picked up by `javac`.
2. We got our util classes in the init method.
3. We’re going to support the latest java version.
4. And we only care about the `AwesomeLogger` in this processor.

### Processing our Annotations
We can finally start to process our annotated elements.

```java
@Override
public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
    //get all elements annotated with AwesomeLogger
    Collection<? extends Element> annotatedElements = roundEnvironment.getElementsAnnotatedWith(AwesomeLogger.class);

    for (Element type : annotatedElements) {
      ...
    }

    return true;
}
```

On line 4, with the help of the `RoundEnvironment` instance provided to us, we’re getting a collection of elements
annotated with our `AwesomeLogger` annotation. The problem is we’re getting **ALL** elements not just the ones that we
set the target for with `@Target(ElementType.TYPE)`, this means if a developer who is using our custom annotations,
annotates a class field `ElementType.FIELD` for example, we will get that element in our collection.
This is not a problem with modern IDEs because they will immediately start complaining about it.
However, we don’t know if everyone using our annotation will be using a modern IDE, it’s always better to be safe than sorry.

```java
package com.example;		// PackageElement

public class User {		// TypeElement

	private String firstName;// VariableElement
	private String lastName; // VariableElement

	public User () {} 	// ExecuteableElement
}
```
We can use `guava` to help us with the filtering of the elements that we care about.

```java
@Override
public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
    //get all elements annotated with AwesomeLogger
    Collection<? extends Element> annotatedElements = roundEnvironment.getElementsAnnotatedWith(AwesomeLogger.class);

    //filter out elements we don't need
    List<TypeElement> types = new ImmutableList.Builder<TypeElement>()
            .addAll(ElementFilter.typesIn(annotatedElements))
            .build();

    for (TypeElement type : types) {
      ...
    }

    return true;
}
```

That’s it, we got the elements that we want right? Wrong! Remember when we said that `**TYPE**` element can be either a class or an interface.
What if it’s abstract or private? Luckily, we can get information for the element, and we can use that information to decide whether that class is valid for us.
For that, we’re going to create a helper method called `isValidClass` to check if the element meets our criteria.

```java
private boolean isValidClass(TypeElement type){
    if(type.getKind() != ElementKind.CLASS){
        messager.printMessage(Diagnostic.Kind.ERROR,type.getSimpleName()+" only classes can be annotated with AwesomeLogger");
        return false;
    }

    if(type.getModifiers().contains(Modifier.PRIVATE)){
        messager.printMessage(Diagnostic.Kind.ERROR,type.getSimpleName()+" only public classes can be annotated with AwesomeLogger");
        return false;
    }

    if(type.getModifiers().contains(Modifier.ABSTRACT)){
        messager.printMessage(Diagnostic.Kind.ERROR,type.getSimpleName()+" only non abstract classes can be annotated with AwesomeLogger");
        return false;
    }

    return true;
}
```

### Error Handling
Since the processor is running in its own `JVM`, throwing an exception will cause the JVM to crash and deliver a confusing, useless error message to the developer.
For that reason, we got an instance of the `Messager` in the `init` method.
As you can see above in our helper method, we use the `messager` to deliver readable error messages to the developer who is misusing our annotation without crashing the `JVM`.
In modern IDEs you can also jump right to the place where the error was caused, pretty handy.

### Code Generation

Now we can finally start generating code, the reason we created our custom annotation and the annotation processor.
We can use the instance of the `Filer` that we got in our `init` method, however when it comes to source java code
generation `JavaPoet` by `Square` (shocking right? who would have guessed that we’ll be using a library made by Square in our project.
These guys have done so much for the Android development community that I’m tempted to offer my firstborn to them as a thank-you. OK, back to our code)
is the default standard.

```java
private void writeSourceFile(TypeElement originatingType) {
    //get Log class from android.util package
    //This will make sure the Log class is properly imported into our class
    ClassName logClassName = ClassName.get("android.util", "Log");

    //get the current annotated class name
    TypeVariableName typeVariableName = TypeVariableName.get(originatingType.getSimpleName().toString());

    //create static void method named log
    MethodSpec log = MethodSpec.methodBuilder(METHOD_LOG)
            .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
            .returns(void.class)
            //Parameter variable based on the annotated class
            .addParameter(typeVariableName, KEY_PARAM_NAME)
            //add a Lod.d("ClassName", String.format(class fields));
            .addStatement("$T.d($S, $L)", logClassName, originatingType.getSimpleName().toString(), generateFormater(originatingType))
            .build();

    //create a class to wrap our method
    //the class name will be the annotated class name + _Log
    TypeSpec loggerClass = TypeSpec.classBuilder(originatingType.getSimpleName().toString() + CLASS_SUFFIX)
            .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
            //add the log statetemnt from above
            .addMethod(log)
            .build();
    //create the file
    JavaFile javaFile = JavaFile.builder(originatingType.getEnclosingElement().toString(), loggerClass)
            .build();

    try {
        javaFile.writeTo(filer);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

### Using The Annotation

We’re finally ready to use the annotation in our app to help us generate the logging method for our `User` class.
Let’s rebuild our project so that the code can be generated. Now we can open our generated class `User_Log` and see what’s inside.

```java
import android.util.Log;

public final class User_Log {
  public static void log(User args) {
    Log.d("User", String.format("firstName - %s lastName - %s city - %s ", args.firstName, args.lastName, args.city));
  }
}
```

We have a static method called `log(User user)` that accepts an instance of our `User` model and uses Android’s `Log` class
to output the contents of the model into the logcat.

In our `MainActivity` we can call our generated class method `User_Log.log(g)` and watch our beautiful logs in logcat.

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        User g = new User("Gandalf", "The White", "Maia");

        User_Log.log(g);
    }
}
```

And that’s pretty much it. Of course, this is a really basic example just to get you started,
but we are all aware of the power that annotations bring to the table.
Take a look at any of the popular libraries that we use like Butterknife, Dagger, AutoValue and a bunch of others.
