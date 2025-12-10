#n8n #automation
# n8n helper

## Import workflow from json

n8n allow you backup into json and then create new workflow and import your flow

To import workflow in new workflow use "..." button near save button and select "Import from file"

<img src="./import_workflow.png" style="width:600px;"/>

##### Import workflow which use sub-workflow

At this moment n8n can't automatically find related sub-workflow, so after import all required flows from json - you need in main workflow to sub-wokflow node and manually select target sub-workflow in drop down menu (of course you need first import sub-workflow in same way as main flow)

<img src="Pasted%20image%2020251210141237.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>
