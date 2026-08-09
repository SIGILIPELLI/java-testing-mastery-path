# 04 · API Testing with RestAssured

Every screen you tested in Level 1 is a thin layer over an HTTP API. Testing
that API directly is faster (milliseconds, not seconds), more stable (no
locators, no waits), and catches a whole class of defects the UI hides —
wrong status codes, leaked fields, broken validation. RestAssured gives Java
a fluent DSL for it, and a well-built framework runs API and UI tests side by
side in one suite.

## 1. Setup

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>json-schema-validator</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
```

RestAssured brings Hamcrest with it, which is why the assertions below read
like English. The examples use
`https://jsonplaceholder.typicode.com` — a free, public, read-mostly fake API.

## 2. given / when / then

```java
package com.example.api;

import org.junit.jupiter.api.*;

import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assertions.*;

class PostsApiTest {

    @BeforeAll
    static void baseSetup() {
        baseURI = "https://jsonplaceholder.typicode.com";
    }

    @Test
    @DisplayName("GET /posts/1 returns the expected post")
    void getSinglePost() {
        given()
            .accept("application/json")
        .when()
            .get("/posts/1")
        .then()
            .statusCode(200)
            .contentType("application/json; charset=utf-8")
            .body("id", equalTo(1))
            .body("userId", equalTo(1))
            .body("title", not(emptyOrNullString()))
            .time(lessThan(3000L));
    }

    @Test
    @DisplayName("GET /posts returns 100 posts, all with a userId")
    void getAllPosts() {
        given()
        .when()
            .get("/posts")
        .then()
            .statusCode(200)
            .body("size()", equalTo(100))
            .body("userId", everyItem(notNullValue()))
            .body("[0].id", equalTo(1));
    }

    @Test
    @DisplayName("GET /posts/9999 returns 404")
    void missingPostReturns404() {
        given()
        .when()
            .get("/posts/9999")
        .then()
            .statusCode(404);
    }
}
```

```
Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

The three clauses map onto the test case template from Level 1 Module 02:
**given** = preconditions, **when** = the step, **then** = expected result.

## 3. Query parameters, path parameters, headers

```java
@Test
void filterByUser() {
    given()
        .queryParam("userId", 3)
        .header("Accept", "application/json")
    .when()
        .get("/posts")
    .then()
        .statusCode(200)
        .body("size()", equalTo(10))
        .body("userId", everyItem(equalTo(3)));
}

@Test
void pathParameters() {
    given()
        .pathParam("resource", "users")
        .pathParam("id", 2)
    .when()
        .get("/{resource}/{id}")
    .then()
        .statusCode(200)
        .body("username", equalTo("Antonette"))
        .body("address.city", equalTo("Wisokyburgh"))     // nested with dots
        .body("company.name", containsStringIgnoringCase("deckow"));
}
```

`address.city` is GPath — RestAssured's JSON path syntax. It also does
filtering: `body("find { it.id == 3 }.title", ...)`.

## 4. POST, PUT, PATCH, DELETE

```java
import io.restassured.http.ContentType;

@Test
void createPost() {
    String payload = """
            {
              "title": "Regression suite results",
              "body": "All 42 tests passed",
              "userId": 7
            }
            """;

    given()
        .contentType(ContentType.JSON)
        .body(payload)
    .when()
        .post("/posts")
    .then()
        .statusCode(201)
        .body("title", equalTo("Regression suite results"))
        .body("userId", equalTo(7))
        .body("id", notNullValue());
}

@Test
void updateAndDelete() {
    given().contentType(ContentType.JSON)
           .body("{\"title\":\"patched\"}")
    .when().patch("/posts/1")
    .then().statusCode(200)
           .body("title", equalTo("patched"));

    given().when().delete("/posts/1").then().statusCode(200);
}
```

### Serialising a POJO instead of a string

```java
public class PostRequest {
    public String title;
    public String body;
    public int userId;

    public PostRequest(String title, String body, int userId) {
        this.title = title; this.body = body; this.userId = userId;
    }
}

@Test
void createFromObject() {
    PostRequest request = new PostRequest("From a POJO", "typed payload", 5);

    given()
        .contentType(ContentType.JSON)
        .body(request)                       // Jackson serialises it
    .when()
        .post("/posts")
    .then()
        .statusCode(201)
        .body("title", equalTo("From a POJO"));
}
```

Hand-written JSON strings rot silently when the contract changes; a POJO
fails to compile, which is exactly the feedback you want.

## 5. Extracting the response

Chained `.body(...)` checks are fine for simple assertions. For anything you
need to compute with, extract.

```java
import io.restassured.response.Response;
import java.util.List;

@Test
void extractAndReuse() {
    Response response = given()
            .when().get("/users")
            .then().statusCode(200)
            .extract().response();

    List<String> emails = response.jsonPath().getList("email");
    List<String> names  = response.jsonPath().getList("name");

    System.out.println("Users returned: " + emails.size());

    assertEquals(10, emails.size());
    assertTrue(emails.stream().allMatch(e -> e.contains("@")),
            "Every user must have a plausible email: " + emails);
    assertEquals("Leanne Graham", names.get(0));
}

// A single value, typed
int firstId = given().when().get("/posts")
        .then().statusCode(200)
        .extract().path("[0].id");

// Chaining: use an id from one call in the next
String token = given().contentType(ContentType.JSON).body(login)
        .when().post("/auth")
        .then().statusCode(200)
        .extract().path("token");
```

## 6. Authentication

```java
// Bearer token -- by far the most common
given().header("Authorization", "Bearer " + token)
    .when().get("/orders").then().statusCode(200);

// Built-in helpers
given().auth().basic("user", "pass")     ...
given().auth().preemptive().basic("user", "pass")  ...
given().auth().oauth2(token)             ...

// API key in a header
given().header("x-api-key", System.getenv("API_KEY"))  ...
```

!!! warning "Never commit a real token"
    Read secrets from environment variables or a git-ignored properties file
    (Module 05). A token pushed to GitHub is compromised within minutes —
    bots scan public commits continuously — and rotating it is somebody's
    whole afternoon.

## 7. Request specs, logging and schema validation

```java
import io.restassured.builder.RequestSpecBuilder;
import io.restassured.specification.RequestSpecification;
import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath;

private static RequestSpecification apiSpec;

@BeforeAll
static void buildSpec() {
    apiSpec = new RequestSpecBuilder()
            .setBaseUri("https://jsonplaceholder.typicode.com")
            .setContentType(ContentType.JSON)
            .addHeader("X-Test-Run", "level2-module04")
            .build();
}

@Test
void usesSharedSpec() {
    given().spec(apiSpec)
        .when().get("/users/1")
        .then().statusCode(200)
               .body(matchesJsonSchemaInClasspath("schemas/user-schema.json"));
}
```

Schema validation is the highest-value API assertion you can write: it checks
every field's presence and type in one line, so a backend that quietly drops
`address` fails immediately instead of three sprints later.

```java
// Logging -- indispensable while debugging
given().log().all()      // full request
    .when().get("/posts/1")
    .then().log().ifValidationFails()   // response only when something breaks
           .statusCode(200);
```

`log().ifValidationFails()` is the setting to leave switched on in CI: silent
when green, fully diagnostic when red.

## 8. API + UI in one test

```java
@Test
@DisplayName("A post created through the API appears in the UI")
void apiSetupThenUiCheck() {
    int id = given().spec(apiSpec)
            .body(new PostRequest("Created via API", "seed data", 1))
            .when().post("/posts")
            .then().statusCode(201)
            .extract().path("id");

    driver.get("https://example.test/posts/" + id);
    assertEquals("Created via API", new PostPage(driver).getTitle());
}
```

Setting up state through the API and asserting through the UI is the single
biggest speed-up available to a UI suite — a login that takes 4 seconds
through the form takes 200 ms as a token injected into a cookie.

## 9. Testing traps

!!! warning "Trap 1 — asserting status code only"
    `statusCode(200)` passes for an endpoint returning an empty list when it
    should return ten records. Always assert the body as well.

!!! warning "Trap 2 — `equalTo(1)` vs `equalTo(1L)`"
    GPath returns `Integer` for small JSON numbers and `Long`/`BigDecimal`
    for large ones. `equalTo(1)` fails against a `Long` with the unhelpful
    message "expected 1 but was 1". Extract and compare typed values when in
    doubt.

!!! warning "Trap 3 — tests that depend on data someone else can change"
    `body("size()", equalTo(100))` breaks the moment a colleague seeds a
    record on the shared environment. Create your own data in the test, or
    assert relative properties (`greaterThan(0)`).

!!! warning "Trap 4 — fake APIs that fake success"
    JSONPlaceholder returns `201` and an id for every POST but does not
    actually store anything, so a follow-up GET returns 404. Excellent for
    learning the DSL; never mistake it for evidence that your assertions
    would pass against a real backend.

!!! warning "Trap 5 — no timeout"
    A hung endpoint blocks the suite indefinitely. Configure connection and
    socket timeouts in a `RestAssuredConfig`, and assert `time(lessThan(...))`
    on endpoints with a performance requirement.

## Cheat sheet

| Task | Code |
|---|---|
| Base URI | `RestAssured.baseURI = "https://api.example.com"` |
| GET | `given().when().get("/path").then()` |
| Query param | `.queryParam("userId", 3)` |
| Path param | `.pathParam("id", 2)` + `get("/users/{id}")` |
| Header | `.header("Authorization", "Bearer " + token)` |
| JSON body | `.contentType(ContentType.JSON).body(payload)` |
| Status | `.statusCode(200)` |
| Field | `.body("id", equalTo(1))` |
| Nested field | `.body("address.city", equalTo("Wisokyburgh"))` |
| List size | `.body("size()", equalTo(100))` |
| Every item | `.body("userId", everyItem(notNullValue()))` |
| Response time | `.time(lessThan(2000L))` |
| Extract value | `.extract().path("token")` |
| Extract list | `.extract().jsonPath().getList("email")` |
| Schema | `.body(matchesJsonSchemaInClasspath("schemas/user.json"))` |
| Shared config | `RequestSpecBuilder` + `given().spec(apiSpec)` |
| Debug logging | `.log().all()` / `.log().ifValidationFails()` |

## Exercise

Use `https://jsonplaceholder.typicode.com`.

1. Write `PostsApiTest` with the three tests from section 2, then add: a
   negative test for `/posts/abc`, and a test asserting every post has a
   non-empty `title` and a `userId` between 1 and 10.
2. Write `UsersApiTest` covering `GET /users` (10 users), `GET /users/1`
   (nested `address.geo.lat` is present), and `GET /users?username=Bret`
   (exactly one match, id 1).
3. Build a `RequestSpecBuilder` spec and refactor both classes onto it.
   Count how many lines you deleted.
4. Write `schemas/user-schema.json` requiring `id`, `name`, `username`,
   `email` and an `address` object, and assert it. Then remove `email` from
   the required list and confirm the test still passes — this is how you
   prove a schema assertion is actually running.
5. POST a new post from a `PostRequest` POJO, extract the returned `id`, and
   then GET that id. Explain in two sentences what happens and why the fake
   API behaves that way (trap 4).
6. Add `.time(lessThan(2000L))` to every test, run the suite five times, and
   record whether any run breached it. Decide whether 2000 ms is the right
   budget for a public API over your connection, and justify the number you
   would put in a test plan.
7. Take three test cases from your Level 1 manual test-case document that are
   really API behaviours (validation, error codes, permissions) and rewrite
   them as RestAssured tests. Note how much faster they run than the UI
   equivalents.
