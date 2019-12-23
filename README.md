# aws-swf-thoughts
My thoughts on writing workflows on AWS SWF.

DISCLAIMER: I have never created workflows using SWF. 
## 1. How to build interface between Application and SWF? High level design for folks to connect to SWF
### Assumptions
- There is separation between Application & SWF code. SWF code is the software around creating domain, registering workflow types, activities, deciders, etc. Application software is the general purpose software typically running as interface to users, like a CRM system, CustomerCare dashboard, mobile app, etc
- Various workflows are defined and runnable but we are looking at mechanisms where external actors can participate in respective workflows, typically to start a workflow or provide a user action like checkout, etc
- This question is more about how to interface with a workflow created using SWF and high level library/interfaces which developers can follow/improve to make it easier to interface with workflows without needing to understand the various complexities around how the workflow is actually implemented
- The approach is not tied to a specific language and is a high level organisation of code. Language specific nuances are avoided where possible.
### Approach
- AWS SWF allows interaction via 3 mechanisms: (1) SDK (2) Flow Framework and (3) Service API. I propose using (3) Service API for an App to talk to the Workflow as it allows for a much simpler implementation. Flow Framework maybe used to implement the code on the workflow side (async deciders, activity, etc) but more on that later when we come to how to write deciders and activities in SWF.
- While Starting a WF is simple, AWS SWF has a notion called [Signal]( https://docs.aws.amazon.com/amazonswf/latest/developerguide/swf-dg-signals.html) to externally inform a workflow execution. Workflow decider code has to implement a [Timer](https://docs.aws.amazon.com/amazonswf/latest/developerguide/swf-dg-timers.html) and have business logic to check if a Signal came before a Timer fired, vice versa and respond accordingly
- In cases, where the application is not directly involved, but the user is involved outside the application, for instance, via email, SMS, etc, then AWS SNS can be used to interact with the long running activity waiting for user action
- For cases, which may need conditional signals (e.g. approve only when some condition is met), the application state should be used for enabling or disabling the signalling mechanism. For instance, if the workflow is that after subscription, a user can subscribe for a new package only if he/she has returned or purchased the rented furniture of the old package, then the IF condition: User wants to subscribe && User out of subscription && User has returned or purchased old subscription furniture is application state. This application state should not be in Decider state.
- Any workflow will have specific defaults, which are relevant for starting (domain, workflow type, tasklist) & signalling (domain) and to keep the code DRY, workflow specific defaults can be packaged separately and can be imported by both the App and SWF code. So, in all there are 3 components involved for every workflow: (a) interfaces, base classes, common code, constants, defaults, exceptions, policies, etc (b) workflow gateway code for UI (starter,signal) and (c) workflow implementations - workflow/activity register, starter, decider, activity, etc. Both (b) and (c) pull in (a) as build dependencies. (b) may not be required for all workflows, only those where application code needs to interact with the workflow.
- Since we have crossed a bounded context between Application and SWF, an association between Application and SWF is required (e.g. subscriptionID and runID), to retrieve the workflow run against a specific application object
- If required, the SDK can be interfaced with an HTTP endpoint, enabling other concerns like RBAC control, etc (who can start and signal the workflow, etc)
### Solution
Let me take a simple case, where CasaOne wishes to upSell subscription. So, someone (Sales) at the App, may trigger a workflow for upsell and someone else (Sales Manager) may want to approve or reject the upSell or adjust the upsell range, add some discount etc. So, following displays 2 steps (1) starting the flow and (2) approving or rejecting one of the stages

### Rough Abstrations and Encapsulations
`com.casaone.utils.swf` <= common concerns

| SWFUtil | 
| ------------- |
| getClient() : Client |
| saveWF(GUID, RunResponse): Boolean |

`com.casaone.wf.swf.common` <= interfaces

| WFAppGatewayI | 
| ------------- |
| startWF(Client, ApplicationID) : RunResponse |
| signalWF(Client, ApplicationID, RunID, Signal, Input) : SignalResponse |
| getStatusWF(Client)

`com.casaone.wf.swf.upsell.app` <= implementation for app

| UpSellWFAppGatewayImpl | 
| ------------- |
| startWF(...)|
| signalWF(...)|

- imports `com.casaone.wf.swf.upsell.common.Types` for getting contextual constants

`com.casaone.wf.swf.upsell.common` <= common concerns for upsell workflow

| Types | 
| ------------- |
| Domain, TaskList, Activities|

### API consumption at App
```
// initiating up selling flow by a manual action from the App
// record in DB for upsell
upSellID = createUpSellRecord()
client = SWFUtil.getClient()
UpSellWFAppGatewayImpl upSell = new UpSellWFAppGatewayImpl()
RunResponse r = upSell.startWF(client, upSellID)
SWFUtil.saveWF(upSellID,r)
```
```
// lets say upsell needs an approval
upSell = getUpSell(request)
client = SWFUtil.getClient()
UpSellWFAppGatewayImpl upSell = new UpSellWFAppGatewayImpl()
RunResponse r = SWFUtil.getWF(upSellID)
upSellDecision = figureUpSell(upSell)
if upSellDecision.possible
  upSell.signalWF(client,upSell.ID,r.RunID,'UpSellApproval',upSellDecision.Result)
else
  upSell.signalWF(client,upSell.ID,r.RunID,'UpSellReject',upSellDecision.Result)
```
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
