### Backward Compatibility: If the Goverance Checks run when you click “Save” API before going into Prod, what happens with APIs in prod or old APIs when you apply a ruleset to a Category? Will they be updated? Or do you have to go API to API to save them? They will be updated as a background asyc

### Clear criteria between what is compliant vs non-compliant APIs, internally and for users/costumers: 
We have 3 serverities Error/Warn/Info. If there is an Error in a Rule, it will make it non compliant

### Wording and naming: “Templates” we already have API Templates in Dashboard, so we need to either change this template's name or the other one.
This has not being decided. Perhaps [Change “API Templates” to “Assets & Examples”.]

### Observability integration is critical 
governance tools must link to existing monitoring rather than replace it


## Answered Questions
### Governance Pricing: 
Add On, Entitlement like UDG. Probably not fully decided

### Are we depriving the Governance PoC once the integration in the Dashboard is introduced? 
Yes
### Did customers use the PoC? 
Next Era & Bottomline

### Apart from applying rulesets to Categories. Can you create an API, go to the governance section, and from there you are able to apply a ruleset to that API to run the checks while you are configuring it? 
You can test an API against a ruleset from the “Goverannce Page” but you cannot govern it, you do that through the categories tag

### Should we use API Category to map the Ruleset OR should we create something new to Map? While creating a Ruleset, you can add a Category, and this ruleset would run against all APIs in that Category? How do customers use “Category”? Does it make sense that all the APIs within a Category are under the same rulesets? Or should we have another tagging system? Maybe “Governance Tags”. 
We are staying with Categories simpler approach. You can apply multiple categories per ruleset and multiple ruleset per category. If thye were using this tag for somehting else, they can always create new category tags for governance.

### When an API is non-compliant, how does Tyk guide you to make the necessary changes?
Yes, each rule within the ruleset (that has a problem with that API) will give you guidence in the Ruleset Definition on how to fix that specific rule, “view issue info”  from a ruleset will explain the affected areas and how to fix.

### How does Governance function with multi-environment organizations (Staging, Production)? How does it apply to Governance, and how does it relate to the API lifecycle? Is on the CP level, so you have governance per environment.

### In Terms of Logs, we should update the Dashboard Audit Logs to include the new Governance Actions, right? 
Yes 

 

## Validate With Customers
Will the Governance only be introduced for OAS & MCP APIs? Yes, in Release 1, but validate with customers.
Vacuum Templates only run in OAS. (MCP TOO)


### With MCPs around the corner, we must think about how they would work with Spectral / Vacuum rulesets, right? 
Yes, could be straight forward as they rely on JSON. OAS structure. 
Introduce MCP Governance Template & Validate with customers.

### Are we enforcing the new lifecycle flow? 
No

Default new flow: Create API → Save → (Governance Checks in the Background) Live

Additional: Create API → Design State (Draft?) → (Save API) Governance Check → Optional Approval (Warning) → Deploy. 
Only for the APIS that have a ruleset aplied to them
We will have a “Warning” gate. If you turn this on, it will move you to the Additional flow.

### “Errors are not technically feasible to Block Deployment for Non-Compliant APIs”. Why are they not feasible? What do the customers think about it? If they were feasible, would we block? 
If we introduced a bug we are blocking every deployment for every API, this would be Critical, and also Operator …
To validate with customers:  Should we give the customer the power to decide if they want to make a ruleset block or not their linked APIs? Or make specific APIs being able to be blocked?

### Can Vacuum / Spectral detect Redundant APIs (APIs with the same functionality or that return the same data) or Zombie APIs (zero traffic, minimal usage)? 
No, it will have to be something else, perhaps easy to implement.
Validate with customers? probably POST 1st release.
This can be a different flow from rulesets, maybe a new page under gov report with APIs with no traffic for example

 
## Customer Insights
Customers (NorthWestern Mutual & Barclays) would like RBAC to be expanded. A read-only shared link of the dashboard for Federated teams. Something like a snapshot you can share.

Region O’s special interest is in security.

### Further Ideas
Customers (NorthWestern Mutual & Barclays) would like RBAC to be expanded. A read-only shared link of the dashboard for Federated teams. Something like a snapshot you can share.