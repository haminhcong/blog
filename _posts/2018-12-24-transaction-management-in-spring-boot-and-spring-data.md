---
title: "Transaction management in Spring with Spring data and Spring Boot"
categories:
  - spring-boot
  - spring-data
---

Transaction management play a important role in web service application. It help changes user made into database via web service is proccessed in a consistency way, follow ACID properties of a database. In this articles, i will test spring transaction management behavior.


The way spring framework manages transactions is described in article [how-does-spring-transactional](https://dzone.com/articles/how-does-spring-transactional)

> The annotation @EnableTransactionManagement tells Spring that classes with the @Transactional annotation should be wrapped with the Transactional Aspect. With this the @Transactional is now ready to be used.


Because of this, if we need enable Transaction Management in application, we need config `EnableTransactionManagement` to Spring Configuration, and annotated `@Transactional` on classes or method need database transaction. In Spring Boot, with AutoConfiguration enabled, application will automatic enable `@EnableTransactionManagement`:

- [TransactionAutoConfiguration.java#L88](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/transaction/TransactionAutoConfiguration.java#L88) 

- [@EnableTransactionManagement in Spring Boot](https://stackoverflow.com/a/50446664)

A important question we need to solve when we use spring transaction management are:

- Which methods annotated by `@Transaction` annotation is in the same transaction ?

To control it, we use [spring transaction propagation](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html) in correct **transaction context**. Two most Propagation mode used are `Propagation.REQUIRED` (default mode) and `Propagation.REQUIRES_NEW `:

- With `Propagation.REQUIRED` mode used in called methods, we have two cases:
  -  If caller method is a transactional-context method, called methods is in the same transaction with caller method and same with other called method. If a exception throwed in any called method, the transaction used in all methods will be rolled back and changes will not be commited.
  -  If callser method is not a transactional-context method, called methods will be in separated transactions with each other. In this cases, the behavior will be the same as when we use `Propagation.REQUIRED_NEW`. If a exception throwed in a method, the transaction used in this method will be rolled back, but it will not affect to transactions in other methods.
- With `Propagation.REQUIRES_NEW` mode used in called methods, the called methods will always in separated transaction will caller method, and alose in separated transaction with other called method. If a exception throwed in a method, the transaction used in this method will be rolled back, but it will not affect to transactions in other methods.

Examples:

- Database has two entity: `Address` and `Customer`:

```java
@Entity
@Table(name = "address")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Address {

  @Id
  @Column(name = "id")
  @SequenceGenerator(name="address_seq", sequenceName = "address_seq",allocationSize=100)
  @GeneratedValue(generator="address_seq")
  private Long id;

  @Column(name="address")
  private String address;
}


@Entity
@Table(name = "customer")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Customer {

  @Id
  @Column(name = "id")
  @SequenceGenerator(name="customer_seq", sequenceName = "customer_seq",allocationSize=100)
  @GeneratedValue(generator="customer_seq")
  private Long id;

  @Column(name = "account_name", unique = true, nullable = false)
  private String accountName;

}

```



- When we use `Propagation.REQUIRED` (default Propagation mode) with caller is non-transactional context:

```java
// Caller method
  @GetMapping(value = "/test-transaction-without-transaction-context")
  public String testTransaction() {
    log.info("Before test transaction");
    testTransactionService.printCurrentData();
    try{
      testTransactionService.testTransactionWithoutTransactionContext();
    }catch (Exception e){
      log.error("Error ", e);
    }
    log.info("After test transaction");
    testTransactionService.printCurrentData();
    return "request-done";
  }

  public void testTransactionWithoutTransactionContext(){
    log.info("test-transaction");

    List<Customer> initCustomerList =
        createCustomerList("customer init 1", "customer init 2");
    List<Address> initAddressList =
        createAddressList("address init 1", "address init 2");

    testTransactionDAO.addData(initCustomerList, initAddressList);

    try {
      List<Customer> testCustomerList =
          createCustomerList("test customer 1", "test customer 2");
      List<Address> testAddressList =
          createAddressList("test address 1", "test address 2");
      testTransactionDAO.addDataException(testCustomerList, testAddressList);
    } catch (Exception e) {
      log.error("Error", e);
    }

    List<Customer> testCustomerList =
        createCustomerList("test customer 3", "test customer 4");
    List<Address> testAddressList =
        createAddressList("test address 3", "test address 4");
    testTransactionDAO.addData(testCustomerList, testAddressList);
  }



// Called methods

  @Transactional(rollbackFor = Exception.class)
  public void addData(List<Customer> customerList, List<Address> addressList) {
    this.customerRepository.saveAll(customerList);
    this.addressRepository.saveAll(addressList);
  }

  @Transactional(rollbackFor = Exception.class)
  public void addDataException(List<Customer> customerList, List<Address> addressList)
      throws Exception {
    this.customerRepository.saveAll(customerList);
    this.addressRepository.saveAll(addressList);
    throw new Exception("test-transaction");
  }

  public List<Customer> getCustomerList() {
    return customerRepository.findAll();
  }

  public List<Address> getAddressList() {
    return addressRepository.findAll();
  }

```

we will get result:

```text
Hibernate: select customer0_.id as id1_1_, customer0_.account_name as account_2_1_, customer0_.address as address3_1_, customer0_.gender as gender4_1_, customer0_.password as password5_1_, customer0_.phone_number as phone_nu6_1_, customer0_.user_name as user_nam7_1_ from customer customer0_
Hibernate: select address0_.id as id1_0_, address0_.address as address2_0_ from address address0_
2018-12-24 23:51:32.865  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Customer List: 
2018-12-24 23:51:32.865  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Address List: 
2018-12-24 23:51:32.866  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : test-transaction
Hibernate: call next value for customer_seq
Hibernate: call next value for customer_seq
Hibernate: call next value for address_seq
Hibernate: call next value for address_seq
Hibernate: insert into customer (account_name, address, gender, password, phone_number, user_name, id) values (?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into customer (account_name, address, gender, password, phone_number, user_name, id) values (?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into address (address, id) values (?, ?)
Hibernate: insert into address (address, id) values (?, ?)
2018-12-24 23:51:32.965 ERROR 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Error

java.lang.Exception: test-transaction
	at com.spring.ws.service.TestTransactionDAO.addDataException(TestTransactionDAO.java:38) ~[classes/:na]
	at com.spring.ws.service.TestTransactionDAO$$FastClassBySpringCGLIB$$c341fb7d.invoke(<generated>) ~[classes/:na]

Hibernate: insert into customer (account_name, address, gender, password, phone_number, user_name, id) values (?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into customer (account_name, address, gender, password, phone_number, user_name, id) values (?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into address (address, id) values (?, ?)
Hibernate: insert into address (address, id) values (?, ?)
2018-12-24 23:51:32.973  INFO 10558 --- [nio-8071-exec-1] c.s.ws.controller.v1.CustomerController  : After test transaction
Hibernate: select customer0_.id as id1_1_, customer0_.account_name as account_2_1_, customer0_.address as address3_1_, customer0_.gender as gender4_1_, customer0_.password as password5_1_, customer0_.phone_number as phone_nu6_1_, customer0_.user_name as user_nam7_1_ from customer customer0_
Hibernate: select address0_.id as id1_0_, address0_.address as address2_0_ from address address0_
2018-12-24 23:51:32.983  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Customer List: 
2018-12-24 23:51:32.983  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 1 - customer init 1
2018-12-24 23:51:32.984  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 2 - customer init 2
2018-12-24 23:51:32.984  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 5 - test customer 3
2018-12-24 23:51:32.984  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 6 - test customer 4
2018-12-24 23:51:32.984  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Address List: 
2018-12-24 23:51:32.984  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 1 - address init 1
2018-12-24 23:51:32.984  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 2 - address init 2
2018-12-24 23:51:32.984  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 5 - test address 3
2018-12-24 23:51:32.984  INFO 10558 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 6 - test address 4

```

In this example, we can see that from non transactional context we called 3 transactional method, and 2nd transaction throw an Exception but this is not affect to first and last called transactional method, and changes in these transaction are still commited.


- When we use `Propagation.REQUIRED` with caller is a transactional context:

```java

```java
// Caller method
  @GetMapping(value = "/test-transaction-with-transaction-context")
  public String testTransaction2() {
    log.info("Before test transaction");
    testTransactionService.printCurrentData();
    try {
      testTransactionService.testTransactionWithTransactionContext();
    } catch (Exception e) {
      log.error("Error ", e);
    }
    log.info("After test transaction");
    testTransactionService.printCurrentData();
    return "request-done";
  }

  @Transactional(rollbackFor = Exception.class)
  public void testTransactionWithTransactionContext(){
    log.info("test-transaction");

    List<Customer> initCustomerList =
        createCustomerList("customer init 1", "customer init 2");
    List<Address> initAddressList =
        createAddressList("address init 1", "address init 2");

    testTransactionDAO.addData(initCustomerList, initAddressList);

    try {
      List<Customer> testCustomerList =
          createCustomerList("test customer 1", "test customer 2");
      List<Address> testAddressList =
          createAddressList("test address 1", "test address 2");
      testTransactionDAO.addDataException(testCustomerList, testAddressList);
    } catch (Exception e) {
      log.error("Error", e);
    }

    List<Customer> testCustomerList =
        createCustomerList("test customer 3", "test customer 4");
    List<Address> testAddressList =
        createAddressList("test address 3", "test address 4");
    testTransactionDAO.addData(testCustomerList, testAddressList);
  }


// Called methods

  @Transactional(rollbackFor = Exception.class)
  public void addData(List<Customer> customerList, List<Address> addressList) {
    this.customerRepository.saveAll(customerList);
    this.addressRepository.saveAll(addressList);
  }

  @Transactional(rollbackFor = Exception.class)
  public void addDataException(List<Customer> customerList, List<Address> addressList)
      throws Exception {
    this.customerRepository.saveAll(customerList);
    this.addressRepository.saveAll(addressList);
    throw new Exception("test-transaction");
  }

  public List<Customer> getCustomerList() {
    return customerRepository.findAll();
  }

  public List<Address> getAddressList() {
    return addressRepository.findAll();
  }

```

we will get result:

```text
Hibernate: select customer0_.id as id1_1_, customer0_.account_name as account_2_1_, customer0_.address as address3_1_, customer0_.gender as gender4_1_, customer0_.password as password5_1_, customer0_.phone_number as phone_nu6_1_, customer0_.user_name as user_nam7_1_ from customer customer0_
Hibernate: select address0_.id as id1_0_, address0_.address as address2_0_ from address address0_
2018-12-24 23:54:20.683  INFO 10909 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Customer List: 
2018-12-24 23:54:20.684  INFO 10909 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Address List: 
2018-12-24 23:54:20.685  INFO 10909 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : test-transaction
Hibernate: call next value for customer_seq
Hibernate: call next value for customer_seq
Hibernate: call next value for address_seq
Hibernate: call next value for address_seq
2018-12-24 23:54:20.797 ERROR 10909 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Error

java.lang.Exception: test-transaction
	at com.spring.ws.service.TestTransactionDAO.addDataException(TestTransactionDAO.java:38) ~[classes/:na]
	at com.spring.ws.service.TestTransactionDAO$$FastClassBySpringCGLIB$$c341fb7d.invoke(<generated>) ~[classes/:na]

2018-12-24 23:54:20.823 ERROR 10909 --- [nio-8071-exec-1] c.s.ws.controller.v1.CustomerController  : Error 

org.springframework.orm.jpa.JpaSystemException: Transaction was marked for rollback only; cannot commit; nested exception is org.hibernate.TransactionException: Transaction was marked for rollback only; cannot commit
	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.convertHibernateAccessException(HibernateJpaDialect.java:312) ~[spring-orm-5.0.9.RELEASE.jar:5.0.9.RELEASE]
	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible(HibernateJpaDialect.java:223) ~[spring-orm-5.0.9.RELEASE.jar:5.0.9.RELEASE]
	at org.springframework.orm.jpa.JpaTransactionManager.doCommit(JpaTransactionManager.java:540) ~[spring-orm-5.0.9.RELEASE.jar:5.0.9.RELEASE]
	at 
Caused by: org.hibernate.TransactionException: Transaction was marked for rollback only; cannot commit
	at org.hibernate.resource.transaction.backend.jdbc.internal.JdbcResourceLocalTransactionCoordinatorImpl$TransactionDriverControlImpl.commit(JdbcResourceLocalTransactionCoordinatorImpl.java:228) ~[hibernate-core-5.2.17.Final.jar:5.2.17.Final]
	at org.hibernate.engine.transaction.internal.TransactionImpl.commit(TransactionImpl.java:68) ~[hibernate-core-5.2.17.Final.jar:5.2.17.Final]
	at org.springframework.orm.jpa.JpaTransactionManager.doCommit(JpaTransactionManager.java:536) ~[spring-orm-5.0.9.RELEASE.jar:5.0.9.RELEASE]
	... 73 common frames omitted

2018-12-24 23:54:20.824  INFO 10909 --- [nio-8071-exec-1] c.s.ws.controller.v1.CustomerController  : After test transaction
Hibernate: select customer0_.id as id1_1_, customer0_.account_name as account_2_1_, customer0_.address as address3_1_, customer0_.gender as gender4_1_, customer0_.password as password5_1_, customer0_.phone_number as phone_nu6_1_, customer0_.user_name as user_nam7_1_ from customer customer0_
Hibernate: select address0_.id as id1_0_, address0_.address as address2_0_ from address address0_
2018-12-24 23:54:20.828  INFO 10909 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Customer List: 
2018-12-24 23:54:20.828  INFO 10909 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Address List: 

```

we can see that, when call from transactional context, no any changes from 3  caller transactional method is commited to database, because they share same transaction, and when a Exception throwed, this transaction will be rollback and not commit any thing in this transaction to database.


- When we use `Propagation.REQUIRE_NEW` mode:

```java
// Caller method
  @GetMapping(value = "/test-transaction-with-transaction-require-new")
  public String testTransaction3() {
    log.info("Before test transaction");
    testTransactionService.printCurrentData();
    try {
      testTransactionService.testTransactionRequireNewTransaction();
    } catch (Exception e) {
      log.error("Error ", e);
    }
    log.info("After test transaction");
    testTransactionService.printCurrentData();
    return "request-done";
  }

  @Transactional(rollbackFor = Exception.class)
  public void testTransactionRequireNewTransaction(){
    log.info("test-transaction");

    List<Customer> initCustomerList =
        createCustomerList("customer init 1", "customer init 2");
    List<Address> initAddressList =
        createAddressList("address init 1", "address init 2");

    testTransactionDAO.addDataRequireNewTransaction(initCustomerList, initAddressList);

    try {
      List<Customer> testCustomerList =
          createCustomerList("test customer 1", "test customer 2");
      List<Address> testAddressList =
          createAddressList("test address 1", "test address 2");
      testTransactionDAO.addDataExceptionRequireNewTransaction(testCustomerList, testAddressList);
    } catch (Exception e) {
      log.error("Error", e);
    }

    List<Customer> testCustomerList =
        createCustomerList("test customer 3", "test customer 4");
    List<Address> testAddressList =
        createAddressList("test address 3", "test address 4");
    testTransactionDAO.addDataRequireNewTransaction(testCustomerList, testAddressList);
  }

// Called method
  @Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRES_NEW)
  public void addDataRequireNewTransaction(List<Customer> customerList, List<Address> addressList) {
    this.customerRepository.saveAll(customerList);
    this.addressRepository.saveAll(addressList);
  }

  @Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRES_NEW)
  public void addDataExceptionRequireNewTransaction(List<Customer> customerList, List<Address> addressList)
      throws Exception {
    this.customerRepository.saveAll(customerList);
    this.addressRepository.saveAll(addressList);
    throw new Exception("test-transaction");
  }

```

We will get the result

```text
2018-12-24 23:59:19.814  INFO 11607 --- [nio-8071-exec-1] c.s.ws.controller.v1.CustomerController  : Before test transaction
2018-12-24 23:59:19.863  INFO 11607 --- [nio-8071-exec-1] o.h.h.i.QueryTranslatorFactoryInitiator  : HHH000397: Using ASTQueryTranslatorFactory
Hibernate: select customer0_.id as id1_1_, customer0_.account_name as account_2_1_, customer0_.address as address3_1_, customer0_.gender as gender4_1_, customer0_.password as password5_1_, customer0_.phone_number as phone_nu6_1_, customer0_.user_name as user_nam7_1_ from customer customer0_
Hibernate: select address0_.id as id1_0_, address0_.address as address2_0_ from address address0_
2018-12-24 23:59:19.998  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Customer List: 
2018-12-24 23:59:19.998  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Address List: 
2018-12-24 23:59:19.998  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : test-transaction
Hibernate: call next value for customer_seq
Hibernate: call next value for customer_seq
Hibernate: call next value for address_seq
Hibernate: call next value for address_seq
Hibernate: insert into customer (account_name, address, gender, password, phone_number, user_name, id) values (?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into customer (account_name, address, gender, password, phone_number, user_name, id) values (?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into address (address, id) values (?, ?)
Hibernate: insert into address (address, id) values (?, ?)
2018-12-24 23:59:20.056 ERROR 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Error

java.lang.Exception: test-transaction
	at com.spring.ws.service.TestTransactionDAO.addDataExceptionRequireNewTransaction(TestTransactionDAO.java:53) ~[classes/:na]
	at com.spring.ws.service.TestTransactionDAO$$FastClassBySpringCGLIB$$c341fb7d.invoke(<generated>) ~[classes/:na]
	at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204) [spring-core-5.0.9.RELEASE.jar:5.0.9.RELEASE]
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:746) [spring-aop-5.0.9.RELEASE.jar:5.0.9.RELEASE]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163) [spring-aop-5.0.9.RELEASE.jar:5.0.9.RELEASE]
	at com.spring.ws.service.TestTransactionService$$EnhancerBySpringCGLIB$$dd724bdf.testTransactionRequireNewTransaction(<generated>) ~[classes/:na]
	at com.spring.ws.controller.v1.CustomerController.testTransaction3(CustomerController.java:82) ~[classes/:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]

Hibernate: insert into customer (account_name, address, gender, password, phone_number, user_name, id) values (?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into customer (account_name, address, gender, password, phone_number, user_name, id) values (?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into address (address, id) values (?, ?)
Hibernate: insert into address (address, id) values (?, ?)
2018-12-24 23:59:20.062  INFO 11607 --- [nio-8071-exec-1] c.s.ws.controller.v1.CustomerController  : After test transaction
Hibernate: select customer0_.id as id1_1_, customer0_.account_name as account_2_1_, customer0_.address as address3_1_, customer0_.gender as gender4_1_, customer0_.password as password5_1_, customer0_.phone_number as phone_nu6_1_, customer0_.user_name as user_nam7_1_ from customer customer0_
Hibernate: select address0_.id as id1_0_, address0_.address as address2_0_ from address address0_
2018-12-24 23:59:20.069  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Customer List: 
2018-12-24 23:59:20.070  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 1 - customer init 1
2018-12-24 23:59:20.070  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 2 - customer init 2
2018-12-24 23:59:20.070  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 5 - test customer 3
2018-12-24 23:59:20.070  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 6 - test customer 4
2018-12-24 23:59:20.070  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : Address List: 
2018-12-24 23:59:20.070  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 1 - address init 1
2018-12-24 23:59:20.070  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 2 - address init 2
2018-12-24 23:59:20.070  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 5 - test address 3
2018-12-24 23:59:20.070  INFO 11607 --- [nio-8071-exec-1] c.s.ws.service.TestTransactionService    : 6 - test address 4
```

We can see that, when called method is annotated with `Propagation.REQUIRE_NEW` mode, when this method is executed, if caller method is in a transactional context, this current transaction will be suspended, and a new transaction is created for called method. If when called method execute, a exception is thrown, only this method will be rolled back, but other transaction will not be affected.


Source code for this lab is available at: <https://github.com/haminhcong/spring-lab/tree/master/spring-transaction-management>

## References

- <https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html>
- <https://stackoverflow.com/a/13052757>
- <https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/EnableTransactionManagement.html>
- <https://dzone.com/articles/how-does-spring-transactional>
- <https://dzone.com/articles/under-the-boot>
- <http://www.codingpedia.org/jhadesdev/how-does-spring-transactional-really-work/>
- <https://www.logicbig.com/tutorials/spring-framework/spring-data/transactions.html>
- <https://github.com/spring-projects/spring-boot>