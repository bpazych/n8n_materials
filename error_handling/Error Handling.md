# Error Handling
#n8n #automation #error_handling

n8n provide quite good option for error handling.
First of all - all nodes support `Settings->On Error` option, which allow you handle any node which can possibly break or throw error.
To use that option, goto `Settings` tab on edit node, and search for `On Error` setting, here select option `Continue (using error output)`

<img src="Pasted%20image%2020251204180939.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>

After that in workflow you will see that node get 2 outputs instead one (by default only success branch), one for default flow when all good, and second `Error` branch, which will be called if node catch some error on items processing.

<img src="Pasted%20image%2020251204181337.png" style="width:400px;margin-left:auto;display:block;margin-right:auto;"/>

#### Error processing

Above we learned how to catch the error, but that is only half of the **error handling**, now we need process that error somehow, return back to user or log that error for future profiling. 

##### Option1: for Webhook node

Webhook node behave as http endpoint, so we should return at least status back to user. To return error code and message - just connect `Respond to Webhook` node to `Error` branch, select what do you want send back (all failed items or custom message) and from `Options` select `Response Code` and set appropriate HTTP Code (like - 401 - auth error, 500 - server exception, etc). Done!

<img src="Pasted%20image%2020251204183946.png" style="width:600px;margin-left:auto;display:block;margin-right:auto;"/>
<img src="Pasted%20image%2020251204184058.png" style="width:400px;margin-left:auto;display:block;margin-right:auto;"/>

##### Option2: Logger services, Slack, etc

In case if your workflow do not serve as Web API - you can send error messages to such tools as `Supabase, Graylog, Grafana Loki, etc` or send into favorite messenger, like `Slack, Telegram, etc.`.

> [Slack Integration](../integrations/slack_api/n8n_slack_api_integration.md)</br>
> [Supabase Integration](tech/tools/n8n/integrations/supabase_api/supabase_api.md)