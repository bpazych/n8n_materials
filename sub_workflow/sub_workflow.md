#n8n #automation 

Sub workflow node - its a way to move out logic out of main workflow.
Sub workflows allow implement some sort of modules which can be reused in many workflows.

### **What Are Sub-Workflows**

- Separate workflows executed from another workflow
- Triggered using **Execute Workflow** node
- Behave like reusable functions / modules 
- Can receive input data and return processed output

Link:
 - https://docs.n8n.io/flow-logic/subworkflows/

---
## Example

Here you can find "dummy" example how to use `Sub-workflow`.

To show that we need 2 workflows:
1) main workflow with our main business logic
2) token-verification sub-workflow, know only how parse JWT token and return payload

We moved token verification in separate workflow - because often you need work with token parse more than one time, and the main implementation of sub workflow doesn't actually changes, same flow - JWT token in -> Payload data out.

Here is main workflow ([link on json file](./main-flow.json))


<img src="Pasted%20image%2020251210140516.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>
Here is sub-workflow - token verification ([link on json file](./token_verification.json))

<img src="Pasted%20image%2020251210140844.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>

> To import this example and play with it you need import `json` files (download by links above) of main flow and sub-workflow ([How to import instruction](../n8n_helper/n8n_helper.md))