# Data Flow
#n8n #automation 
### Cross-node data flow

Normal node output - array of objects or array of `OneOuputItem`, on input node take items one - by - one until read all from prev node output. Node **do not** stream items one by one on output. 

Example:

Prev Node `Supabase: GetMany` - return array of entities (objects). Only after node done with handling data - its will send on output ready to use array of objects.

Current Node: `Set (Edit Fields)` - map `name` property from entity and remap on `full_name`. 

Inside Set node expression will look like `"full_name": {{$json.name}}`. $json - represent one entity. 

What happen underhood
- `Set` node run loop over input entities and apply our expression for each entity


### Default node execution

You can find that in editor execution of node can take time. For example simple `Code` node which map or transform small piece of data - run 2-3seconds, which is unusual for real code execution. 
The reason that n8n service first send input data and expression (its code which you write in `Code` node or any other node where you fill or change some fields) to the server, execute your code there and return with result. Its involve http requests where major part of the time take the establishing connection.

General sequence:

```txt
WorkflowEngine -> Node: execute(itemsIn: Item[])
Node -> Node: init outputItems := []

loop for each item in itemsIn
    Node -> ExpressionEvaluator: evaluate(expressions, currentItem)
    ExpressionEvaluator --> Node: evaluatedValues
    Node -> Node: build newItem using evaluatedValues
    Node -> Node: outputItems.add(newItem)
end

Node --> WorkflowEngine: itemsOut := outputItems[]
WorkflowEngine -> NextNode: execute(itemsOut)
```

### Other types of execution

#### One for all
Can be used in `Code` node if selected option. In such case n8n engine will provide Expression Evaluator with all input items at once, you can get all items if use `.all()` method. Some like that - `var items = $input.all()` which will get all input items and assign to items variable.

Advantage over default node execution will be noticeable only on big chunks of input data because its bypass n8n ExpressionEvaluator call overhead, which by default iterate over all items.

#### Nodes that changes item count

Such nodes as `If, Merge, Split, Aggregate, Execute Workflow, Switch` create branches and change output item count - so input data count != output data count. That behavior can cause side effects. 

For example 

We have next workflow which will remap keys in array object.

![](Pasted%20image%2020251204114940.png)

Nodes:
- Generate array (Code node) - generate simple array object like
```js
[
  {"age":12},
  {"age":32},
  {"age":2},
]
```

- map objects key (Set node) - update objects key `age` to `modified_age`
- If age older than 30 (If node) - create branches where in `true` branch going all items with value more that 30 (only one object in our case) and `false` for rest. 
- Post if key map for older (Set node) - one more time reassign keys from `modified_age` to `post_if_modified_age_over_30`
- Post if key map for younger (Set node) reassign keys from `modified_age` to `post_if_modified_age_under_30`

This example show only branching in n8n and don't have any actual business problem to solve.

So what is the flow and options to get value for first `Set` node (map objects key)
- `{{$json.age}}` its default way how you should address input data in 99% of the time.
- `{{ $('Generate array').item.json.age }}` use that in case if for some reason you can't place current node next after data provider node.
 ![](Pasted%20image%2020251204120115.png)
Both of this options work just fine because we don't have any nodes which can create branches. First options will work if current node places next to previous (data provider) node, if between them you have other nodes which dont create branches but transform input data you probably gonna use second approach (but be cautions, you can override some results of previous nodes).

Second option should have some consideration to use but in general its don't cause syntax errors (only can break you business logic).

But if we place in workflow some node which create branches (like If on our case) we no longer can use second approach safely. n8n no longer know what item its should return to you. In that case you have options:
- use first approach - `{{$json.modified_age}}` - which is right choice
- use `.first()` method. This option should go through consideration, because its with big probability cause unwanted side effects (on screen bellow)
![](Pasted%20image%2020251204121618.png)
As we see option with address node directly and use `.item` property no longer works, n8n do not know what item you refers to. Second, correct option, works as expected. And last one - works, but cause some side effect. n8n know that addressed node have array of outputs and we just ask for first one - which is fine, but its going in conflict with our business logic - we want in this branch work only with objects which age property greather than 30 - but first object in our 'map objects key' node is 12.


### Branching nodes summary

Ways to retrieve input data:
- `$json.` - the correct way to get input data 
- `.all()`-  method which return full array of output data from target node. Useful for cases when you need implement some manual data transformation in `Code` node.
- `.first()` - return first item. Use that only if you sure that target node should produce only one item of ouput data. In other cases - avoid it.

Also branch execution change from version to version but 
- before version 1.0 n8n executed both branches in parallel - first node in one and other branch then second node in both branches, and so on
- from version 1.0 n8n execute branches in sequential order, from topmost to bottommost.

### Execute Workflow node

This node stood out from other because this node allow you move part of your logic in separate workflow.
Such sub-workflow you might want to use if:
- reuse logic - if some parts of workflow you implement multiple time - consider move that logic in to Sub-workflow node and replace in your main workflow
- main workflow become too bulky - create modules with sub-workflow. Its eaysier to fix or update small logic unit than in full workflow.


