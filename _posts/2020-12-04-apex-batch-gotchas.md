---
layout: post
title:  "3 Apex Batch Gotchas"
date:   2021-01-15
categories: Apex
author: john
---

## 1. Scheduling a Batch

### The Gotcha

Your batch looks something like this:

    public class AccountActivationBatch implements Database.Batchable<sObject>, Schedulable {

      private Date executeDate;

      public AccountActivationBatch(Date executeDate) {
        this.executeDate = executeDate
      }

      public void execute(SchedulableContext sc) {
        Database.executeBatch(this);
      }

      // ... Batchable & Scheduleable implementation below

The problem is on line 1: `implements Database.Batchable<sObject>, Schedulable`. Implementing both of these interfaces can lead to tricky bugs. When an instance of a class is scheduled that instance gets saved & tucked away somewhere in Salesforce's backend. When execution time rolls around, Salesforce dusts off your instance & calls the `execute` method. Do you see the tricky bug in `AccountActivationBatch`?

The problem is that the `executeDate` of `AccountActivationBatch` will be the same every time the class is executed as part of its scheduled run. 

For example, if today is Nov 24th, 2020 and the class gets scheduled like so:

    System.schedule('Activate Accounts', '0 0 * * * ?', new AccountActivationBatch(Date.today()));

Then it will always execute with the activation date of Nov 24th, 2020. What is probably intended here for the batch to execute for the current date, not the same date over & over.

If a batch seems simple, perhaps because it is stateless, you may be tempted to 'save time' by having the batch implement the schedulable interface. In this case, you should be wary of how a less-wise developer might make your batch stateful. It is best to program defensively and not allow a batch class to implement the schedulable interface.

### The Ungotcha

Make your job easy and [separate concerns](https://en.wikipedia.org/wiki/Separation_of_concerns#:~:text=In%20computer%20science%2C%20separation%20of,code%20of%20a%20computer%20program.). Handle the batch and the scheduled execution of the batch separately. This way, you don't have to worry about stale property values. 

Here is how that might look:

    public class AccountActivationBatchScheduledExecutor implements Schedulable {
      public void execute(SchedulableContext sc) {
        Database.executeBatch(new AccountActivationBatch(Date.today()));
      }
    }


    public class AccountActivationBatch implements Database.Batchable<sObject> {

      private Date executeDate;

      public AccountActivationBatch(Date executeDate) {
        this.executeDate = executeDate
      }

      // ... Batchable implementation below

Another benefit of this approach is that you are freer to change the apex code of the batch. This is because it is not saved for scheduled execution. Changing a class that is scheduled for execution can lead to unexpected behaviour. This is because the saved version of a class can conflict with the updated version of the class. 

For example, suppose you schedule an instance of a class that has no properties. Then you update the class to contain a single property that is set in the constructor. When the scheduled instance of the class is executed, the constructor will **not** be executed, and so the new property will be null. This will likely lead to a null pointer exception when the property is referenced. This is very tricky to debug!

## 2. Ordering

### The Gotcha

From the [developer documentation](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_batch_interface.htm):

> "Batches of records **tend to** execute in the order in which they’re received from the start method. However, the order in which batches of records execute depends on various factors. **The order of execution isn’t guaranteed.**

That is, if the order of execution is important then you must not rely on the `ORDER BY` clause in the batch query.

### The Ungotcha

The potential downsides of getting the order of execution wrong are huge. Executing business logic in the wrong order can have far-reaching & irreversible consequences. 

The upsides of specifying the `ORDER BY` to get a loose order of execution are usually small. You might get a small performance boost by, say, processing similar records together. However, by allowing this dangerous practice you open yourself up to potentially serious errors.

It is best to not allow batches to specify an `ORDER BY` clause at all. Making this a general rule forces your development team to be aware of this pitfall & to work around it.

If ordering is required then use a series of batches instead of a single batch. Use as many batch jobs as it takes to process the records in the correct order.

## 3. Stale Records

### The Gotcha**s**

The records passed to the ‘execute’ method may:

* no longer fit the criteria or,
* not include a record that fits the criteria.

When the ‘start’ method returns a query locator, the IDs of the records satisfying the locator are copied into temporary storage. The batch then iterates over these stored IDs in chunks. [For each chunk it will retrieve the freshest version of the record available and pass it to the execute method](https://foobarforce.wordpress.com/2015/10/25/batch-apex-query-behaviour/).

#### The record may no longer fit the criteria.

This occurs when a record is modified so that it no longer fits the criteria after the start method executes, but before the record is processed by the batch. This also occurs if the record is modified by another process during execution of the execute method.

#### The batch may not include a record that fits the criteria

This occurs when a record is updated to fit the batch criteria after the start method executes.

### A Quick Example

An org contains 5000 accounts (numbered 1-5000 in the `Batch_Number__c` field) and the following batch class. Unfortunately for this blog post this experiment requires the use of `ORDER BY` in the test batch.

    public class BatchOrderTest implements Database.Batchable<SObject> {
        final String ordering;
        
        public BatchOrderTest(String ordering) {
            this.ordering = ordering;
        }
        
      public Database.QueryLocator start(Database.BatchableContext context) {
        return Database.getQueryLocator('SELECT Id, Batch_Tag__c FROM Account WHERE Batch_Tag__c = null ORDER BY Batch_Number__c ' + ordering);
      }

      public void execute(Database.BatchableContext context, List<Account> scope) {
            for (Account acc : scope) {
                if (acc.Batch_Tag__c == null) {
                    acc.Batch_Tag__c = ordering + ' FIRST';
                } else {
                    acc.Batch_Tag__c = acc.Batch_Tag__c + '; ' + ordering + ' SECOND';
                }
            }
            update scope;
      }

      public void finish(Database.BatchableContext context) {
      }
    }

The batch class is executed in parallel as follows:

    Database.executeBatch(new BatchOrderTest('ASC'));
    Database.executeBatch(new BatchOrderTest('DESC'));

The resulting accounts demonstrate that the execute method can be passed stale records that do not fit the batch criteria.

    SELECT batch_tag__c, count(id) num FROM account GROUP BY batch_tag__c

| Batch_Tag__c              | Num |
|------------------------------ |
| ASC FIRST               | 200 |
| ASC FIRST; DESC SECOND  | 2400 |
| DESC FIRST      | 200 |
| DESC FIRST; ASC SECOND  | 2200 |


The 200 accounts with the 'ASC FIRST' or 'DESC FIRST' show that at some point each batch overwrote the other's changes. The 'ASC FIRST; DESC SECOND' and 'DESC FIRST; ASC SECOND' accounts show that the execute method was passed records that did not fit the batch criteria `WHERE Batch_Tag__c = null`.

## The Ungotcha

The batch must requery the records at the start of the execute method and filter out records that no longer fit the query criteria. 

In the `execute()` method, lock records using the `FOR UPDATE` clause to ensure that the record is not modified. Be conservative with the use of the `FOR UPDATE` clause as this can degrade system performance if the locks last a long time.
