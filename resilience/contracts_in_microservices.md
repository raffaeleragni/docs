# Conventions for resilience in microservices

The aim of these conventions is to raise resilience in the microservice ecosystem.

## JSON serialization

<sub><sup>Note: here Jackson will be used as a reference for feature configuration.</sup></sub>

These convention applies to JSON entities, whether they are in a request/response or in a message/entity, or stored somewhere.

- Avoid failing on unknown properties.
  This prevents downstream services to break when a new property is being introduced. Additive changes must not break downstream services. 
  
  `.configure(FAIL_ON_UNKNOWN_PROPERTIES, false)`

- Omit properties with null values: treat non present properties as null or vice versa without errors.
  This behaviour makes the entity more data oriented than schema oriented and avoid some changes that are not meant to be breaking, to break downstream services.
  
  `.serializationInclusion(NON_NULL)`

- Decide for a standard along the microservices that is consistent. This is just an example, chose the variants you need:

  `.propertyNamingStrategy(SNAKE_CASE)`

  `.enable(INDENT_OUTPUT)`

- Be careful with enumerators. Enums are the most breaking aspect of contracts. Some programming languages use different cases, and usually allow for future values to be processed, either by adding a `@JsonEnumDefaultValue` and/or by considering unknown as null.
  
  `.configure(ACCEPT_CASE_INSENSITIVE_ENUMS, true)`

  `.configure(READ_UNKNOWN_ENUM_VALUES_AS_NULL, true)`

- A nice to have is to support Optionals for nullable types.

  `mapper.registerModule(new Jdk8Module());`

- And records support.

  ```
  mapper.setVisibility(
    mapper.getSerializationConfig().getDefaultVisibilityChecker()
      .withFieldVisibility(JsonAutoDetect.Visibility.ANY)
      .withGetterVisibility(JsonAutoDetect.Visibility.ANY)
      .withSetterVisibility(JsonAutoDetect.Visibility.ANY)
      .withCreatorVisibility(JsonAutoDetect.Visibility.ANY)
  );
  ```

- Timestamps require some attention, first disabling the default serialization into integers, and then converting them to a preferred format, such as RFC3339.

  `.disable(WRITE_DATES_AS_TIMESTAMPS)`

  ```
  var module = new SimpleModule();
  DateTimeFormatter formatterWrite = ISO_OFFSET_DATE_TIME;
  DateTimeFormatter fromatterRead = ISO_OFFSET_DATE_TIME;
  module.addSerializer(ZonedDateTime.class, new JsonSerializer<ZonedDateTime>() {
      @Override
      public void serialize(ZonedDateTime zonedDateTime, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
      jsonGenerator.writeString(formatterWrite.format(zonedDateTime));
      }
  });
  module.addDeserializer(ZonedDateTime.class, new JsonDeserializer<ZonedDateTime>() {
      @Override
      public ZonedDateTime deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
      return ZonedDateTime.from(fromatterRead.parse(p.getText()));
      }
  });
  mapper.registerModule(module);
  ```

## HTTP service-to-service calls

These are a set of headers and convention that will make API calls (service to service) more traceable. Request headers are responsibility of the client calling, while response headers are responsibility of the server responding.

- request header (non-standard) `Request-Time` UNIX millisecs of the time of requesting, calculated from client side.
- request header `User-Agent` containing an identifier of the service/module calling, ex `mobile-app-1.0.2`
- request header 
- response header (non-standard) `Response-Time` UNIX millisecs of the time of responding, calculated from server side. (`Date` header also exists but it is not a timestamp field)
- response header `Server` containing an identifier of the service/module responding, ex `orders-tracking-1.3.2`
- response header (non-standard) `Request-Id` 

Other behaviors include how the client and servers will behave when certain headers are used, and what is expected of them.

- respect the `Cache-Control` for GET requests at client side.
- respect `429` response code and `Retry-After` header.

## Throttling and Retries

When server returns status code `429` will also add a header `Retry-After`
Response status 429 and `Retry-After` header containing the seconds to wait for the client before attempting to hit the server again.

In case the serer is busy and cannot respond at all to any of the sent requests (they keep queuing in memory), the client should also circuit break with some default wait time and make those request fail or retry in a longer timespan. Preferrably these retries will be kept in their own memory queue by the client and not occupy the socket I/O.

Retrying should happen with exponential backoff. For example if the base retry time is `100ms`, the first retry will happen after 100, the 2nd after 200 from the 1st, the 3rd after 400 from the 2nd, the 4th 800 from the 3rd and so on. The base retry time can be chosen depending of your needs.

### Proper return codes

Server should return 4xx or at least 400 for requests that have issues in deserialization to distinguish between what can be retried and what not. Ideally 4xx errors should not be alerting as critical, but only as warnings, or ar the very least one category lower than the 5xx errors.

Client should not retry on common 4xx validation errors (excluding 429).

### Messaging/Events

Messages also should be distinguished between retriable and not. Non retrieable messages (message unparseable, message won't change next time) should be either discarded or dead letter queued. This means that the application needs to clearly separate its exception flow between the two cases and be able to recognize the difference.
