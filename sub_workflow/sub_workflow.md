#n8n #automation 

Sub workflow node - its a way to move out logic out of main workflow.
Sub workflows allow implement some sort of modules which can be reused in many workflows.

### **What Are Sub-Workflows**

- Separate workflows executed from another workflow
- Triggered using **Execute Workflow** node
- Behave like reusable functions / modules 
- Can receive input data and return processed output
    

---

### **Why Use Sub-Workflows**
- **Reuse logic** across multiple workflows
- **Reduce clutter** in complex workflows
- **Isolate responsibilities** (single-purpose mini-flows)
- **Improve maintainability** — easier to update one sub-flow
- **Standardize behavior** (validation, formatting, notifications, etc.)
    

---

### **Typical Use Cases**
- **Shared transformations**
    - format dates, clean data, normalize structures
- **Shared integrations**
    - API calls reused across workflows
    - auth/refresh token handling
- **Reusable business rules**
    - scoring, filtering, validation logic
- **Error processing modules**
    - send alerts, write logs, notify Slack
- **Heavy logic offloading**
    - long chains of operations separated from main flow
        

---

### **Benefits**
- Cleaner parent workflows
- Reduced duplication
- Faster debugging (each sub-flow is independent)
- Easier scaling—modify one workflow, impact many
- Better versioning and testing