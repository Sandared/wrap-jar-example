# wrap-jar-example
A small project to show how to wrap a Jar into a bundle that works well in an OSGi runtime

How to wrap a Jar:
* We will do this for the gremlin-driver Jar
* Setup an empty Maven project, i.e., no src folder, just a pom.xml wtih the following content:
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>io.qbilon</groupId>
    <artifactId>osgi-gremlin-driver</artifactId>
	<version>1.0.0-SNAPSHOT</version>
    <packaging>bundle</packaging>
    <name>My Wrapping Bundle</name>

    <description>This OSGi bundle simply wraps the gremlin-driver artifact.</description>

	<properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.tinkerpop</groupId>
            <artifactId>gremlin-driver</artifactId>
            <version>3.4.4</version>
            <optional>false</optional>
        </dependency>
        <dependency>
            <groupId>org.apache.tinkerpop</groupId>
            <artifactId>gremlin-driver</artifactId>
            <version>3.4.4</version>
            <optional>false</optional>
            <classifier>sources</classifier>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
				<groupId>org.apache.felix</groupId>
				<artifactId>maven-bundle-plugin</artifactId>
				<version>4.2.1</version>
				<extensions>true</extensions>
				<dependencies>
					<dependency>
						<groupId>biz.aQute.bnd</groupId>
						<artifactId>biz.aQute.bndlib</artifactId>
						<version>5.0.0</version>
					</dependency>
				</dependencies>
				<configuration>
					<instructions>
						<Bundle-SymbolicName>${project.artifactId}</Bundle-SymbolicName>
						<Bundle-Version>${project.version}</Bundle-Version>
					</instructions>
				</configuration>
			</plugin>
        </plugins>
    </build>
</project>
```
In this state when `mvn clean install` is called it does nothing but produce an empty jar with only the necessary manifest headers.

Let's add some packages that we actually want to export from the dependency. Alter the instructions of the maven-bundle-plugin (mbp) like this:
```xml
<configuration>
	<instructions>
		<Bundle-SymbolicName>${project.artifactId}</Bundle-SymbolicName>
		<Bundle-Version>${project.version}</Bundle-Version>
		<Export-Package>org.apache.tinkerpop.gremlin.*</Export-Package>
		<!-- Don't know why, but without this Private-Package section the maven-bundle-plugin doesn't do anything -->
		<Private-Package></Private-Package>
	</instructions>
</configuration>
```

You will now get a warning like this:
```
[WARNING] Bundle io.qbilon:osgi-gremlin-driver:bundle:1.0.0-SNAPSHOT : Split package, multiple jars provide the same package:org/apache/tinkerpop/gremlin/driver
Use Import/Export Package directive -split-package:=(merge-first|merge-last|error|first) to get rid of this warning
Package found in   [Jar:gremlin-driver, Jar:gremlin-driver]
Class path         [Jar:gremlin-driver, Jar:gremlin-core, Jar:gremlin-shaded, Jar:commons-configuration, Jar:commons-lang, Jar:commons-collections, Jar:snakeyaml, Jar:javatuples, Jar:hppc, Jar:jcabi-manifests, Jar:jcabi-log, Jar:javapoet, Jar:exp4j, Jar:slf4j-api, Jar:jcl-over-slf4j, Jar:netty-all, Jar:groovy, Jar:groovy-json, Jar:commons-lang3, Jar:gremlin-driver]
```

This means the mbp tried to include the packages mentioned in the Export-Package section, but found packages matching this pattern in different jars. (Well in this case it seems it is also warning about finding multiple packages matching this expression in one jar, i.e., the gremlin-driver jar.)
This probably is because we only gave it `org.apache.tinkerpop.gremlin.*` and in the gremlin-driver jar there are multiple packages that fit this expression, e.g.: `org.apache.tinkerpop.gremlin.driver` / `org.apache.tinkerpop.gremlin.jsr223` / etc.

In order to get rid of this warning just write the Export-Package statement like this:
```xml
<Export-Package>org.apache.tinkerpop.gremlin.*;-split-package:=merge-first</Export-Package>
```

-split-package:=merge-first advises the mbp to 	"Merge split packages but do not add resources that come later in the classpath. That is, the first resource wins" (https://bnd.bndtools.org/heads/private_package.html) 

Now we have a jar that exports all the stuff we want it to export.
Problem is, there are some dependencies that this bundle needs, as we can see in its MANIFEST.MF

For those dependencies there are two options:
1) The dependency itself is an OSGi bundle with a proper Manifest -> no problem. Load this bundle too in your runtime and you should be fine
2) The dependency is not an OSGi bundle (i.e. it has no Manifest with proper OSGi entries). For those we have three options:
	1) We include the packakes of these dependencies that are needed by our bundle in our bundle. This is done by adding those packages step by step to either Export-Package or Private-Package
	2) Include the whole jar in our bundle by Embed-Dependency/_exportcontents (not as fine grained as packages, but does the job)
	3) Turn the respective jar into a bundle like we do with the gremlin-driver (probably means more work, but also the cleanest solution)

I personally usually go through all dependency jars and if they don't have a proper manifest, include them by Import-Package/Private-Package.

So first we open the generated MANIFEST.MF and see that in the Import-Package section it states that it needs `com.jcabi.manifests` which probably comes from the transitive dependency `com.jcabi:jcabi-manifests:jar:1.1` (You can see those dependencies by typing `mvn dependency:tree` or using the Java Dependencies View in Theia/VSCode)
This jar has no proper Manifest (using the Java Dependencies View in Theia/VSCode you can navigate to the Manifest of this jar and will see it has no OSGi Header entries), so I put the package into the Private-Package section to advise mbp to include these packages into my bundle like this:

```xml
<Private-Package>
	com.jcabi.manifests*
</Private-Package>
```

In order to advise the mbp also to not further include this package in the Import-Package section of my manifest I also add it to the Import-Package section of the mbp configuration, like this:

```xml 
<Import-Package>
	!com.jcabi.manifests*,
	*
</Import-Package>
```

The `!` advises the mbp NOT to include this package or any subpackage. The additional `*` advises it to still import the rest.
Now when we call `mvn clean install` the resulting jar also contains the package `com.jcabi.manifests`

We procede like this for the next few packages, e.g., com.jcabi.log*, com.squareup.javapoet*

groovy.json is the first real OSGi bundle. As it already has a proper manifest I do not put the packages anywhere, but instead load the bundle into my OSGi runtime, so that the dependencies are resolved at runtime.

`io.netty.*`, which is the next Import-Package entry on our list is also no OSGi bundle, but here it becomes a little bit tricky. Netty usually is provided by containers like Apache Karaf (through the http feature) so this one dependency we just can leave alone.

`javax.lang.model.util` is a typical `javax.*` dependency also usually provided by the container. There are several `javax.*` following, that we will all ignore.

Next one is `net.objecthunter.exp4j` which probably comes from the `exp4j-0.4.8.jar` (A look in the Java Dependencies view confirms this, as we can see the packages here too). No OSGi Manifest -> include it in the Private-Package section.

ATTENTION: when you executed `mvn clean install` afterwards, then there is suddenly a warning like this:
```
[WARNING] Bundle io.qbilon:osgi-gremlin-driver:bundle:1.0.0-SNAPSHOT : Export org.apache.tinkerpop.gremlin.process.traversal.step.map,  has 1,  private references [net.objecthunter.exp4j]
```

This means that some code we export (Through Export-Package) references some code that we just declared as Private-Package. This should not be, as this mixes API (exported) with Implementation (private). So we have to export this one too, as otherwise there could be the case, that for example a type used as parameter in our API is not accessible during runtime. Therefore our Export/Import/Private-Package sections look like this now:

```xml
<Export-Package>
	org.apache.tinkerpop.gremlin.*;-split-package:=merge-first,
	net.objecthunter.exp4j*
</Export-Package>
<Import-Package>
	!com.jcabi.manifests*,
	!com.jcabi.log*,
	!com.squareup.javapoet*,
	!net.objecthunter.exp4j*, // Not sure if this should not be negated -> Imports and Exports the package
	*
</Import-Package>
<Private-Package>
	com.jcabi.manifests*,
	com.jcabi.log*,
	com.squareup.javapoet*
</Private-Package>
```

Next one is `org.apache.commons.collections`, but this is an OSGi bundle. 
Same goes for:
* `org.apache.commons.configuration`
* `org.apache.commons.lang3`
* `org.yaml.snakeyaml`

org.apache.log4j.*?
