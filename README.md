#![shaved wookiee](http://www.gravatar.com/avatar/62cf8eb12029b66dfa837efa365f12b4) swookiee* JVM REST Service Runtime 

[![Build Status](https://travis-ci.org/swookiee/com.swookiee.runtime.png?branch=develop)](https://travis-ci.org/swookiee/com.swookiee.runtime)

## What is swookiee*
swookiee is a shaved wookiee and looks like this:

![shaved wookiee](http://www.gravatar.com/avatar/62cf8eb12029b66dfa837efa365f12b4?s=80)

swookiee basically is a JVM Runtime for REST Services. It is lightweight (~13 MB) and includes libraries like guava, joda-time, metrics, etc. It also supports service implementation written in Groovy and Scala (+7MB each). It uses a Jersey 2.5, Jetty, Jackson stack on top of the Equinox OSGi runtime to serve REST Services. swookiee also provides multiple REST APIs to deploy and control services and components via REST.

Our main goals are:
* simplify exposing REST Services in the JVM
* JVM-Polyglot: Solve problems in best fit language
* reduce mandatory infrastructure and architectural knowledge from developers
* define boundaries and APIs
* enable deployment on artifact level (very simple OSGi [RFC-182 implementation](https://github.com/osgi/design/tree/master/rfcs/rfc0182))
* increase transparency through direct metrics, graphite and JSON logging support

## Build \& start

Build & Start
```shell
cd com.swookiee.runtime
mvn clean install

cd com.swookiee.runtime
mvn exec:exec
```

## Web Console
When swookiee is up and running the Web Console can be accessed via [`http://localhost:8080/system/console`](http://localhost:8080/system/console). The default credentials are `admin:admin123`.

## REST Services

To expose a REST service in swookiee you can describe the REST API with a simple interface:
```java
@Path("/hello")
@Produces(APPLICATION_JSON)
public interface HelloWorld {
    @GET
    String hello();
}
```

Through using the given [`@Component`](http://www.osgi.org/javadoc/r5/cmpn/org/osgi/service/component/annotations/Component.html) annotation the implementation of the interface will be exposed.
```java
@Component
public class HelloWorldService implements HelloWorld {
    @Override
    public String hello() {
        return "Hola!";
    }
}
```

A REST service will be exposed using the [OSGi - JAX-RS Connector](https://github.com/hstaudacher/osgi-jax-rs-connector). Thus, every JAX-RS component which is an available OSGi service will also be published via Jersey. You will find a deteiles Jersey user guide [here](https://jersey.java.net/documentation/latest).

## Filters, Exception Mapping, ...

In addition to provide services you might need to register Filter (e.g. for CORS). This can be done via `ContainerResponseFilter` and `ContainerRequestFilter` implementations:

```java
@Component
@Provider
public class MyFilter implements ContainerResponseFilter {

    @Override
    public void filter(final ContainerRequestContext requestContext, final ContainerResponseContext responseContext) {
    }
}
```

```java
@Component
@Provider
public class MyFilter implements ContainerRequestFilter {

    @Override
    public void filter(final ContainerRequestContext requestContext) {
    }
}
```

In order to register an exception mapper you have to implement the `ExceptionMapper` interface:
```java
@Component
@Provider
public class MyExceptionMapper implements ExceptionMapper<MyException> {

    @Override
    public Response toResponse(final MyException exception) {
        return Response.status(Status.INTERNAL_SERVER_ERROR).build();
    }
}

```

## JSON logging output
In case you want to use logstash and kibana to analyze your logs you might want to use JSON as an output format.
This can be achieved through setting the property `productionLogging` to `true`.

## Deployment and runtime management via REST
[RFC-182](https://github.com/osgi/design/tree/master/rfcs/rfc0182)

## Configuration
The Swookiee Runtime can be configured programmatically and via the UI of the Web Console. 

### Programmatic Configuration
The programmatic configuration can be done via an API on top of the OSGi Configuration Admin. The full sample can be found [here](https://github.com/swookiee/com.swookiee.sample/tree/develop/com.swookiee.sample.configuration)

Define a POJO with the components you want to configure. In this case we want to change the admin user credentials and the graphite reporter configuration:
```java
public class Configuration {
    public AdminUserConfiguration adminUserConfiguration;
    public GraphiteReporterConfiguration graphiteReporterConfiguration;
}
```

Define a YAML file which matches the attributes of your POJO. Additionally set the desired parameter values:
```yml
adminUserConfiguration:
  username: "admin"
  password: "admin42"
    
graphiteReporterConfiguration:
  graphiteHost: localhost
  graphitePort: 2003
  reportingIntervalInSeconds : 42
  reportingEnabled: true
  reportingPrefix: "configDemo"
```

In order to set your configuration values to the addressed components you can create a Declarative Service Component which is then using `ConfigurationUtils` to set your configuration parameters
```java
@Component
public class ConfigurationProviderSample {

    private ConfigurationAdmin configAdmin;

    @Activate
    public void activate(final BundleContext bundleContext) {

        final URL configurationFile = bundleContext.getBundle().getResource("Configuration.yaml");
        ConfigurationUtils.applyConfiguration(Configuration.class, configurationFile, configAdmin);

    }

    @Reference
    public void setConfigurationAdmin(final ConfigurationAdmin configurationAdmin) {
        this.configAdmin = configurationAdmin;
    }

    public void unsetConfigurationAdmin(final ConfigurationAdmin configurationAdmin) {
        this.configAdmin = null;
    }
}
```

### UI
Most configuration parameters can be controlled via the Web Console: [http://localhost:8080/system/console/configMgr](http://localhost:8080/system/console/configMgr)

## Pushing metrics to graphite

Every published REST interface will be monitored via metrics. In addition basic JVM statistics will be collected. In order to configure the graphite reporter you can configure various settings:

```yml
graphiteReporterConfiguration:
  graphiteHost: localhost
  graphitePort: 2003
  reportingIntervalInSeconds : 42
  reportingEnabled: true
  reportingPrefix: "configDemo"
```

Example Screenshot of some made up traffic:

![graphite](https://raw.github.com/swookiee/swookiee.github.io/master/screenshots/graphite.png)

## Swagger
The above described REST Resource can be enriched using `swagger-annotations`. The full sample can be found [here](https://github.com/swookiee/com.swookiee.sample/tree/develop/com.swookiee.sample.swagger).

We could add the `@Api` and `@ApiOperation` annotations to our sample service:
```java
@Path(Status.PATH)
@Produces(APPLICATION_JSON)
@Api(value = Status.PATH, description = "Returns a status")
public interface Status {

    final static String PATH = "/status";

    @GET
    @ApiOperation(value = "Ping the Status API to receive a StatusObject", response = StatusObject.class)
    public StatusObject ping();
}
```

The following steps are important to let swookiee deliver you swagger api documentation:

1. Make sure you have the `maven-swagger-plugin` configured properly and you are able to see generated swagger json files? Also make sure that the `<basePath>/services</basePath>` is set according to your configuration. Otherwise you won’t be able to test you API.
2. Is the generated json part of your bundle/artefact/`.jar`?
3. Add the `X-Swagger-Documentation` header in you `MANIFEST.MF` or your `maven-bundle-plugin` configuration to point to your documentation folder inside your `jar`, such as: `X-Swagger-Documentation: /swagger`.

After installing the bundle you will find the `swagger-ui` under `http://localhost:8080/apidocs/api`. Click on the small swookiee icon to see your documentation and test your API.

Screenshot of the sample:
![swagger](https://raw.github.com/swookiee/swookiee.github.io/master/screenshots/swagger.png)

## Archetype(s) & Tooling & Samples
TODO

### Eclipse

### Netbeans

## Todos
 * provide rpm, deb and other packages
 * support more JVM languages
 * documentation and samples

## Thanks
Many thanks to Intuit Data Services for the support!

[![intuit](http://about.intuit.com/about_intuit/press_room/intuit_logos/intuit_blue.gif)](http://intuit.com)

## License
The code is published under the terms of the [Eclipse Public License, version 1.0](http://www.eclipse.org/legal/epl-v10.html).

\* derived from "shaved wookiee"
