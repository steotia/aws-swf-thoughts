# aws-swf-thoughts
My thoughts on writing workflows on AWS SWF.

DISCLAIMER: I have never created workflows using SWF. 
## 1. How to build interface between Application and SWF? High level design for folks to connect to SWF
### Assumptions
- There is separation between Application & SWF code. SWF code is the software around creating domain, registering workflow types, activities, deciders, etc. Application software is the general purpose software typically running as interface to users, like a CRM system, CustomerCare dashboard, mobile app, etc
- Various workflows will be defined but we are looking at mechanisms where external actors can participate in respective workflows, typically to start a workflow or provide a user action like approval, checkout, etc
### Approach
- AWS SWF allows interaction via 3 mechanisms: (1) SDK (2) Flow Framework and (3) Service API. I propose using the Flow Framework for both (a) external and (b) internal communication with SWF components. The Flow framework has factory methods to get both internal and external clients. The Application code can use the external client to communicate with the workflow defined in SWF, while the SWF code can use the internal client to perform async communication
- Applications typically will either start a workflow or send a message to it which can in turn trigger decision(s) and activity(ies)
- For messaging, AWS SWF has a notion called [Signal]( https://docs.aws.amazon.com/amazonswf/latest/developerguide/swf-dg-signals.html) to externally inform a workflow execution. Workflow decider code has to implement a [Timer](https://docs.aws.amazon.com/amazonswf/latest/developerguide/swf-dg-timers.html) and have business logic to check if a Signal came before a Timer fired, vice versa and respond accordingly
- For cases, which may need conditional signals (e.g. approve only when some condition is met), generated external client also provides to obtain the last `Status` of the workflow. That in combination with Application state can be used in Application business logic. 
- Since we have crossed a bounded context between Application and SWF, an association between Application and SWF is required (e.g. appID and runID), to communicate with the correct workflow process
- If required, the external client can be wrapped with an HTTP endpoint, enabling other concerns like RBAC control, etc (who can start and signal the workflow, etc)
### Solution
#### AWS Flow provides a library to allow comunication
If we use AWS Flow, we get the following agents which can talk async
0. XActivities and YWorkflow contain the API definition (*Interface*), XActivitiesImpl & YWorkflowImpl are implementing it
1. Activity Worker: pending Activities <-> ActivityWorker <-> XActivitiesImpl <-> done Activities 
2. Workflow Worker: pending Decisions <-> WorkflowWorker <-> YWorkflowImpl <-> (XActivitiesClientImpl) <-> pending Activities
3. Workflow Worker: pending Decisions <-> WorkflowWorker <-> YWorkflowImpl <-> more Decisions
In addition to this, we also get an External Client which can be used to communicate externally with SWF. So, if `YWorkflow` is the Interface then `YWorkflowClientExternalFactoryImpl` is a Factory which can create an External client via `getClient(<ID>)`. Hence, we can see that if our workflow is defined using AWS Flow, we get some boilerplate code already to start talking to it from an Application
### API consumption at App
Let me take a simple case, where CasaOne wishes to upSell subscription. So, someone (Sales) at the App, may trigger a workflow for upsell and someone else (Sales Manager) may want to approve or reject the upSell or adjust the upsell range, add some discount etc. So, following displays 2 steps (1) starting the flow and (2) approving or rejecting one of the stages

Following is what I can expect to be a minimal interface which can encapsulate all the boilerplate and configuration away.
```
// initiating up-selling flow by a manual action from the App
// insert record in App DB for upsell
upSellID = createUpSellRecord()
// create config object
config = SWFConfig.createConfig(UpSellConfig.getDomain())
// create external client to talk to workflow
wf = UpSellConfig.createExternalClient(config)
// start the workflow via the client
wf.upsell(upSellID)
// save the response to the DB for later
SWFUtils.saveWF(upSellID,wf.getWorkflowExecution())
```
```
// lets say upsell needs an approval
upSell = getUpSell(request)
config = SWFConfig.createConfig(UpSellConfig.getDomain())
// retrieve the execution ID and run ID from DB
WorkflowExecution wfe = SWFUtil.getWF(upSellID)
// create the client to talk to the respective workflow
wf = UpSellConfig.createExternalClient(config,wfe)
// app logic to figure if upsell is possible
upSellDecision = figureIfUpSellPossible(upSell)
if upSellDecision.possible
  // send approval signal
  wf.approve(upSellDecision.getData())
else
  // send rejection signal
  wf.reject(upSellDecision.getReason())
```


### Rough Abstrations and Encapsulations
`com.casaone.utils.swf` <= utility classes

| SWFConfig | 
| ------------- |
| createConfig(String) : SWFConfig |
| createSWFClient() : AmazonSimpleWorkflow |

| SWFUtils | 
| ------------- |
| saveWF(String,WorkflowExecution) : Boolean |
| getWF(String) : WorkflowExecution |

`com.casaone.wf.swf.upsell` <= upsell SWF code

| UpSellConfig | 
| ------------- |
| createExternalClient(SWFConfig,WorkflowExecution) : UpSellWorkflowClientExternal |

| UpSellWorkflow (*Interface*) | 
| ------------- |
| upsell() | <= starter code
| approve() | <= signal
| reject() | <= signal


## 2. How to design the 'activities' and 'deciders' in SWF?
### Assumptions
- Activities and Deciders are written using the Flow Framework 
- Activities and Deciders are running as workers on EC2 machines
### Approach
AWS Flow Framework is a useful abstraction over the AWS SDK, taking over much of the grunt work of managing async communication. There is no point is re-writing it. However, the Flow Framework is not trivial to use, more focus could be on how to handle different workflow topologies via using the various overloaded activity methods with various combinations of Promise parameters for different kinds of blocking behavior. So instead of over engineering in the beginning, my recommendation is to start using it and then figure out common functionality which can be further abstracted out. Also, refer to AWS Flow Framework [recipes](https://github.com/aws-samples/aws-flow-ruby-samples) to see different ways of using the framework.
### Solution
#### General engineering flow
1. Design the entire workflow in the form of a correct flowchart, breaking down activities and decisions, understanding where we could benefit from parallelism.
2. First break down Activities into logical groups and create Interfaces for each logical group of activities, properly using the `@Activities`, `ActivityRegistrationOptions` for the `Interface` and other annotations to configure the various policies for the methods on the `Interface` (Backoff, timeouts, etc)
3. Create the WorkFlow Interface to create the entrypoint (`@Execute`), various signal functions (`@Signal`) or status function (`@GetStatus`), where necessary. Not all workflows will need Signalling
4. Implement the Activity Interfaces
5. Workflow communicates to Activities via the Activities Client - generate the Activities Clients (proxy as per the `@Activities` Interfaces).
6. In the Worker Implementation, use the Activities Client to perform activities. Use `Promise`s to perform activities asynchronously. Use `TryCatchFinally` to manage Exceptions raised during `Promise` execution and cleanup resources.
7. Create executables for both Worker and Activity Host. Create JAR via Maven and deploy
#### Activities Packaging
Certain activities are generic, for instance, read/write to storage, sending email/SMS, creating PDF, etc. So instead of packaging with the the workflow, I would encourage the team to package per functionality and import when required.
#### Change management
## Apprehensions
- Workflows are best represented as state machines. AWS SWF is not a true representation of a state machine, thus Deciders can become very complex and start accumulating conditions over a complex workflow
- AWS SWF deciders and activity workers are based on Long Polling which have issues (a) consumes more resources than needed - an EC2 machine is always running (b) there is lag between activities - this lag can accumulate over activities and decisions and can make the overall workflow slow (c) can encounter issues when workers do not gracefully shutdown between polls
- Even if activity can be run as serverles, Decider programs are not natively serverless
- There is no way to visualize the state diagram and the current state, this is a very big minus
- Flow Framework is not very easy to consume, low level APIs are not async in nature
- AWS SWF is supported but no longer being actively developed
### Alternative
- AWS Step Functions instead of using AWS SWF - Haven't deeply investigated but at a high level and reading up, AWS Step Functions seem to be a good alternative in that (a) can be created declaratively (b) behaves like a state machine (c) has better integration with other AWS managed services (d) is native serverless, possibly cheaper to develop and operate
- AWS SWF is a good solution but AWS Step Functions are a higher abstraction and lets people focus more on the workflow than the software to make it work. People can focus on activities and deciders are part of the way you create the decision tree.
