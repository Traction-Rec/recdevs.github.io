---
layout: post
title:  "3 Apex Batch Gotchas"
date:   2020-12-04
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

      // ... Batchable & Scheduleable implementation below

The problem is on line 1: `implements Database.Batchable<sObject>, Schedulable`. Implementing both of these interfaces can lead to unexpected side effects. When an instance of a class is scheduled it gets saved & tucked away somewhere in Salesforce's backend. When execution time rolls around, Salesforce dusts off your class & calls the `execute` method. Do you see the problem in `AccountActivationBatch`?

The problem is that the `executeDate` of `AccountActivationBatch` will be the same every time the class is executed. 

For example, if today is Nov 24th 2020 and the class gets scheduled like so:

    System.schedule('Activate Accounts', '0 0 * * * ?', new AccountActivationBatch(Date.today()));

Then it will always execute with the activation date of Nov 24th 2020. What is probably intended here is that the batch always executes with the current date.

If a batch seems simple, perhaps because it is stateless, you may be tempted to 'save time' by making the batch schedulable. In this case you should be wary of how a less-wise developer might modify your batch to add state. It is best to program defensively and not allow your batch to be schedulable.

### The Ungotcha

Make your job easy and [separate concerns](https://en.wikipedia.org/wiki/Separation_of_concerns#:~:text=In%20computer%20science%2C%20separation%20of,code%20of%20a%20computer%20program.). Handle the batch and the scheduled execution of the batch separately. This way, you don't have to worry about stale property values. 

Another benefit of this approach is that you are more free to upgrade the apex code of the batch. There are weird side-effects caused by updating a scheduled class. If the batch were scheduled then you would have to worry about those side effects.

Here is how that might look:

    public class AccountActivationBatchScheduledExecutor implements Schedulable {
      public void execute(SchedulableContext sc) {
        new AccountActivationBatch(Date.today()).run();
      }
    }


    public class AccountActivationBatch implements Database.Batchable<sObject> {

      private Date executeDate;

      public AccountActivationBatch(Date executeDate) {
        this.executeDate = executeDate
      }

      // ... Batchable implementation below

## 2. Ordering

### The Gotcha

From the [developer documentation](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_batch_interface.htm):

> "Batches of records **tend to** execute in the order in which they’re received from the start method. However, the order in which batches of records execute depends on various factors. **The order of execution isn’t guaranteed.**

That is, if order of execution is important then you must not rely on the `ORDER BY` clause in the batch query.

### The Ungotcha

The potential downside of getting the order of execution wrong is huge. Executing business logic in the wrong order can have far-reaching & irreversable consequences. 

The upsides of specifying the `ORDER BY` to get a loose order of execution are usually small. You might get a small performance boost by, say, processing similar records together. However, by allowing this dangerous practice you open yourself up to potentially serious errors.

It is best to not allow batches to specify an `ORDER BY` clause. Making this a general rule forces your development team to be aware of this pitfall & to work around it.

If ordering is required then use a series of batches instead of single batch. Use as many batch jobs as it takes to process the records in the correct order.

## 3. Stale Records

### The Gotcha**s**

The records passed to the ‘execute’ method may:

* no longer fit the criteria
* not include a record that fits the criteria
* may be deleted

When the ‘start’ method returns a query locator the IDs of the records satisfying the locator are copied into temporary storage. The batch then iterates over these stored IDs in chunks. For each chunk it will retrieve the freshest version of the record available and pass it to the execute method.

#### The record may no longer fit the criteria.

This occurs when a record is modified so that it no longer fits the criteria after the temporary storage is populated with the records included in the scope. This also occurs if the record is modified by another process in the moment that the execute method is called.

#### The batch may not include a record that fits the criteria

This occurs when a record is updated to fit the batch criteria after the temporary storage is populated.

#### May be deleted

Exactly the same as ‘no longer fit the criteria’, just with more interesting consequences.

## The Ungotcha

If it is essential that the that records still fit the criteria the batch must requery the records at the start of the execute method and filter out records that no longer fit the query criteria. 

To ensure that the record is not modified during the execution of the batch then lock the record using the ‘for update’ clause. Be conservative with use of the ‘for update’ clause as this can degrade system performance if the locks last a long time.