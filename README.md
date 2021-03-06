# Agoda Rate-limited API home assignment

- Agoda Rate-limited API
- Requirement [Rate_API_2020.pdf](./Rate_API_2020.pdf)
- Built with JDK, Spring Boot, Postgres 


<!-- TABLE OF CONTENTS -->
<details open="open">
  <summary><b>Table of Contents</b></summary>
  <ol>
    <li><a href="#setup">Setup</a></li>
    <li><a href="#endpoints">Endpoints</a></li>
    <li><a href="#rate-limited-api-solution">Rate-limited API Solution</a></li>
    <li><a href="#integration-tests">Integration Tests</a></li>
  </ol>
</details>

<!-- Setup -->
## Setup

#### Installation & Setup

1. Download [JDK 11](http://jdk.java.net/archive/) 
2. Download [Postgres 12](https://www.postgresql.org/download/) 
3. Open IDE (e.g. IntelliJ)
4. Setting JDK 11: **File** > **Project Structures..** > on **Project SDK** section, click **Edit**, and import the downloaded JDK.
5. Setting up Database and Schema on Postgres.

    5.1 Database: Open the installed **pgAdmin 4** which comes with Postgres installation > Right click on left panel in **Browser** section > **Create** > **Database** > fill `hotelBooking` into **Database** input box.
    
    5.2 Schema: Expands your created database > Right click **Schemas** > **Create** > **Schema** > fill `svc_hotel` in **Name** input box.

    Note: You can change the database and schema name in `src/main/resources/application.properties`.
        
    ```
    schema.name=svc_hotel
    database.name=hotelBooking
    ```
        
6. Import dependencies in pom.xml by right click the `pom.xml` > **Maven** > **Reload**.
7. Run `src/main/java/com/hotelbooking/SvcHotelApplication.java`.
    - The server port is `8080`.
8. Curl to init data in database: `curl --location --request POST 'localhost:8080/init'`.
    - It'll read data from `src/main/resources/hoteldb.csv` and INSERT into the database.
    - [Database design](https://drive.google.com/file/d/1z6HBKNuGCMV8mIlXoo6vIrmHkaKpKQQZ/view) 
    
<!-- Endpoints -->
## Endpoints
1. `GET localhost:8080/city`
    - Request Mapping:
    
    |Field name| Required|Location|List of Value|
    |----------|---------|--------|-------------|
    |city|Mandatory|Params|-|
    |sort|Optional|Params|ASC, DESC|
    
    - Example: `GET localhost:8080/city?city=Bangkok&sort=ASC`.

2. `GET localhost:8080/room`
    - Request Mapping:
    
    |Field name| Required|Location|List of Value|
    |----------|---------|--------|-------------|
    |sort|Optional|Params|ASC,DESC|
    
    - Example: `GET localhost:8080/room?sort=ASC`.
    
<!-- Rate-limited API Solution -->
## Rate-limited API Solution

#### Configuration files

- `application.properties`
    - Configure rate limit max requests and period of the max requests.
    
    ```rate-limitting.paths=/city,/room
       
       rate-limitting.max.requests./city=10
       rate-limitting.specific-period./city=5
       
       rate-limitting.max.requests./room=100
       rate-limitting.specific-period./room=10
  
       rate-limitting.max.requests.default=50
       rate-limitting.specific-period.default=10
    ```
  
    - `rate-limitting.paths` configures a list of the paths (of APIs) that need to do rate limited.
    - `rate-limitting.max.requests./city` configures max requests of path `/city`.
       
      `rate-limitting.max.requests./room` configures max requests of path `/room`.
    
    - `rate-limitting.specific-period./city` configures period of the max requests of path `/city`.
    
      `rate-limitting.specific-period./room` configures period of the max requests of path `/room`.
    - `rate-limitting.max.requests.default` configures default max requests. If the max requests of specific paths not defined, this default value will be applied.
    - `rate-limitting.specific-period.default` configures default period of the max requests. If the period of specific paths not defined, this default value will be applied.
    - For example,
        - With `rate-limitting.max.requests./city=10` and `rate-limitting.specific-period./city=5`, the `/city` endpoint can receive a maximum of 10 requests every 5 second.
        - With `rate-limitting.max.requests./room=100` and `rate-limitting.specific-period./room=10`, the `/room` endpoint can receive maximum 100 requests every 10 seconds.
- `ThrottleRateLimitConfig.java`
    - Read configurations into Java Bean.
        - `rateLimittingMaxRequest` keep max requests of each endpoint which read from `application.properties`.
            - Data type: `HashMap<String, Integer>` - keep **path** of the endpoint as key, and max requests of the endpoint calls on a specific period.
        - `RateLimit.java` is data model of the endpoints and related fields including:
            - `key` represents **path** of the endpoint.
            - `count` represents the number of the endpoint calls on a specific period.
            - `refreshDatetime` represents the latest time the **path** is refresh from counting max requests (when reset `count = 0`).
            - `lastAccessDatetime` represents the latest time the `key` is accessed from read or write.
            - `expiredAfterWrite` represents the expired time for each period of the key.
            - `blockRequestUntilDatetime` 
                - represents the end of blocked request access time (default is 5 seconds after the endpoint accessed at over the defined max requests limit).
                - As the requirement aren't specified that after refresh cache should refresh the rate limiting count on the period or not, 
                this project refresh counting the rate limiting after block request for 5 seconds.
        - `requestCountsPerPath` 
            - Data type: `ConcurrentHashMap<String, RateLimit>` 
            - keep **path** of the endpoint as key, and the `RateLimit` data model.

- `RequestThrottlePerRequestFilter.java`
    - Using [OncePerRequestFilter](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/OncePerRequestFilter.html)
        - Filter base class that aims to guarantee a single execution per request dispatch, on any servlet container.
    - `shouldNotFilter(HttpServletRequest request)` function - won't limit rate on not-defined endpoints.
    - `isMaximumRequestsExceeded(path)` function:
        - returns whether the **path** request calls exceeded the max within the defined period.
     
        ```
        private boolean isMaximumRequestsExceeded(String key) {
          RateLimit rateLimit = requestCountsPerPath.get(key);
          synchronized (rateLimit) {
  
              int requests = rateLimit.getCount();
  
              if (requests > rateLimittingMaxRequest.get(key)) {
                  rateLimit.blockIncomingRequest();
                  return true;
              }
  
              requests++;
              rateLimit.write(requests);
              return false;
          }
      }
        ```

      - If it exceeded the max limit, it'll return `HttpStatus.TOO_MANY_REQUESTS`, Http status 429.

## Integration Tests
- `HotelRateLimitIntegrationTest.java` contains 6 cases:
    1. Feature: `/city` returns collect result
         `/city` return hotels of the input city and I might want to sorted price.
      
       Given I retrieve hotels from `/city`
           When I call `/city`
           And I specify city name with a specific city name e.g. Bangkok
           And optionally, I can sort result by price in descending or ascending order
           Then I should get correctly hotels of the city and price in the correct order
    2. Feature: `/room` returns collect result
         `/room` returns all hotels, and I might want to sorted price.
      
       Given I retrieve hotels from `/room`
           When I call `/room`
           And optionally, I can sort result by price in descending or ascending order
           Then I should get correctly hotels data with price in the correct order
    3. Feature: `/city` can stop responding after the limited-rate gets higher than the threshold
         If the rate gets higher than the threshold on an endpoint, the API should stop responding.
      
       Given The `/city` endpoint can receive maximum 10 requests every 5 seconds
           When I call `/city` at 11st time with 5 seconds
           Then I should get 429 http status.
    4. Feature: `/city` can stop responding after the limited-rate gets higher than the threshold
         If the rate gets higher than the threshold on an endpoint, the API should stop responding for 5 seconds on that endpoint ONLY, before allowing other requests
      
       Given The `/city` endpoint can receive maximum 10 requests every 5 seconds.
           When I call `/city` at 11st time with 5 seconds.
           Then I should get 429 http status instead of hotel result for 5 seconds.
           Then I still can successfully call to `/room`.
    5. Feature: `/city` can unblock stopping responding after 5 seconds
        If the rate gets higher than the threshold on an endpoint, the API should stop responding for 5 seconds on that endpoint, before allowing other requests.
        After that, it can receive new requests.
     
       Given The `/city` endpoint can receive maximum 10 requests every 5 seconds.
          When I call `/city` wait after blocking for 5 seconds.
          Then I successfully send request to `/city`.
    6. Feature: `/room` can stop responding after the limited-rate gets higher than the threshold.
        If the rate gets higher than the threshold on an endpoint, the API should stop responding for 5 seconds on that endpoint, before allowing other requests.
     
       Given The `/room` endpoint can receive maximum 100 requests every 10 seconds.
          When I call `/room` at 101st time with 10 seconds.
          Then I should get 429 http status instead of hotel result ONLY for 5 seconds.