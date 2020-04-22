# PHP Swagger Test
[![Build Status](https://travis-ci.org/byjg/php-swagger-test.svg?branch=master)](https://travis-ci.org/byjg/php-swagger-test)
[![Maintainable Rate](https://sonarcloud.io/api/project_badges/measure?project=php-swagger-test&metric=sqale_rating)](https://sonarcloud.io/dashboard?id=php-swagger-test)
[![Reliability Rate](https://sonarcloud.io/api/project_badges/measure?project=php-swagger-test&metric=reliability_rating)](https://sonarcloud.io/dashboard?id=php-swagger-test)
[![Security Rate](https://sonarcloud.io/api/project_badges/measure?project=php-swagger-test&metric=security_rating)](https://sonarcloud.io/dashboard?id=php-swagger-test)
[![Quality Gate](https://sonarcloud.io/api/project_badges/measure?project=php-swagger-test&metric=alert_status)](https://sonarcloud.io/dashboard?id=php-swagger-test)
[![Code Coverage](https://sonarcloud.io/api/project_badges/measure?project=php-swagger-test&metric=coverage)](https://sonarcloud.io/dashboard?id=php-swagger-test)
[![Bugs](https://sonarcloud.io/api/project_badges/measure?project=php-swagger-test&metric=bugs)](https://sonarcloud.io/dashboard?id=php-swagger-test)
[![Code Smells](https://sonarcloud.io/api/project_badges/measure?project=php-swagger-test&metric=code_smells)](https://sonarcloud.io/dashboard?id=php-swagger-test)
[![Techinical Debt](https://sonarcloud.io/api/project_badges/measure?project=php-swagger-test&metric=sqale_index)](https://sonarcloud.io/dashboard?id=php-swagger-test)
[![Vulnerabilities](https://sonarcloud.io/api/project_badges/measure?project=php-swagger-test&metric=vulnerabilities)](https://sonarcloud.io/dashboard?id=php-swagger-test)

[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/byjg/php-swagger-test/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/byjg/php-swagger-test/?branch=master)


A set of tools for test your REST calls based on the OpenApi specification using PHPUnit. 
Currently, this library supports the OpenApi specifications `2.0` (former swagger) and `3.0`.
Some features like callbacks, links and references to external documents/objects weren't implemented. 

PHP Swagger Test can help you to test your REST Api. You can use this tool both for Unit Tests or Functional Tests.

This tool reads a previously Swagger JSON file (not YAML) and enable you to test the request and response. 
You can use the tool "https://github.com/zircote/swagger-php" for create the JSON file when you are developing your
rest API. 

The ApiTestCase's assertion process is based on throwing exceptions if some validation or test failed.

# Use cases for PHP Swagger test

You can use the Swagger Test library as:

- Functional test cases
- Unit test cases
- Runtime parameters validator
- Validate your specification


## Functional Test cases

Swagger Test provide the class `SwaggerTestCase` for you extend and create a PHPUnit test case. The code will try to 
make a request to your API Method and check if the request parameters, status and object returned are OK.

```php
<?php
/**
 * Create a TestCase inherited from SwaggerTestCase
 */
class MyTestCase extends \ByJG\ApiTools\ApiTestCase
{
    public function setUp()
    {
        $schema = \ByJG\ApiTools\Base\Schema::getInstance(file_get_contents('/path/to/json/definition'));
        $this->setSchema($schema);
    }
    
    /**
     * Test if the REST address /path/for/get/ID with the method GET returns what is
     * documented on the "swagger.json"
     */
    public function testGet()
    {
        $request = new \ByJG\ApiTools\ApiRequester();
        $request
            ->withMethod('GET')
            ->withPath("/path/for/get/1");

        $this->assertRequest($request);
    }

    /**
     * Test if the REST address /path/for/get/NOTFOUND returns a status code 404.
     */
    public function testGetNotFound()
    {
        $request = new \ByJG\ApiTools\ApiRequester();
        $request
            ->withMethod('GET')
            ->withPath("/path/for/get/NOTFOUND")
            ->assertResponseCode(404);

        $this->assertRequest($request);
    }

    /**
     * Test if the REST address /path/for/post/ID with the method POST  
     * and the request object ['name'=>'new name', 'field' => 'value'] will return an object
     * as is documented in the "swagger.json" file
     */
    public function testPost()
    {
        $request = new \ByJG\ApiTools\ApiRequester();
        $request
            ->withMethod('POST')
            ->withPath("/path/for/post/2")
            ->withRequestBody(['name'=>'new name', 'field' => 'value']);

        $this->assertRequest($request);
    }

    /**
     * Test if the REST address /another/path/for/post/{id} with the method POST  
     * and the request object ['name'=>'new name', 'field' => 'value'] will return an object
     * as is documented in the "swagger.json" file
     */
    public function testPost2()
    {
        $request = new \ByJG\ApiTools\ApiRequester();
        $request
            ->withMethod('POST')
            ->withPath("/path/for/post/3")
            ->withQuery(['id'=>10])
            ->withRequestBody(['name'=>'new name', 'field' => 'value']);

        $this->assertRequest($request);
    }

}
```

### Functional Tests without a Webserver

Sometimes, you want to run functional tests without making the actual HTTP
requests and without setting up a webserver for that. Instead, you forward the
requests to the routing of your application kernel which lives in the same
process as the functional tests. In order to do that, you need a bit of
gluecode based on the `AbstractRequester` baseclass:

```php
class MyAppRequester extends ByJG\ApiTools\AbstractRequester
{
    /** @var MyAppKernel */
    private $app;

    public function __construct(MyAppKernel $app)
    {
        parent::construct();
        $this->app = $app;
    }

    protected function handleRequest(RequestInterface $request)
    {
        return $this->app->handle($request);
    }
}
```
You now use an instance of this class in place of the `ApiRequester` class from the examples above. Of course, if you need to apply changes to the request or the response in order
to fit your framework, this is exactly the right place to do it.

@todo: Explain the WebRequest MockHttpClient.
@todo: Explain in the Docs sections the `RestServer` component 

## Using it as Unit Test cases

If you want mock the request API and just test the expected parameters you are sending and 
receiving you have to:

**1. Create the Swagger or OpenAPI Test Schema**

```php
<?php
$schema = \ByJG\ApiTools\Base\Schema::getInstance($contentsOfSchemaJson);
```

**2. Get the definitions for your path**

```php
<?php
$path = '/path/to/method';
$statusExpected = 200;
$method = 'POST';

// Returns a SwaggerRequestBody instance
$bodyRequestDef = $schema->getRequestParameters($path, $method);

// Returns a SwaggerResponseBody instance
$bodyResponseDef = $schema->getResponseParameters($path, $method, $statusExpected);
```

**3. Match the result**

```php
<?php
if (!empty($requestBody)) {
    $bodyRequestDef->match($requestBody);
}
$bodyResponseDef->match($responseBody);
```

If the request or response body does not match with the definition an exception NotMatchException will
be throwed. 

## Runtime parameters validator

This tool was not developed only for unit and functional tests. You can use to validate if the required body
parameters is the expected. 

So, before your API Code you can validate the request body using:

```php
<?php
$schema = \ByJG\ApiTools\Base\Schema::getInstance($contentsOfSchemaJson);
$bodyRequestDef = $schema->getRequestParameters($path, $method);
$bodyRequestDef->match($requestBody);
```

## Validate your Specification

PHP Swagger has the class `MockRequester` with exact the same functionalities of `ApiRequester` class. The only
difference is the `MockRequester` don't need to request to a real endpoint.

This is particularly usefull if you want to check if your OpenApi specification is returning the expected values. 

```php
<?php
class MyTest extends ApiTestCase
{
    public function testExpectOK()
    {
        $expectedResponse = \ByJG\Util\Psr7\Response::getInstance(200)
            ->withBody(new MemoryStream(json_encode([
                "id" => 1,
                "name" => "Spike",
                "photoUrls" => []
            ])));

        // The MockRequester does not send the request to a real endpoint
        // Just returning the expected Response object sent in the constructor 
        $request = new \ByJG\ApiTools\MockRequester($expectedResponse);
        $request
            ->withMethod('GET')
            ->withPath("/pet/1");

        $this->assertRequest($request); // That should be "True"
    }
}
```   

# Install

```
composer require "byjg/swagger-test=3.1.*"
```

# Questions?

Github issue.

# References

This project uses the [byjg/webrequest](https://github.com/byjg/webrequest) component. It implements the PSR-7 specification, 
and a HttpClient / MockClient to do the requests. Check it out to get more information. 

---
OpenSource ByJG: [https://opensource.byjg.com/](https://opensource.byjg.com/)
