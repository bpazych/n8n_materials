
# Grafana Loki logging system
#n8n #automation #grafana_loki

Links:
 - https://grafana.com/docs/grafana/latest/administration/service-accounts/

By default n8n don't have integration with `Grafana Loki`, only with grafana dashboard and users.
To send logs to Grafana you need:
- setup account on https://grafana.com/
- get access token 
- setup REST API node in n8n


### Setup account and get access token

Register with your email just by enter on https://grafana.com

#### Get Access Token

Goto https://bohdanpazych.grafana.net/a/grafana-setupguide-app/getting-started/logs-onboarding/select-infrastructure (or from home page, left panel > `Connections`> `Add new connection`> click on `Logs Observability`)


<img src="Pasted%20image%2020251208125547.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>

Now in `Select your infrastructure` select option `Platform as a service` and in `Select how your platform sends logs` -> `Send logs over HTTP`.
<img src="Pasted%20image%2020251208132041.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>

Now in `Configure sending logs via HTTP` you will see 2 main steps. First - lets create our token simply enter name how that token will be stored, create by pressing `Create token` button, and copy and save token somewhere.
<img src="Pasted%20image%2020251208132337.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>

Now we need 3 things to do
1) find our INSTANCE_ID. Todo that goto https://grafana.com/orgs and navigate to `Grafana Cloud` and click on your nickname and then in address bar you will see some like `https://grafana.com/orgs/bohdanpazych/stacks/1462614` where last numbers  `1462614` its your instance id. Alternatively you can get token by click on `Node` in `Step2: Send logs from your application node` there you can find variable `INSTANCE_ID`
 <img src="Pasted%20image%2020251208140256.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>

<img src="Pasted%20image%2020251208140646.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>
2) You need convert concatenation of  INSTANCE_ID : token to base64. In n8n you can create `Code` node and use this code:
	```javascript
	const INSTANCE_ID = "1462614";
	const API_KEY = "<YOUR_API_KEY>";
	const authPair = `${INSTANCE_ID}:${API_KEY}`;
	const encoded = Buffer.from(authPair).toString('base64');
	
	return {"token":encoded};
	```

3) Also we need url for our instance and simplest way to get that - get in on `Step 2: Send logs from your application code` , variable `url`.

##### N8n setup 

Now we have all required data to integrate with Grafana. Now lets create workflow
<img src="Pasted%20image%2020251208185349.png" style="width:750px;margin-left:auto;display:block;margin-right:auto;"/>
`data_to_log` - code node which just generate some data which will be sent to grafana.
`grafana_token` - code node which create base64 token for `Authentication`  header
`send log` - HTTP node which actually send data.

[JSON example](./grafana_integration.json)