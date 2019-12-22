# aws-swf-thoughts
My thoughts on writing workflows on AWS SWF
## How to build interface between application and SWF?
### Assumptions
- There is separation between Application & SWF code. SWF code is the software around creating domain , registering workflow types, activities, deciders, etc. Application software is the general purpose software typically running as interface to users, like a CRM system, CustomerCare dashboard, mobile app, etc
- Various workflows are defined and runnable but we are looking at mechanisms where external actors can participate in respective workflows, typically to start a workflow or provide a user action like checkout, etc
- This question is more about how to interface with a workflow created using SWF and high level library/interfaces which developers can follow/improve to make it easier to interface with workflows without needing to understand the various complexities around how the workflow is actually implemented
### Approach
- AWS SWF allows interaction via 3 mechanisms: (1) SDK (2) Flow Framework and (3) Service API. I propose using (3) Service API for an App to talk to the Workflow as it allows for a much simpler implementation. Flow Framework maybe used to implement the code on the workflow side (async deciders, activity, etc) but more on that later when we come to how to write deciders and activities in SWF.
- While Starting a WF is simple, AWS SWF has a notion called [Signal]( https://docs.aws.amazon.com/amazonswf/latest/developerguide/swf-dg-signals.html) to externally inform a workflow execution
- Any workflow will have specific defaults, which are relevant for starting (domain, workflow type, tasklist) & signalling (domain) and to keep the code DRY, workflow specific defaults can be packaged separately and can be imported by both the App and SWF code
