---
layout: post
title:  "The strange case of never-ending Java application"
date: 2020-09-18 20:46
categories: it java spring
---

In this post I will try to keep a record of my attempts to approach and resolve a 
not-too-obvious problem: problem of a Java application, that seems to not be able to finish execution.

So, I had a Spring Boot-based app, that was working just fine, as a CLI app, that had Spring Data JPA integration, 
picocli integration, and whatnot. And I had to add an `rdf4j` support into this app, to be able to execute SPARQL 
queries against WikiData. I already had working standalone app to run those queries, so I assumed this to be a 30 
minutes task. 

Little did I know... 

So, how the original app was running? A typical output would look like this: 

```
2020-09-18 21:09:10.880  INFO 14892 --- [           main] l.lockservice.StandardLockService        : Successfully released change log lock
2020-09-18 21:09:11.066  INFO 14892 --- [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
2020-09-18 21:09:11.136  INFO 14892 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 5.4.15.Final
2020-09-18 21:09:11.331  INFO 14892 --- [           main] o.hibernate.annotations.common.Version   : HCANN000001: Hibernate Commons Annotations {5.1.0.Final}
2020-09-18 21:09:11.433  INFO 14892 --- [           main] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.hibernate.dialect.SQLiteDialect
2020-09-18 21:09:12.276  INFO 14892 --- [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000490: Using JtaPlatform implementation: [org.hibernate.engine.transaction.jta.platform.internal.NoJtaPlatform]
2020-09-18 21:09:12.284  INFO 14892 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2020-09-18 21:09:13.689  INFO 14892 --- [           main] e.mkv.estonian.EstonianFormsApplication  : Started EstonianFormsApplication in 6.732 seconds (JVM running for 7.271)
2020-09-18 21:09:13.709  INFO 14892 --- [           main] ee.mkv.estonian.command.PronounsCommand  : Starting processing command 'pronoun'
2020-09-18 21:09:14.692  INFO 14892 --- [           main] ee.mkv.estonian.command.PronounsCommand  : Finished processing command 'pronoun'
2020-09-18 21:09:14.704  INFO 14892 --- [extShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
2020-09-18 21:09:14.707  INFO 14892 --- [extShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
2020-09-18 21:09:14.710  INFO 14892 --- [extShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
```

Now, when I've added a "wikidata" command support, and executed it, the output would look as follows: 

```
2020-09-18 21:11:32.660  INFO 2684 --- [           main] e.mkv.estonian.EstonianFormsApplication  : Started EstonianFormsApplication in 6.913 seconds (JVM running for 7.45)
2020-09-18 21:11:32.664  INFO 2684 --- [           main] ee.mkv.estonian.command.WikiCommand      : Starting processing command 'wikidata'
2020-09-18 21:11:32.746 DEBUG 2684 --- [           main] org.hibernate.SQL                        : select partofspee0_.id as id1_8_, partofspee0_.eki_codes as eki_code2_8_, partofspee0_.part_of_speech as part_of_3_8_, partofspee0_.wikidata_code as wikidata4_8_ from parts_of_speech partofspee0_ where partofspee0_.part_of_speech=?
2020-09-18 21:11:32.796 DEBUG 2684 --- [           main] org.hibernate.SQL                        : select representa0_.id as id1_9_, representa0_.representation as represen2_9_ from representations representa0_ where representa0_.representation=?
2020-09-18 21:11:33.806  INFO 2684 --- [           main] ee.mkv.estonian.command.WikiCommand      : No such lexeme has been found
2020-09-18 21:11:33.806  INFO 2684 --- [           main] ee.mkv.estonian.command.WikiCommand      : Finished processing command 'wikidata'

> Task :EstonianFormsApplication.main() FAILED

Execution failed for task ':EstonianFormsApplication.main()'.
> Build cancelled while executing task ':EstonianFormsApplication.main()'
```

NB: this `Build cancelled` is here because I basically have to stop an application (it's run in an IntelliJ IDEA as an 
`Application`-based Run Configuration, by the way), in order to be able to proceed with my work. Otherwise the app would just hang forever.

Hmmmm.. what on Earth could have caused such behaviour? 

This is the code I've added: 

```java
@Component
@Slf4j
public class WikidataUploader {

    private final static BiFunction<PartOfSpeech, Representation, String> QUERY_LEXEME = (grammaticalCategoryId, lemma) -> String.format("SELECT ?lexeme ?lemma WHERE {\n" +
            "  ?lexeme dct:language wd:Q9072;wikibase:lexicalCategory wd:%s;wikibase:lemma ?lemma.\n" +
            "  FILTER (STR(?lemma)=\"%s\")}", grammaticalCategoryId.getWikidataCode(), lemma.getRepresentation());

    private final QueryExecutor executor;

    public WikidataUploader(QueryExecutor queryExecutor) {
        this.executor = queryExecutor;
    }

    public Optional<String> checkLexeme(Lexeme lexeme) {
        String query = QUERY_LEXEME.apply(lexeme.getPartOfSpeech(), lexeme.getLemma());
        final Long lexemeId = lexeme.getId();

        try (TupleQueryResult lexemesResult = executor.executeQuery(query)) {
            List<BindingSet> results = lexemesResult.stream().collect(Collectors.toList());
        } catch (Throwable error) {
            log.error("Error while checking lexeme {}: {}", lexemeId, error.getMessage(), error);
            throw new RuntimeException(error);
        }

        return Optional.empty();
    }

    public String saveLexeme(Lexeme lexeme) {
        throw new NotImplementedException("Saving lexemes is not yet implemented");
    }
}
```

and this is `QueryExecutor`: 

```java
@Component
public class QueryExecutor implements DisposableBean {
    private static final String BASE_URI = "https://query.wikidata.org/sparql";

    private final HttpClient httpClient = HttpClients.custom()
            .setDefaultRequestConfig(
                    RequestConfig
                            .custom()
                            .setCookieSpec(CookieSpecs.STANDARD)
                            .build()
            )
            .build();

    private final RDF4JProtocolSession session = new RDF4JProtocolSession(httpClient, Executors.newScheduledThreadPool(1));
    private final Dataset dataset = new SimpleDataset();

    public QueryExecutor() {
        session.setRepository(BASE_URI);
    }

    public TupleQueryResult executeQuery(String query) throws IOException {
        return session.sendTupleQuery(QueryLanguage.SPARQL, query, BASE_URI, dataset, false, 10);
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("Query executor is closing");
        session.close();
        ((CloseableHttpClient) httpClient).close();
        System.out.println("Query executor is closed");
    }
}
```

That's certainly not the most elegant RDF4j code, but mind you - it works well enough in the other application (where 
it's not a Spring component, though). So, what could be the culprit? And how do I pin it down? 

--end of episode 1 (2020-09-18 21:27 EEST)--

Seen that `destroy()` method above? Rest assured, initially it wasn't there, I've only added this while trying
 (sheepishly) to mitigate the issue. It didn't change anything, though.
 
So, what could be done next? Looking at the log output for "normal" execution, we find messages from JPA and connection 
pool, so let's get rid of those first. 

Let's create simple empty Spring Boot project without any dependencies: 

```
mkdir spring-rdf4j
cd spring-rdf4j
curl https://start.spring.io/starter.zip -d type=gradle-project -o demo.zip | unzip demo.zip
```

Then let's add the RDF4j dependency: 

```
	implementation group: 'org.eclipse.rdf4j', name: 'rdf4j-repository-sparql', version: '3.2.2'
```

and copy-paste the `QueryExecutor` class as is. Also, let's have our main app class implement CLI and execute 
SPARQL-query upon running the app: 

```java
package com.example.demo;

import com.example.demo.wikidata.QueryExecutor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication implements CommandLineRunner {

	@Autowired
	QueryExecutor queryExecutor;

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

	@Override
	public void run(String... args) throws Exception {
		System.out.println("Starting application");
		queryExecutor.executeQuery("SELECT ?lexeme ?lemma WHERE {\n" +
						"  ?lexeme dct:language wd:Q9072;wikibase:lexicalCategory wd:Q1084;wikibase:lemma ?lemma.\n" +
						"  FILTER (STR(?lemma)=\"aprill\")\n" +
						"}");
		System.out.println("Exiting application");
	}
}
```

Now we can run the app, aaaaannddd: bingo! It still hangs! After producing this: 

```
2020-09-21 19:47:28.346  INFO 14832 --- [           main] com.example.demo.DemoApplication         : Starting DemoApplication on DESKTOP-TSSQSTI with PID 14832 (C:\Develop\Java\spring-rdf4j\build\classes\java\main ...)
2020-09-21 19:47:28.349  INFO 14832 --- [           main] com.example.demo.DemoApplication         : No active profile set, falling back to default profiles: default
2020-09-21 19:47:29.944  INFO 14832 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 1.983 seconds (JVM running for 2.489)
Starting application
Exiting application
```

That's good! I love it when things are reproduced with minimal setup needed. Let's if I'll want to take these words back, though...

Ok, now I'd want to try it out without an IDE. Stupid, you say? Nah, probably so. I just want to be sure I am not impacted
 by anything too smart from JetBrains here.. So, I run `gradlew bootJar` and then `java -jar build/libs/demo-0.0.1-SNAPSHOT.jar`
 
Ok, it was stupid indeed. The behaviour is intact. 

Let's now follow some tips from [this article][thread-dump]. While the JAR is hanging forever, `jps` command will give 
us PID of the process we're after (4756, in my case). And `jstack -l 4756` will give us.. something. Let's see what we have here: 

```
2020-09-21 19:59:17
Full thread dump OpenJDK 64-Bit Server VM (11.0.8+10 mixed mode):

Threads class SMR info:
_java_thread_list=0x00000176ae6a6c70, length=11, elements={
0x00000176ac2d5000, 0x00000176ac2de800, 0x00000176ac334000, 0x00000176ac336000,
0x00000176ac337000, 0x00000176ac33b000, 0x00000176ac33e000, 0x00000176ac4ac000,
0x00000176ac4ad800, 0x00000176ac4b0800, 0x000001768876a800
}

"Reference Handler" #2 daemon prio=10 os_prio=2 cpu=0.00ms elapsed=126.57s tid=0x00000176ac2d5000 nid=0x3bcc waiting on condition  [0x0000004acb3fe000]
   java.lang.Thread.State: RUNNABLE
	at java.lang.ref.Reference.waitForReferencePendingList(java.base@11.0.8/Native Method)
	at java.lang.ref.Reference.processPendingReferences(java.base@11.0.8/Reference.java:241)
	at java.lang.ref.Reference$ReferenceHandler.run(java.base@11.0.8/Reference.java:213)

   Locked ownable synchronizers:
	- None

"Finalizer" #3 daemon prio=8 os_prio=1 cpu=0.00ms elapsed=126.57s tid=0x00000176ac2de800 nid=0x3234 in Object.wait()  [0x0000004acb4ff000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(java.base@11.0.8/Native Method)
	- waiting on <0x000000070b938850> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(java.base@11.0.8/ReferenceQueue.java:155)
	- waiting to re-lock in wait() <0x000000070b938850> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(java.base@11.0.8/ReferenceQueue.java:176)
	at java.lang.ref.Finalizer$FinalizerThread.run(java.base@11.0.8/Finalizer.java:170)

   Locked ownable synchronizers:
	- None

"Signal Dispatcher" #4 daemon prio=9 os_prio=2 cpu=0.00ms elapsed=126.56s tid=0x00000176ac334000 nid=0x3230 runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"Attach Listener" #5 daemon prio=5 os_prio=2 cpu=0.00ms elapsed=126.56s tid=0x00000176ac336000 nid=0x3968 waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"C2 CompilerThread0" #6 daemon prio=9 os_prio=2 cpu=1781.25ms elapsed=126.56s tid=0x00000176ac337000 nid=0x2d4c waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
   No compile task

   Locked ownable synchronizers:
	- None

"C1 CompilerThread0" #8 daemon prio=9 os_prio=2 cpu=750.00ms elapsed=126.56s tid=0x00000176ac33b000 nid=0x944 waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
   No compile task

   Locked ownable synchronizers:
	- None

"Sweeper thread" #9 daemon prio=9 os_prio=2 cpu=31.25ms elapsed=126.56s tid=0x00000176ac33e000 nid=0x333c runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"Service Thread" #10 daemon prio=9 os_prio=0 cpu=0.00ms elapsed=126.53s tid=0x00000176ac4ac000 nid=0x2afc runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"Common-Cleaner" #11 daemon prio=8 os_prio=1 cpu=15.63ms elapsed=126.53s tid=0x00000176ac4ad800 nid=0x6bc in Object.wait()  [0x0000004acbcff000]
   java.lang.Thread.State: TIMED_WAITING (on object monitor)
	at java.lang.Object.wait(java.base@11.0.8/Native Method)
	- waiting on <0x000000070b926708> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(java.base@11.0.8/ReferenceQueue.java:155)
	- waiting to re-lock in wait() <0x000000070b926708> (a java.lang.ref.ReferenceQueue$Lock)
	at jdk.internal.ref.CleanerImpl.run(java.base@11.0.8/CleanerImpl.java:148)
	at java.lang.Thread.run(java.base@11.0.8/Thread.java:834)
	at jdk.internal.misc.InnocuousThread.run(java.base@11.0.8/InnocuousThread.java:134)

   Locked ownable synchronizers:
	- None

"pool-1-thread-1" #16 prio=5 os_prio=0 cpu=0.00ms elapsed=122.78s tid=0x00000176ac4b0800 nid=0x2eb8 waiting on condition  [0x0000004acbdfe000]
   java.lang.Thread.State: WAITING (parking)
	at jdk.internal.misc.Unsafe.park(java.base@11.0.8/Native Method)
	- parking to wait for  <0x000000070fdce168> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(java.base@11.0.8/LockSupport.java:194)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(java.base@11.0.8/AbstractQueuedSynchronizer.java:2081)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(java.base@11.0.8/ScheduledThreadPoolExecutor.java:1170)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(java.base@11.0.8/ScheduledThreadPoolExecutor.java:899)
	at java.util.concurrent.ThreadPoolExecutor.getTask(java.base@11.0.8/ThreadPoolExecutor.java:1054)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@11.0.8/ThreadPoolExecutor.java:1114)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@11.0.8/ThreadPoolExecutor.java:628)
	at java.lang.Thread.run(java.base@11.0.8/Thread.java:834)

   Locked ownable synchronizers:
	- None

"DestroyJavaVM" #17 prio=5 os_prio=0 cpu=2625.00ms elapsed=122.77s tid=0x000001768876a800 nid=0x3ad0 waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"VM Thread" os_prio=2 cpu=46.88ms elapsed=126.58s tid=0x00000176aba20800 nid=0x30a4 runnable  

"GC Thread#0" os_prio=2 cpu=0.00ms elapsed=126.60s tid=0x0000017688784800 nid=0x3158 runnable  

"GC Thread#1" os_prio=2 cpu=0.00ms elapsed=126.24s tid=0x00000176acaca000 nid=0x2dd0 runnable  

"GC Thread#2" os_prio=2 cpu=0.00ms elapsed=126.24s tid=0x00000176acaca800 nid=0x2b94 runnable  

"GC Thread#3" os_prio=2 cpu=0.00ms elapsed=126.24s tid=0x00000176ac78c800 nid=0x1d5c runnable  

"G1 Main Marker" os_prio=2 cpu=0.00ms elapsed=126.60s tid=0x00000176887e2800 nid=0xe58 runnable  

"G1 Conc#0" os_prio=2 cpu=15.63ms elapsed=126.60s tid=0x00000176887e3800 nid=0x26c0 runnable  

"G1 Refine#0" os_prio=2 cpu=0.00ms elapsed=126.59s tid=0x00000176ab90b800 nid=0x34d4 runnable  

"G1 Young RemSet Sampling" os_prio=2 cpu=0.00ms elapsed=126.59s tid=0x00000176ab90d800 nid=0x21a4 runnable  
"VM Periodic Task Thread" os_prio=2 cpu=31.25ms elapsed=126.53s tid=0x00000176ac4ac800 nid=0x2fd8 waiting on condition  

JNI global refs: 18, weak refs: 0
```  

So, what on Earth do I make of it? I see just two non-daemon threads, one of which is "DestroyJavaVM", which is, as per 
[this answer][destroy-java-vm], perfectly reasonable, which leaves us with "pool-1-thread-1" as primary suspect. It says
it's "waiting on condition 0x0000004acbdfe000", but there's no mention of this id anywhere else in the dump, so what is it?

Now I want to get back to IDE and see what we can get in the debugger. For being able to attach to the process from 
external debugger, let's execute it as follows: 

```
java -jar build/libs/demo-0.0.1-SNAPSHOT.jar -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=1044
```

Now, from the IDE I can "Attach to process" and take a "Thread dump" from within it. Hmm, ok, now I essentially see the 
exact same data as above, don't see how to get anything useful out of it. 

Let's approach it from a more old-school way: 

```
java -verbose -jar build/libs/demo-0.0.1-SNAPSHOT.jar
``` 

This gives the following output: 

```
Exiting application
[Loaded org.eclipse.rdf4j.query.BooleanQueryResultHandler from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-query-3.2.2.jar!/]
[Loaded org.eclipse.rdf4j.query.resultio.RDFStarDecodingQueryResultHandler from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-queryresultio-api-3.2.2.jar!/]
[Loaded org.eclipse.rdf4j.query.resultio.binary.BinaryQueryResultConstants from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-queryresultio-binary-3.2.2.jar!/]
[Loaded org.eclipse.rdf4j.common.io.IOUtil from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-util-3.2.2.jar!/]
[Loaded java.io.FileWriter from C:\Progra~1\Java\jdk1.8.0_144\jre\lib\rt.jar]
[Loaded java.util.Collections$CopiesList from C:\Progra~1\Java\jdk1.8.0_144\jre\lib\rt.jar]
[Loaded org.eclipse.rdf4j.model.impl.SimpleLiteral from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-model-3.2.2.jar!/]
[Loaded org.eclipse.rdf4j.model.vocabulary.RDF from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-model-3.2.2.jar!/]
[Loaded org.eclipse.rdf4j.query.AbstractBindingSet from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-query-3.2.2.jar!/]
[Loaded org.eclipse.rdf4j.query.impl.ListBindingSet from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-query-3.2.2.jar!/]
[Loaded org.eclipse.rdf4j.query.resultio.ValueMappingBindingSet from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-queryresultio-api-3.2.2.jar!/]
[Loaded org.eclipse.rdf4j.rio.helpers.RDFStarUtil from jar:file:/C:/Develop/Java/spring-rdf4j/build/libs/demo-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/rdf4j-rio-api-3.2.2.jar!/]
[Loaded org.eclipse.rdf4j.query.resultio.RDFStarDecodingQueryResultHandler$$Lambda$236/1029991670 from org.eclipse.rdf4j.query.resultio.RDFStarDecodingQueryResultHandler]
[Loaded java.util.concurrent.locks.LockSupport from C:\Progra~1\Java\jdk1.8.0_144\jre\lib\rt.jar]
```

(a Hawk Eye among my imaginary readers will notice that this is run under Java 8, not Java 11. I just thought that Java 8 
is probably a safer choice for troubleshooting purposes)

Okkkkk, so we're on to something, it seems... Let's get back to IDE and set a breakpoint on 
`RDFStarDecodingQueryResultHandler::handleSolution` method. And - it's fired.. Let's follow the executed code with 
"Step Over" function. Put's some value into queue... then get's done with the queue.. nothing to see here... 
and then we're hanging again.. 

*AAAAAAAAARGHHHHHHHHHHHHHH. Fork it, I'm leaving (for now)*

--end of episode 2 (2020-09-21 21:00 EEST)--
 
 Ok, where could we go from here? 

Obvious step would be to start actually consuming the `ResultSet`, so let's do this: 

```java
			while (result.hasNext()) {
				result.next();
			}
```

Run, and ... it did not change anything.. Ok, one thing less to try.

Ok, so, if I run the app in IDE debugger, and click "Pause execution", when the app is hanging on exit, invariably it's
"pool-1-thread-1" thread, with the same stacktrace as presented above. So, it seems like it's hanging (awaiting) on 
"available" condition in the `java.util.concurrent.ScheduledThreadPoolExecutor.DelayedWorkQueue::take` method.

Seems like some weird thread pooling issue. Maybe I could find some other executor? Hmm... Let's see. `RDF4JProtocolSession`
only has one constructor, and it expects a `ScheduledExecutorService` instance. And it seems like there're not too many 
ways to instantiate one... I've tried with `Executors.newSingleThreadScheduledExecutor()` as well, but it did not change 
anything, obviously. 

What if we set field watchpoint on `java.util.concurrent.ScheduledThreadPoolExecutor.DelayedWorkQueue.available` field? 
Let's try it out. I've checked "Log Breakpoint hit message", "Log Stacktrace" and set "Evaluate and log" field to "leader" 

Here's the output: 

```
Connected to the target VM, address: '127.0.0.1:54378', transport: 'socket'

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.4.RELEASE)

2020-09-23 20:57:44.706  INFO 16064 --- [           main] com.example.demo.DemoApplication         : Starting DemoApplication on DESKTOP-TSSQSTI with PID 16064 (C:\Develop\Java\spring-rdf4j\out\production\classes started by kiril in C:\Develop\Java\spring-rdf4j)
2020-09-23 20:57:44.711  INFO 16064 --- [           main] com.example.demo.DemoApplication         : No active profile set, falling back to default profiles: default
2020-09-23 20:57:46.986  INFO 16064 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 2.813 seconds (JVM running for 3.533)
Starting application
{java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue@3,921}.available will be accessed at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.offer(ScheduledThreadPoolExecutor.java:1024)
Breakpoint reached
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.offer(ScheduledThreadPoolExecutor.java:1024)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.add(ScheduledThreadPoolExecutor.java:1037)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.add(ScheduledThreadPoolExecutor.java:809)
	at java.util.concurrent.ScheduledThreadPoolExecutor.delayedExecute(ScheduledThreadPoolExecutor.java:328)
	at java.util.concurrent.ScheduledThreadPoolExecutor.schedule(ScheduledThreadPoolExecutor.java:533)
	at java.util.concurrent.ScheduledThreadPoolExecutor.execute(ScheduledThreadPoolExecutor.java:622)
	at java.util.concurrent.Executors$DelegatedExecutorService.execute(Executors.java:668)
	at org.eclipse.rdf4j.http.client.BackgroundResultExecutor.autoCloseRunnable(BackgroundResultExecutor.java:63)
	at org.eclipse.rdf4j.http.client.BackgroundResultExecutor.parse(BackgroundResultExecutor.java:33)
	at org.eclipse.rdf4j.http.client.SPARQLProtocolSession.getBackgroundTupleQueryResult(SPARQLProtocolSession.java:669)
	at org.eclipse.rdf4j.http.client.SPARQLProtocolSession.sendTupleQuery(SPARQLProtocolSession.java:373)
	at com.example.demo.wikidata.QueryExecutor.executeQuery(QueryExecutor.java:40)
	at com.example.demo.DemoApplication.run(DemoApplication.java:23)
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:795)
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:779)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:322)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1237)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226)
	at com.example.demo.DemoApplication.main(DemoApplication.java:17)
null
Exiting application
{java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue@3,921}.available will be accessed at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1081)
Breakpoint reached
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1081)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:809)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
null
``` 

So, sounds pretty legit? the task is offered once and taken once? but, which threads were _executing_ 
those operations? Checking "Suspend" box on the watchpoint and running again shows, that the first (`offer`) operation 
is run in the `main` thread, while `take` operation is run in the `pool-1-thread-1` thread, i.e. the one which is 
(supposedly) created by our single-threaded thread pool.     

At this point I decided to try and run this code in the "Spring-less" application, i.e. the one which only uses RDF4j 
without Spring at all. And you know what? It behaves exactly like this one!! Except, with that application, I managed to 
put `System.exit(0)` at the end of the `main` method. And then forgot about it completely.. FFS...

So, now, if I add that same call into the end of my 'run' method, it also finishes alright. 

So, the mystery is not resolved, but at least now it seems I can put the finger squarely on RDF4J and call it a day..

[thread-dump]: https://dzone.com/articles/how-analyze-java-thread-dumps
[destroy-java-vm]: https://stackoverflow.com/a/34433567/2583044