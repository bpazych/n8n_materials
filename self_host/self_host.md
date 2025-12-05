#n8n #automation 

Self hosted n8n usually installed in 2 ways:
- **Global installation** — `npm install -g n8n`, configs in `~/.n8n`, custom nodes stored under `~/.n8n/nodes`.
- **Docker / Container** — run via Docker / Docker Compose for easier deployment, portability, and container isolation. [](https://docs.n8n.io/hosting/installation/docker/?utm_source=chatgpt.com)

### Global installation
In case if you choice install n8n globally - on install will be created folder in home directory - `.n8n` where main configs will be saved. Also here you can find `nodes` directory where you can store your custom nodes.

### DB Support

n8n mainly support 2 DB types:
- PostgreSQL
- SQLite

n8n Cloud, which work in n8n service also use this 2 DB, one or another - depends on your plan.

### Pros / Cons 
Pros and cons relatively to the service n8n:
Pros:
- no limitation on active workflows
- possible workarounds for VCS and env management without enterprise subscription (with some hacks)
- full control over infra on you
- easy backups
Cons:
- full control over infra on you
- some limitation in plan selection
- VCS and envs more like workarounds and many limitations