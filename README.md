# WireMock PACT

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/se.bjurr.wiremockpact/wiremock-pact/badge.svg)](https://repo1.maven.org/maven2/se/bjurr/wiremockpact)

Get requests from [WireMock](https://github.com/wiremock/wiremock/) and create [PACT JSON](https://docs.pact.io/). You may publish the JSON to a broker using [pact-jvm](https://github.com/pact-foundation/pact-jvm/tree/master/provider).

This repostory contains:

- `wiremock-pact-lib` - *A library that can transform WireMock [ServeEvent](https://github.com/wiremock/wiremock/blob/master/src/main/java/com/github/tomakehurst/wiremock/stubbing/ServeEvent.java):s to PACT JSON.*
- `wiremock-pact-extension` - *A WireMock extension that is intended to ease usage of the library.*
- `wiremock-pact-example-springboot-app` - *A SpringBoot application that shows how it can be used.*

## Usage - Junit 5

The extension is both a WireMock extension and a JUnit 5 extension. When using [`wiremock-spring-boot`](https://wiremock.org/docs/solutions/spring-boot/) it can be configured like this in a base class of your tests:

```java
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import com.maciejwalkowiak.wiremock.spring.ConfigureWireMock;
import com.maciejwalkowiak.wiremock.spring.EnableWireMock;
import com.maciejwalkowiak.wiremock.spring.WireMockConfigurationCustomizer;
import org.junit.jupiter.api.extension.RegisterExtension;
import se.bjurr.wiremockpact.wiremockpactextension.WireMockPactExtension;
import se.bjurr.wiremockpact.wiremockpactlib.api.WireMockPactConfig;

@EnableWireMock({
  @ConfigureWireMock(
      name = "wiremock-service-name",
      property = "wiremock.server.url",
      stubLocation = "wiremock",
      configurationCustomizers = {WireMockPactBaseTest.class})
})
public class WireMockPactBaseTest implements WireMockConfigurationCustomizer {
  @RegisterExtension
  static WireMockPactExtension WIREMOCK_PACT_EXTENSION =
      new WireMockPactExtension(
          WireMockPactConfig.builder() //
              .setConsumerDefaultValue("WireMockPactExample") //
              .setProviderDefaultValue("UnknownProvider") //
              .setPactJsonFolder("src/test/resources/pact-json"));

  @Override
  public void customize(
      final WireMockConfiguration configuration, final ConfigureWireMock options) {
    configuration.extensions(WIREMOCK_PACT_EXTENSION);
  }
}
```

## Usage - Library

It can be used as a library.

```java
public class ExampleTest {
  private static WireMockServer server;
  private static WireMockPactApi wireMockPactApi;

  @BeforeAll
  public static void beforeEach() throws IOException {
    server = new WireMockServer();
    server.start();

    stubFor(
        post(anyUrl())
            .willReturn(
                ok()
                .withHeader("content-type", "application/json")
                .withBody("""
                {"a":"b"}
                """))
            .withMetadata(
                new Metadata(
                    Map.of(
                        WIRE_MOCK_METADATA_PACT_SETTINGS,
                        new MetadataModelWireMockPactSettings("some-specific-provider")))));

    wireMockPactApi =
        WireMockPactApi.create(
            new WireMockPactConfig()
                .setConsumerDefaultValue("my-service")
                .setProviderDefaultValue("unknown-service")
                .setPactJsonFolder("the/pact-json/folder");
    wireMockPactApi.clearAllSaved();
  }

  @Test
  public void testInvoke() {
    // Do stuff that invokes WireMock...
  }

  @AfterAll
  public static void after() {
    for (final ServeEvent serveEvent : server.getAllServeEvents()) {
      wireMockPactApi.addServeEvent(serveEvent);
    }
    // Save pact-json to folder given in WireMockPactApi
    wireMockPactApi.saveAll();
    server.stop();
  }
}
```

## Mappings metadata - Set provider in mapping

You can adjust any mappings file like this to specify the provider of a mapping in its [metadata](https://github.com/wiremock/spec/blob/main/wiremock/wiremock-admin-api/schemas/stub-mapping.yaml) field:

```diff
{
  "id" : "d68fb4e2-48ed-40d2-bc73-0a18f54f3ece",
  "request" : {
    "urlPattern" : "/animals/1",
    "method" : "GET"
  },
  "response" : {
    "status" : 202
  },
  "uuid" : "d68fb4e2-48ed-40d2-bc73-0a18f54f3ece",
+  "metadata": {
+   "wireMockPactSettings": {
+     "provider":"some-other-system"
+   }
+  }
}
```

Or programmatically:

```java
    stubFor(
        post(anyUrl())
            .withMetadata(
                new Metadata(
                    Map.of(
                        WIRE_MOCK_METADATA_PACT_SETTINGS,
                        new MetadataModelWireMockPactSettings("some-specific-provider")))));
```
