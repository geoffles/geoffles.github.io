---
layout: default
title: Drools Testing with Junit
categories: Testing
tags: [java,junit,drools,testing]
date: 2018-01-18 9:26:00+0200
---

***Abstract:** Unit testing of Drools Rules with JUnit using stateless KIE sessions*

* TOC
{:toc}

# Introduction

As with any other code, Drools code needs to be tested as well. Being able to provide simple unit tests that can not only validate the syntax but also check the expected behaviour can be a very nice to have in your build pipeline.

So what better way to provide unit testing than with all your other JUnit tests?

In order to achieve this, you'll want some repeatable and easy way of setting up your KIE session with some rules, defining some facts, set it to run and inspect the output. A helper object can provide an easy interface into setting up and running everything.

The helper's responsibilities then are:
-  Create the KIE session
-  Load rules into the session
-  Load facts into the session
-  "Execute" the session
-  Capture the resulting working memory and return it

As a matter of preference, I quite like the fluent style interface and so have set up my helper to work something like this:

{% highlight java %}
@Test
public void MyTest() {
    Map<String,Object> facts = new HashMap<>();
    
    //setup some facts...
    
    DrlHelper helper = new DrlHelper()
            .withDrl(this,"MyRules.drl")
            .build();

    DrlHelperOutput results = helper.execute(facts);

    //Some assertions...
}
{% endhighlight %}

# The Helper Object Interface

My helper interface has the following members defined:

{% highlight java %}
public interface DrlHelper {
    DrlHelperOutput execute(Map<String, Object> facts);
    DrlHelper withDrl(Object context, String ruleName);
    DrlHelper build();
}
{% endhighlight %}

The `withDrl` method is part of the fluent interface that will allow me to define the rules I'd like to participate in my test.

The `build` method is needed to explicitly tell the helper that I am done busy definining my session. The build method will actually create the stateless KIE session which means it will also compile all the rules. This means that you can execute the session as many times as you like, and being statless, there is no state that will bleed across executions.

The `execute` method will then load the facts (supplied as an argument), run all the rules loaded into the session and provide you with the working memory in the form of the output object.

The object looks like this:

{% highlight java %} 
public class DrlHelperOutput {
    private List<Object> objects;
    private Map<String, Object> facts;

    public DrlHelperOutput(List<Object> objects, Map<String, Object> facts) {
        this.objects = objects;
        this.facts = facts;
    }

    public List<Object> getObjects() {
        return objects;
    }

    public Map<String, Object> getFacts() {
        return facts;
    }
}
{% endhighlight %}

# Implementing the KIE Session

The stateless KIE session guarantees no shared state between test cases and therefore it's reusability.

Creating the session has been split into two principle parts, namely the `constructor` and the `build` method. The constructor provides the dependencies for calls to `withDrl` which loads rules the rules into a KIE file, while the `build` method compiles your rule files, creates the session and is also the ideal opportunity to add any event listeners that you'd like for tracing.

## The Constructor
As mentioned the constuctor establishes some dependencies. These are the `KieServices` and `KieFilesystem` that will be used later. The code is:

{% highlight java %} 
public DrlHelper() {
    kieServices = KieServices.Factory.get();
    kieFileSystem = kieServices.newKieFileSystem();
}
{% endhighlight %}


## WithDrl

As mentioned, the `withDrl` is the fluent interface by which rules are loaded into the session. At this point there techincally isn't any session, but there is a "file system" from which the session will be loaded.

Files are loaded by retrieving a jar resource and writing it to the KieFilesystem as a DRL file type. For convenience, a context object simplifies resource retrieval - since the test should be in the same package as the rule and you can pass the test object as the context.

Thus the method looks like:

{% highlight java %} 
public DrlHelper withDrl(Object context, String ruleName){
    URL resource = context.getClass().getResource(ruleName);
    if (resource == null){
        throw new IllegalArgumentException("ruleName '" + ruleName + "' does not resolve to a resource");
    }

    File ruleFile = new File(resource.getPath());
    kieFileSystem.write(ResourceFactory.newFileResource(ruleFile).setResourceType(ResourceType.DRL));
    return this;
}
{% endhighlight %}

## Build

As mentioned, the `build` method completes the plumbing for creating your Kie session from your now loaded rules. What remains to be done is to create a `KieBuilder` (and compile the rules) and a `KieSession`. The `KieBuilder` can be acquired from your already created `KieServices`. The `KieSession` is acquired from a container which in turn requires a `ReleaseId` which is then acquired from a `KieRepository`.

Compiling the rules is an expensive operation, so you might also want to restrict it from being called more than once. You'll also want to report an "build" errors from your rules.

So your `build` method might look like:

{% highlight java %} 
public DrlHelper build() {
    if (kSession != null){
        throw new IllegalStateException("Kie Session has already been built.");
    }

    KieBuilder kieBuilder = kieServices.newKieBuilder(kieFileSystem);
    kieBuilder.buildAll();

    if (kieBuilder.getResults().hasMessages(Message.Level.ERROR)) {
        throw new RuntimeException("Build Errors:\n" + kieBuilder.getResults().toString());
    }

    KieRepository kieRepository = kieServices.getRepository();
    KieContainer kContainer = kieServices.newKieContainer(kieRepository.getDefaultReleaseId());

    kSession = kContainer.newStatelessKieSession();



    return this;
}
{% endhighlight %}

# Running the Rules

Being a stateless session, each execution must happen as a collection of commands, which includes the loading of facts into the working memory. The `CommandFactory` has a collection of factory methods to create the commands needed to set up your stateless execution.

First you will need an insert command for each fact you'd like in working memory. A simple way to supply facts is with a string keyed map object. Next you will need a fire all rules command that will instruct the session to evaluate rules on your working memory. Your commands should be stored in a list object as the order of execution is obviously important.

You may also want to acquire all objects out of your working memory for inspection by your unit tests. This can be done by supplying get objects command in the list. You will need an identifier key for this "query" result which you may want to store as a constant in your helper.

Finally you need to create a batch execution command into which you'll pass your list of commands for execution to the session. The execute method will return an `ExecutionResults` object that will allow you to now examine all loaded facts and retrieve your "all objects" query result (if you supplied one).

The above work is then implemented like so:

{% highlight java %} 
public DrlHelper execute(Map<String, Object> facts) {
    if (kSession == null){
        throw new IllegalStateException("No Kie Session defined. Did you call 'build()'?");
    }

    List<Command> commands = new ArrayList<>();

    for (Map.Entry<String, Object> entry : facts.entrySet()) {
        Command insertFactCommand = CommandFactory.newInsert(entry.getValue(), entry.getKey());
        commands.add(insertFactCommand);
    }

    commands.add(CommandFactory.newFireAllRules());
    commands.add(CommandFactory.newGetObjects(GET_OBJECTS_KEY));

    ExecutionResults executionResults = kSession.execute(CommandFactory.newBatchExecution(commands));
    Map<String, Object> results = new HashMap<>(executionResults.getIdentifiers().size());

    for (String identifier : executionResults.getIdentifiers()) {
        results.put(identifier, executionResults.getValue(identifier));
    }

    return new DrlHelperOutput( (List<Object>)(executionResults.getValue(GET_OBJECTS_KEY)), results);
}
{% endhighlight %}


# Bringing it All Together

Once you're done, you now have a helper that should look something like this:

{% highlight java %} 
public class DrlHelper {
    private static final String GET_OBJECTS_KEY = "_getObjects";

    private  StatelessKieSession kSession;
    private final KieFileSystem kieFileSystem;
    private final KieServices kieServices;

    public DrlHelper() {
        kieServices = KieServices.Factory.get();
        kieFileSystem = kieServices.newKieFileSystem();
    }

    public DrlHelperOutput execute(Map<String, Object> facts) {
        if (kSession == null){
            throw new IllegalStateException("No Kie Session defined. Did you call 'build()'?");
        }

        List<Command> commands = new ArrayList<>();

        for (Map.Entry<String, Object> entry : facts.entrySet()) {
            Command insertFactCommand = CommandFactory.newInsert(entry.getValue(), entry.getKey());
            commands.add(insertFactCommand);
        }

        commands.add(CommandFactory.newFireAllRules());
        commands.add(CommandFactory.newGetObjects(GET_OBJECTS_KEY));

        ExecutionResults executionResults = kSession.execute(CommandFactory.newBatchExecution(commands));
        Map<String, Object> results = new HashMap<>(executionResults.getIdentifiers().size());

        for (String identifier : executionResults.getIdentifiers()) {
            results.put(identifier, executionResults.getValue(identifier));
        }

        return new DrlHelperOutput( (List<Object>)(executionResults.getValue(GET_OBJECTS_KEY)), results);
    }

    public DrlHelper withDrl(Object context, String ruleName){
        URL resource = context.getClass().getResource(ruleName);
        if (resource == null){
            throw new IllegalArgumentException("ruleName '" + ruleName + "' does not resolve to a resource");
        }

        File ruleFile = new File(resource.getPath());
        kieFileSystem.write(ResourceFactory.newFileResource(ruleFile).setResourceType(ResourceType.DRL));
        return this;
    }    

    public DrlHelper build() {
        if (kSession != null){
            throw new IllegalStateException("Kie Session has already been built.");
        }

        KieBuilder kieBuilder = kieServices.newKieBuilder(kieFileSystem);
        kieBuilder.buildAll();

        if (kieBuilder.getResults().hasMessages(Message.Level.ERROR)) {
            throw new RuntimeException("Build Errors:\n" + kieBuilder.getResults().toString());
        }

        KieRepository kieRepository = kieServices.getRepository();
        KieContainer kContainer = kieServices.newKieContainer(kieRepository.getDefaultReleaseId());

        kSession = kContainer.newStatelessKieSession();

        return this;
    }
}
{% endhighlight %}

# Conclusion

Creating a helper object that encapsulates the setup, loading, and execution of rules means that you can focus on testing your rule outputs instead of running the rules - in fact it's almost surprising that there didn't seem to be any built in one-shot helper object for setting this stuff up.

This will work best when you load the fewest neccesary rules needed to actually test your rules. You can also add other `with*` methods for any other resources you'd like to add to the session when executing (such as BPMN2 resources).

Using a fluent interface means that your tests follow a natural-ish language which also makes it easier for others to read and understand.

Because creating the helper is expensive by virtue of the rule building for the session, you may want to hold onto your helper instances for all tests that will be run the rules you passed in.