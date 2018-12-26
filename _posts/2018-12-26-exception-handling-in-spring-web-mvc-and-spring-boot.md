---
title: " Exception handling in Spring Web MVC and Spring Boot"
categories:
  - spring-boot
  - spring-web-mvc
---

When an exception is thrown during service proccess a request, sometime we need return a response with error status code (status code <> 200) and a message to inform user the request is not processed successfully. To do it  in Spring Web MVC, we has many solutions, this article will show 3 commons solution to handle exceptions and return appropriate response.


## Approach 1: Use `ResponseStatus` and CustomClassException class

```java

// CustomerController class

public class CustomerController {
  @GetMapping("/customers")
  public List<Customer> getAllCustomers(HttpServletRequest req) {
    try {
      List<Customer> customerList;
      customerList = customerService.getCustomerList();
      return customerList;
    } catch (Exception ex) {
      String errorMessage = "Fail to get customer list from database";
      log.error(errorMessage, ex);
      throw new CustomerListErrorException(errorMessage);
    }
  }
 }
 
 // CustomerListErrorException class
 
 @ResponseStatus(value= HttpStatus.INTERNAL_SERVER_ERROR)
 public class CustomerListErrorException extends RuntimeException {
   public CustomerListErrorException(String msg){
     super(msg);
   }
 }
```

Example Response:

```json
//GET http://localhost:8071/v1/customers

//HTTP/1.1 500 
//Content-Type: application/json;charset=UTF-8
//Transfer-Encoding: chunked
//Date: Wed, 26 Dec 2018 11:01:08 GMT
//Connection: close

{
  "timestamp": "2018-12-26T11:01:08.744+0000",
  "status": 500,
  "error": "Internal Server Error",
  "message": "Fail to get customer list from database",
  "path": "/v1/customers"
}

//Response code: 500; Time: 112239ms; Content length: 164 bytes

```


## Approach 2: Using `@ExceptionHandler` for handle Exception for a Single Controller

```java
@Slf4j
@RestController
@RequestMapping("/v1")
public class CustomerController {

  private CustomerService customerService;

  @Autowired
  public CustomerController(CustomerService customerService) {
    this.customerService = customerService;
  }

  @GetMapping(value = "/customers", params = "id")
  public CustomerDTO getCustomer(@RequestParam("id") Long id) throws CustomerNotFoundException {
    Customer customer = customerService.getCustomer(id);
    if (customer == null) {
      throw new CustomerNotFoundException(id);
    }
    return customerService.getCustomerDetail(customer);

  }

  @ResponseStatus(HttpStatus.NOT_FOUND)
  @ExceptionHandler(CustomerNotFoundException.class)
  @ResponseBody HTTPErrorResponseDTO handleCustomerNotFoundException(HttpServletRequest req, Exception ex) {
    log.warn(ex.getMessage());
    return new HTTPErrorResponseDTO(HttpStatus.NOT_FOUND.value(), ex.getMessage());
  }

}
```

Example Response:


```json
//GET http://localhost:8071/v1/customers?id=4
//HTTP/1.1 404 
//Content-Type: application/json;charset=UTF-8
//Transfer-Encoding: chunked
//Date: Wed, 26 Dec 2018 10:46:53 GMT
{
  "statusCode": 404,
  "errorMessage": "Customer 4 not found!"
}
//Response code: 404; Time: 270ms; Content length: 57 bytes

```

## Approach 3: Using `@ExceptionHandler` with `@ControllerAdvice` for handle Exception for All Controller, if exception is not handled in this controller.

```java

// class CustomerController

package com.spring.ws.controller.v1;

@Slf4j
@RestController
@RequestMapping("/v1")
public class CustomerController {

  private CustomerService customerService;

  @Autowired
  public CustomerController(CustomerService customerService) {
    this.customerService = customerService;
  }

  @GetMapping("/customers")
  public List<Customer> getAllCustomers(HttpServletRequest req) {
    try {
      List<Customer> customerList;
      customerList = customerService.getCustomerList();
      return customerList;
    } catch (Exception ex) {
      String errorMessage = "Fail to get customer list from database";
      log.error(errorMessage, ex);
      throw new CustomerListErrorException(errorMessage);
    }
  }

  @GetMapping(value = "/customers", params = "id")
  public CustomerDTO getCustomer(@RequestParam("id") Long id) throws CustomerNotFoundException {
    Customer customer = customerService.getCustomer(id);
    if (customer == null) {
      throw new CustomerNotFoundException(id);
    }
    return customerService.getCustomerDetail(customer);

  }
}

// class CustomerServiceExceptionHandler
package com.spring.ws.exceptions;

@ControllerAdvice
@Slf4j
public class CustomerServiceExceptionHandler {


  @ResponseStatus(HttpStatus.NOT_FOUND)
  @ExceptionHandler(CustomerNotFoundException.class)
  @ResponseBody HTTPErrorResponseDTO handleCustomerNotFoundException(
      HttpServletRequest req, CustomerNotFoundException ex) {
    log.warn(ex.getMessage());
    return new HTTPErrorResponseDTO(HttpStatus.NOT_FOUND.value(), ex.getMessage());
  }

  @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
  @ExceptionHandler(CustomerListErrorException.class)
  @ResponseBody HTTPErrorResponseDTO handleCustomerListErrorException(
      HttpServletRequest req, CustomerListErrorException ex) {
    log.error(ex.getMessage());
    return new HTTPErrorResponseDTO(HttpStatus.INTERNAL_SERVER_ERROR.value(), ex.getMessage());
  }
}

```


```json
//GET http://localhost:8071/v1/customers

//HTTP/1.1 500 
//Content-Type: application/json;charset=UTF-8
//Transfer-Encoding: chunked
//Date: Wed, 26 Dec 2018 11:21:47 GMT
//Connection: close

{
  "statusCode": 500,
  "errorMessage": "Fail to get customer list from database"
}

//Response code: 500; Time: 32637ms; Content length: 75 bytes


//GET http://localhost:8071/v1/customers?id=4
//HTTP/1.1 404 
//Content-Type: application/json;charset=UTF-8
//Transfer-Encoding: chunked
//Date: Wed, 26 Dec 2018 10:46:53 GMT
{
  "statusCode": 404,
  "errorMessage": "Customer 4 not found!"
}
//Response code: 404; Time: 270ms; Content length: 57 bytes

```
## Documents and References

- <https://spring.io/blog/2013/11/01/exception-handling-in-spring-mvc>
- <https://blog.jayway.com/2014/10/19/spring-boot-error-responses/>
- <https://docs.spring.io/spring-framework/docs/4.1.x/spring-framework-reference/html/mvc.html#mvc-ann-controller-advice>
- <https://www.javadevjournal.com/spring/exception-handling-for-rest-with-spring/>
- <https://dzone.com/articles/leverage-http-status-codes-to-build-a-rest-service>

## Sample projects

- <https://github.com/paulc4/mvc-exceptions>