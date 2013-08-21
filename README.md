# SObject Work Queue #

Apex Batch is something like the last resort for Apex developers to circumvent limitations of the Salesforce Platform when working with "Large" data volumes.  When using Batch as the asynch backbone of a bigger system you soon find obvious drawbacks:

- Jobs are put in a queue, but when that queue is full (Max. 5 concurrent batches), the job fails instead of being scheduled for later processing.
- Jobs of different Batches might work and conflict on the same data. There is locking mechanism or a guaranteed order.
- Poor support to handle party failed batch runs. Its really hard to find out where and why a single job failed and to restore data consitency.

## Design Goals: ##

We tried to build a custom queue that overcomes those drawbacks.

- Must prevent Max 5 batch in parallel limit - We should never run into this limit with work that is processed over the queue.	 	 	 
- The queue needs to be so generic that "work" on any type of database object needs to be enqueued.	 	 	 
- Any type of modification of database objects need to be possible. This should be transparently handled by the queue.	 	 	 
- Provides better error diagnostics like Batch. Knows last successful Id, full stacktrace and sends email to developers.	 
- Secures data integrity like Batch or better. Failures should not leave data in inconsistent state or user of the infrastructure should be able to handle them.	 	 	 
- Optimistic locking : Instead of locking all many records we do not to process work on Ids that have other work already scheduled.	 	 	 
- Work that can be run synchronously, should not be queued and processed asynch.

> ![SObject Work Queue : How work is definied, enqueued and processed](https://dl.dropboxusercontent.com/u/240888/SObjectWorkQueueInfrastructure.png)

## Architectural Components ##

- `interface SObjectProcessor` : Command interface to define a type of work that can be performed on a Set of Ids

        public interface SObjectProcessor {
          String getFullClassName();
	        void setRecordIds(List<Id> recordIds);
	        void setParameterObject(Object parameterObject);
	        Boolean canRunSynchronously();
	        void process(SObjectWork.LastSuccessfulId lastSuccesfulId);
        }

- `class SObjectWork` : Defines concrete work to do on which concrete Ids

        List<Id> ids = ...;
        Map<String, Object> paramsMap = new Map<String, Object>();
        paramsMap.put('name', value);
        SObjectProcessor processor = new SObjectWorkTestHelper.ExampleSObjectProcessor();
        
        SObjectWork work = new SObjectWork.Builder(oppsToProcess, processor).withParams(paramsMap).build();

- `class SObjectWorkQueue` : singleton class to add work to the queue 

        SObjectWorkQueue.addWork(work);

- `SObject SObjectWork__c` : a Custom SObject that persits SObjectWork instances in the database for later processing
- `class SObjectWorkSerializer` : converts an instance of SObjectWork into 1-n records of SObjectWork__c
- `class SObjectWorkDeserializer`: converts SObjectWork__c records back to their memory representation SObjectWork

## Usage ##

For usage examples please see the test classes especially `SObjectWorkQueue_Test` and `SObjectWorkTestHelper`.



## SObject Work Queue License ##

Copyright (C) 2013 UP2GO International LLC

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
