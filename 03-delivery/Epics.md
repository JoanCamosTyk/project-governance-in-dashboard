# Delivery EPICS

## 1. Governance Ruleset Management (The Configuration)
Governance in Dashboard Ruleset Management (The Configuration)


### Goal
Establish the foundational backend & frontend capabilities to store, manage, and execute governance rulesets using the Vacuum engine within automated workflows & Tyk Dashboard.

### Description
As a platform engineer, I need to create and customize governance rulesets to ensure our APIs meet organizational standards and industry best practices.

This epic will provide comprehensive capabilities for importing, creating and editing Spectral-compatible rulesets that define our API governance policies. This enables the enforcement of consistent API design, security, and compliance standards across our organization.

The system will support both pre-built industry-standard rulesets (Templates) and custom rulesets, allowing governance teams to tailor policies to their specific requirements.

This epic covers the "First Part" of the governance flow. It focuses on the Dashboard APIs and UI required to define what governance means for the organization (creating rulesets, using templates, and configuring severities).


### Features
Ruleset Creation: Support creating from scratch, importing JSON/YAML files, and using templates.

Raw Definition Editor: A built-in code editor to view and modify the raw JSON/YAML definition of a ruleset.

Ruleset Configuration: Define Ruleset Name, Description, Status, Category, and Deployment Warning

Supported Specifications: OpenAPI Specification (OAS) and Model Context Protocols (MCPs)

Ruleset Templates: Provide real, out-of-the-box templates for OAS & MCP / Can also be customer configured templates

Deployment Warning: Per-ruleset toggle, OFF by default. When OFF, non-compliant APIs deploy without any popup or gate — they just show as "non-compliant" in the dashboard. When ON, saving a non-compliant API shows a warning with "Deploy Anyway" / "Go Back". v1 never hard-blocks deployments.

### Questions
Is the Ruleset Repository (list) a Dashboard UI feature, or can you get this from CI/CD pipelines?. 

### Stories

#### Operator integration - TykRuleset CRD

#### TT- 17019 API for Create Rulesets
Context and preconditions
Backend API to Create rulesets


Product idea
Proposed solution from a product perspective, i.e. wireframes, Figma prototypes and etc.
Should be filled by the reporter.


What we will do
In summary, we will follow the pattern: Model + Adapter + Service (Create) + Repository (Add) + Handler (POST /) + Route mount. In order to do so, we will do:


Adapter —> dashboard/adapter/ruleset/adapter.go (new file)
Source: copy-paste governance/internal/storage/adapter/adapter.go.

Mechanical changes only:

Update import paths

CreatedAt / UpdatedAt  converts into → DateCreated / LastUpdated

Metadata.ID string ↔ ID.Hex() / ObjectIdHex()

New fields (ResourceType, Action, Categories, Owner, RulesetID) pass through without transformation

The adapter handles YAML→JSON conversion and rule extraction from the Spectral definition format. 

Service —> dashboard/service/ruleset/service.go (new file)
Source: copy-paste governance/internal/service/ruleset/service.go — Create method only. Import paths only.

Repository —> dashboard/repository/ruleset/ruleset.go (new file)
Source: copy business logic from governance/internal/repository/ruleset/ruleset.go Add (lines 118–136).



type Repository interface {
    Add(ctx context.Context, ruleset *model.Ruleset) error
}
 

Storage call mapping (governance → tyk-analytics):

Governance

tyk-analytics

r.db.Query(ctx, &model.Ruleset{}, &result, filter) 

r.storage.Query(filter, &result) 

r.db.Insert(ctx, ruleset) 

r.storage.Insert(ruleset) 

Add logic:

Duplicate name check scoped to org_id + deleted_at: nil — return ErrNameExists if found

Insert the ruleset

Indexes —> call from the repository constructor using storage.CreateIndex:



func (r *rulesetRepository) ensureIndexes(ctx context.Context) {
    indexes := []model.Index{
        {Name: "name",          Keys: []model.DBM{{"name": 1}}},
        {Name: "org_id",        Keys: []model.DBM{{"org_id": 1}}},
        {Name: "org_id_name",   Keys: []model.DBM{{"org_id": 1}, {"name": 1}}},
        {Name: "ruleset_id",    Keys: []model.DBM{{"ruleset_id": 1}}},
        {Name: "resource_type", Keys: []model.DBM{{"resource_type": 1}}},
        {Name: "categories",    Keys: []model.DBM{{"categories": 1}}},
    }
    row := &model.Ruleset{}
    for _, idx := range indexes {
        if err := r.storage.CreateIndex(ctx, row, idx); err != nil {
            log.WithError(err).Errorf("failed to create index %s on tyk_rulesets", idx.Name)
        }
    }
}
Call r.ensureIndexes(context.Background()) at the end of NewRepository.

Mock generation:



//go:generate mockgen -destination=./mock/ruleset.go -package mock . Repository
Handler —> dashboard/handlers/ruleset/ruleset.go (new file)
Source: copy-paste Add method from governance/internal/handlers/rulesets/rulesets.go (lines 146–182) + multipart helpers from utils.go.

Mechanical changes: responsewriter → handlers.HandleError, zap → logrus, response shape → model.APIResponse.

Create flow:

Detect Content-Type —> parse JSON body or multipart form (metadata + ruleset parts)

Validate name is non-empty —> status 400 if missing

Set server-side fields:



newRuleset.RulesetID    = uuid.NewHex()
newRuleset.Owner        = request.Session(r).EmailAddress
newRuleset.OrgID        = request.Session(r).OrgID
newRuleset.DateCreated  = time.Now()
newRuleset.LastUpdated  = time.Now()
Default and validate Action:



if newRuleset.Action == "" {
    newRuleset.Action = RulesetActionNone
}
if newRuleset.Action != RulesetActionNone && newRuleset.Action != RulesetActionWarn {
    handlers.HandleError("invalid action", http.StatusBadRequest, w, h.instrument)
    return
}
Default and validate ResourceType:



if newRuleset.ResourceType == "" {
    newRuleset.ResourceType = RulesetResourceTypeOASAPI
}
if newRuleset.ResourceType != RulesetResourceTypeOASAPI {
    handlers.HandleError("invalid resource type", http.StatusBadRequest, w, h.instrument)
    return
}
Call svc.Create(ctx, &newRuleset) — ErrNameExists → 409, other errors → 500

Set audit diff:



putAuditDiffToHTTPReqCtx(nil, &newRuleset, r)
Return 201 + model.APIResponse{Status: "success", Meta: newRuleset.ID.Hex()}

putAuditDiffToHTTPReqCtx — new helper, same logic as existing putAuditDiffToReqCtx in audit.go but accepts *http.Request instead of *web.Request.

 

Mount (POST only for this ticket, extended in next tickets):



func (h *Handler) Mount(r *mux.Router, addActionMw func(http.HandlerFunc, string) http.HandlerFunc) {
    r.HandleFunc("", addActionMw(h.Create, "Create Ruleset")).Methods(http.MethodPost)
}
Routes —> dashboard/api_ruleset.go (new file)


func setupRulesetsRouter(r *mux.Router) {
    sub := r.PathPrefix("/api/rulesets").Subrouter()
    sub.StrictSlash(true)
    sub.Use(AuthMiddleware, CheckCSRFMiddleware, InstrumentationMiddleware,
        AddSecureHeadersMiddleware, CacheControlMiddleware)
    rulesetHandler.New(rulesetrepo.NewRepository(MainStorage), log, instrument).Mount(sub, addAPIAction)
}
Call setupRulesetsRouter(HTTPMuxer) inside GenerateRoutes() in dashboard/server.go.

Swagger — swagger.yml
Add under paths::


Swagger spec

 

Acceptance criteria
[ ] POST /api/rulesets with valid JSON body returns 201

[ ] POST /api/rulesets with multipart form (metadata + ruleset parts) returns 201

[ ] Returns 409 if a ruleset with the same name already exists within the org

[ ] Returns 400 if name is missing

[ ] Returns 400 for unknown action or resource_type values

[ ] action defaults to none, resource_type defaults to oas-api when not provided

[ ] categories stored as-is — optional, no validation on values

[ ] RulesetID auto-generated as UUID on creation

[ ] OrgID and Owner always set server-side from session; cannot be set by caller

[ ] DateCreated and LastUpdated set server-side

[ ] Returns 401 for unauthenticated requests

[ ] Returns 403 for users with read or deny on GovernanceGrp

[ ] Users with write or admin on GovernanceGrp can create

[ ] MongoDB indexes created on repository startup

[ ] Audit record with action "Create Ruleset" and diff nil → newRuleset written on success

[ ] Failed creates do not write an audit diff

[ ] swagger.yml updated with POST /api/rulesets path

[ ] All API tests pass

[ ] All unit tests pass

Testing
API tests — tests/api/tests/dashboard_api/rulesets_test.py (new file)
Add tests to the api suite, so we can test e2e the endpoint. A starter point would be as next:

 


API test for Create ruleset
Unit tests
Handler —> dashboard/handlers/ruleset/ruleset_test.go (new file)


func getRulesetHandlerAndMockRepo(t *testing.T) (*Handler, *repomock.MockRepository) {
    t.Helper()
    ctrl := gomock.NewController(t)
    repo := repomock.NewMockRepository(ctrl)
    return New(repo, logrus.New(), &health.Stream{}), repo
}
TestRuleset_Create:

Sub-test

Setup

Expected

invalid JSON body 

malformed JSON 

400 

missing name 

no name field 

400 

invalid action 

action: "block" 

400 

invalid resource type 

resource_type: "grpc-api" 

400 

duplicate name 

repo.Add returns ErrNameExists 

409 

repo error 

repo.Add returns generic error 

500 

success - JSON 

valid body; verify OrgID, Owner, RulesetID, DateCreated set server-side; defaults applied 

201 

success - with categories 

body includes categories 

201 

success - multipart 

multipart form 

201 

audit diff on success 

verify ctxDiff set in request context 

— 

Repository —> dashboard/repository/ruleset/ruleset_mock_test.go (new file)
TestRuleset_MockAdd:

Sub-test

Storage mock

Expected

success 

Query returns empty, Insert returns nil 

no error; DateCreated, LastUpdated, RulesetID set 

duplicate name 

Query returns existing ruleset 

ErrNameExists 

db error on duplicate check 

Query returns error 

error propagated 

db error on insert 

Query returns empty, Insert returns error 

error propagated 



#### TT-17020 API for List and get single Rulesets
Context and preconditions
Context preceding the task and a problem to solve: any researches, feedback from customers, etc.
Should be filled by the reporter.


Product idea
Here we will provide the backend apis to read rulesets, it can be a single ruleset by ID or a list of rulesets given a filter (default filter: same org ID as session)


What we will do
Add GET /api/rulesets/{id} (single) and GET /api/rulesets (list with filters). Extends the repository, service, and handler created in Create ruleset. No new files; only additions to existing ones.

 

Service —> add to dashboard/service/ruleset/service.go
Source: copy-paste Get and List from governance/internal/service/ruleset/service.go. Import paths only.

Repository —> extend dashboard/repository/ruleset/ruleset.go
Add to interface:



type Repository interface {
    Add(ctx context.Context, ruleset *model.Ruleset) error
    GetByID(ctx context.Context, orgID, id string) (*model.Ruleset, error)
    List(ctx context.Context, orgID string, req ListRequest) (*ListResponse, error)
}
type ListRequest struct {
    Search map[string]interface{}
    Sort   string // e.g. "-last_updated"; empty defaults to "-last_updated"
}
type ListResponse struct {
    Rulesets []model.Ruleset
}
GetByID  two-stage lookup: always tries ruleset_id first, then falls back to _id. This mirrors the Policy findUniquePolicy + findPolicyFallback pattern and is necessary because ruleset_id can be any string — including a valid ObjectId hex (the default case where ruleset_id = ID.Hex()). A simple IsObjectIdHex branch would silently miss documents whose custom ruleset_id happens to be 24 hex chars.



func (r *rulesetRepository) GetByID(_ context.Context, orgID, id string) (*model.Ruleset, error) {
    var ruleset model.Ruleset
    // Stage 1: always try ruleset_id first (covers custom and default values)
    filter := model.DBM{"ruleset_id": id, "org_id": orgID, "deleted_at": model.DBM{"$eq": nil}}
    if err := r.storage.Query(filter, &ruleset); err == nil {
        return &ruleset, nil
    }
    // Stage 2: fall back to _id if id is a valid ObjectId hex
    if model.IsObjectIdHex(id) {
        filter = model.DBM{"_id": model.ObjectIdHex(id), "org_id": orgID, "deleted_at": model.DBM{"$eq": nil}}
        if err := r.storage.Query(filter, &ruleset); err == nil {
            return &ruleset, nil
        }
    }
  return nil, ErrNotFound
}
 

Why stage 1 first: ruleset_id is unique per org (enforced by the org_id_ruleset_id index), so the first query is unambiguous. Trying ruleset_id before _id means a custom ruleset_id that looks like an ObjectId hex is always found correctly without needing an $or query.

List  all matching, handler paginates:



```go```
var allowedSortFields = map[string]string{
    "q":         "name",
    "date_created": "date_created",
    "last_updated":  "last_updated",
}
func parseSortSpec(sort string) model.DBM {
    if sort == "" {
        sort = "-last_updated"
    }
    dir := 1
    field := sort
    if strings.HasPrefix(sort, "-") {
        dir = -1
        field = sort[1:]
    }
    if mapped, ok := allowedSortFields[field]; ok {
        return model.DBM{mapped: dir}
    }
    return model.DBM{"last_updated": -1} // fallback
}
func (r *rulesetRepository) List(_ context.Context, orgID string, req ListRequest) (*ListResponse, error) {
    var rulesets []model.Ruleset
    filter := model.DBM{
        "org_id":     orgID,
        "deleted_at": model.DBM{"$eq": nil},
    }
    for k, v := range req.Search {
        filter[k] = v
    }
    sortSpec := parseSortSpec(req.Sort)
    // Verify the storage API supports sort options; use QueryWithOptions or equivalent if available.
    if err := r.storage.QueryWithOptions(filter, &rulesets, model.QueryOptions{Sort: sortSpec}); err != nil {
        return nil, err
    }
    return &ListResponse{Rulesets: rulesets}, nil
}
Handler —> extend dashboard/handlers/ruleset/ruleset.go
Source: copy-paste Get and List from governance/internal/handlers/rulesets/rulesets.go + parseFilters / loadDefaultListFilters helpers from utils.go.

Get flow:

Extract id from mux.Vars(r) —> 400 if empty. Accepts MongoDB ObjectId hex (→ queries _id) or ruleset_id string (→ queries ruleset_id) —> same dual-ID pattern as Update and Delete.

Call svc.Get(ctx, orgID, id) —> ErrNotFound → 404

Return 200 + ruleset JSON

No audit diff as this is a read operation.

List flow:

Build filter map from supported query params (see table below)

Call svc.List(ctx, orgID, ListRequest{Search: filters})

Paginate: handlers.PageDataFromReq(r, len(rulesets), config.Global().PageSize)

Return 200 + {"rulesets": rulesets[start:end], "pages": pages}

Supported query params for List:

Param

Type

Behaviour

q 

string 

case-insensitive regex match against

active 

bool 

exact match 

is_template 

bool 

exact match 

categories 

string 

rulesets that contain this category value 

resource_type 

string 

exact match 

sort

string

sort field; prefix - for descending (e.g. -last_updated). Sortable: name, date_created, last_updated. Default: -last_updated. 

Update Mount:



func (h *Handler) Mount(r *mux.Router, addActionMw func(http.HandlerFunc, string) http.HandlerFunc) {
    r.HandleFunc("",      addActionMw(h.Create, "Create Ruleset")).Methods(http.MethodPost)
    r.HandleFunc("",      addActionMw(h.List,   "List Rulesets")).Methods(http.MethodGet)
    r.HandleFunc("/{id}", addActionMw(h.Get,    "Get Ruleset")).Methods(http.MethodGet)
}
Swagger —> swagger.yml
Add get operation under the existing /api/rulesets path block:


Swagger spec for get operation
 

Acceptance criteria
[ ] GET /api/rulesets/{id} returns 200 for a valid ID within the org

[ ] Accepts ruleset_id string in path — always tried first (stage 1); works for both default and custom values, including hex-format custom IDs

[ ] Accepts MongoDB ObjectId hex in path — tried as _id fallback (stage 2) when stage 1 finds nothing

[ ] A custom ruleset_id that looks like a valid ObjectId hex is found correctly via stage 1 (never misrouted to _id)

[ ] Returns 404 if not found or belongs to a different org

[ ] Soft-deleted rulesets are never returned by Get or List

[ ] GET /api/rulesets returns paginated list

[ ] List supports ?name= (regex), ?active=, ?is_template=, ?categories=, ?resource_type= filters

[ ] List supports ?sort=<field> with - prefix for descending; sortable fields: name, date_created, last_updated

[ ] Default sort is -last_updated when sort param is absent

[ ] Unknown sort field falls back to -last_updated (no 400)

[ ] No audit diff written for read operations

[ ] Returns 401 unauthenticated, 403 for deny on GovernanceGrp

[ ] swagger.yml updated with GET list and GET single paths + RulesetList schema

[ ] All API tests pass

[ ] All unit tests pass

Testing
API tests —> add to tests/api/tests/dashboard_api/rulesets_test.py

API test for get rulesets
Handler —> add to dashboard/handlers/ruleset/ruleset_test.go
TestRuleset_Get:

Sub-test

Setup

Expected

not found 

svc.Get returns ErrNotFound 

404 

repo error 

svc.Get returns generic error 

500 

success by object id 

ObjectId hex in path; stage-1 misses, stage-2 _id matches 

200 

success by custom ruleset_id 

non-hex string in path; stage-1 ruleset_id matches 

200 

success by custom ruleset_id (hex-looking) 

hex-format custom ruleset_id in path; stage-1 ruleset_id matches before stage-2 is tried 

200 

TestRuleset_List:

Sub-test

Setup

Expected

repo error 

svc.List returns error 

500 

success - no filters 

2 rulesets returned 

200 

success - name filter 

?q=foo passed through to repo 

200 

success - category filter 

?categories=payments passed through 

200 

success - pagination 

15 rulesets, ?p=2 returns correct slice 

200 

success - sort asc 

?sort=name → ListRequest.Sort == "name" passed to repo 

200 

success - sort desc 

?sort=-last_updated → ListRequest.Sort == "-last_updated" 

200 

success - default sort 

no sort param → ListRequest.Sort == "" (repo defaults to -last_updated) 

200 

Repository —> add to dashboard/repository/ruleset/ruleset_mock_test.go

TestRuleset_MockGetByID:

Sub-test

Storage mock

Expected

success by custom ruleset_id (non-hex) 

stage-1 Query on ruleset_id returns ruleset 

ruleset returned; stage 2 never reached 

success by custom ruleset_id (hex-looking) 

stage-1 Query on ruleset_id returns ruleset 

ruleset returned; stage-2 _id branch never reached 

success by object id (_id) 

stage-1 Query on ruleset_id returns not found; stage-2 Query on _id returns ruleset 

ruleset returned via _id fallback 

success by default ruleset_id 

id equals _id.Hex(); stage-1 Query on ruleset_id returns ruleset (since ruleset_id == _id.Hex() and index matches) 

ruleset returned via stage 1 

not found 

both stage-1 and stage-2 queries return error 

ErrNotFound 

not found - hex id, no _id match 

stage-1 returns error; stage-2 _id query also returns error 

ErrNotFound 

not found 

Query returns error 

ErrNotFound 

TestRuleset_MockList:

Sub-test

Storage mock

Expected

success 

Query returns 2 rulesets 

ListResponse with both 

category filter 

verify categories key in filter 

passed through correctly 

sort desc 

Sort: "-last_updated" → storage called with {"last_updated": -1} 

correct sort spec 

sort asc 

Sort: "name" → storage called with {"name": 1} 

correct sort spec 

unknown sort field 

Sort: "-invalid" → falls back to {"last_updated": -1} 

fallback applied 

db error 

Query returns error 


#### TT-17021 API for Update rulesets
Context and preconditions
Context preceding the task and a problem to solve: any researches, feedback from customers, etc.
Should be filled by the reporter.


Product idea
API endpoint to update one single ruleset


What we will do
Summary: Add PUT /api/rulesets/{id}. Extends repository, service, and handler. Includes name uniqueness check on rename, audit diff (before → after), and RulesetID immutability guarantee.

Service —> add to dashboard/service/ruleset/service.go
Source: copy-paste Update from governance/internal/service/ruleset/service.go (lines 109–142). Import paths only.

Repository —> extend dashboard/repository/ruleset/ruleset.go
Add to interface:



type Repository interface {
    Add(ctx context.Context, ruleset *model.Ruleset) error
    GetByID(ctx context.Context, orgID, id string) (*model.Ruleset, error)
    List(ctx context.Context, orgID string, req ListRequest) (*ListResponse, error)
    Update(ctx context.Context, orgID string, ruleset *model.Ruleset) error
}
Update —> source: copy logic from governance/internal/repository/ruleset/ruleset.go (lines 176–223), adapt storage calls.

Key logic:

Fetch existing via GetByID and return ErrNotFound if missing. Records deleted cannot be updated

Name uniqueness check only when name changed, excluding self:



filter = model.DBM{"name": ruleset.Name, "org_id": orgID, "_id": model.DBM{"$ne": ruleset.ID}, "deleted_at": model.DBM{"$eq": nil}}
Set LastUpdated = time.Now()

If incoming DateCreated is zero, preserve existing value

Always preserve existing DeletedAt (prevents accidental un-deletion).

RulesetID is always taken from the existing document —> never overwritten from the request

r.storage.Update(model.DBM{"_id": ruleset.ID, "org_id": orgID}, ruleset)

Handler —> extend dashboard/handlers/ruleset/ruleset.go
Source: copy-paste Update from governance/internal/handlers/rulesets/rulesets.go (lines 241–278). Mechanical changes: response writer, logger.

Update flow:

Parse body —> JSON or multipart

Extract id from mux.Vars(r) and  return 400 if empty

Validate name is non-empty and return 400 if missing

Fetch existing via svc.Get(ctx, orgID, id) return 404 if not found (also used for audit diff)

Set rulesetRequest.ID from path id

Call svc.Update(ctx, orgID, &rulesetRequest)

Error mapping: ErrNotFound → 404, ErrNameExists → 409, other → 500

Set audit diff:



putAuditDiffToHTTPReqCtx(existingRuleset, &updatedRuleset, r)
Return 200 + updated ruleset JSON

Update Mount:



func (h *Handler) Mount(r *mux.Router, addActionMw func(http.HandlerFunc, string) http.HandlerFunc) {
    r.HandleFunc("",      addActionMw(h.Create, "Create Ruleset")).Methods(http.MethodPost)
    r.HandleFunc("",      addActionMw(h.List,   "List Rulesets")).Methods(http.MethodGet)
    r.HandleFunc("/{id}", addActionMw(h.Get,    "Get Ruleset")).Methods(http.MethodGet)
    r.HandleFunc("/{id}", addActionMw(h.Update, "Update Ruleset")).Methods(http.MethodPut)
}
Swagger —> add put under /api/rulesets/{rulesetId}

Swagger defintion for the update operation

Acceptance criteria
[ ] PUT /api/rulesets/{id} with valid JSON body returns 200

[ ] PUT /api/rulesets/{id} with multipart form also works

[ ] Returns 404 if not found or belongs to a different org

[ ] Returns 409 if new name conflicts with another ruleset in the org

[ ] Returns 400 if name is missing or empty

[ ] DateCreated is never overwritten on update

[ ] RulesetID is never changed on update

[ ] OrgID always taken from session

[ ] Soft-deleted rulesets cannot be updated

[ ] Returns 401 unauthenticated, 403 for insufficient permissions

[ ] Audit record with action "Update Ruleset" and diff (before → after) written on success

[ ] Failed updates do not write an audit diff

[ ] swagger.yml updated with PUT path

[ ] All API tests pass

[ ] All unit tests pass

Testing
API tests —> add to tests/api/tests/dashboard_api/rulesets_test.py

e2e test for Update API
Unit tests
Handler —> add to dashboard/handlers/ruleset/ruleset_test.go
TestRuleset_Update:

Sub-test

Setup

Expected

invalid JSON body 

malformed JSON 

400 

missing id in path 

empty path var 

400 

missing name 

name: "" 

400 

ruleset not found 

svc.Get returns ErrNotFound 

404 

duplicate name 

svc.Update returns ErrNameExists 

409 

repo error 

svc.Update returns generic error 

500 

success - JSON 

valid body; verify OrgID from session, RulesetID unchanged, audit diff set 

200 

success - multipart 

multipart form 

200 

Repository — add to dashboard/repository/ruleset/ruleset_mock_test.go
TestRuleset_MockUpdate:

Sub-test

Storage mock

Expected

success 

GetByID returns existing, Update returns nil 

no error; LastUpdated set 

not found 

GetByID returns error 

ErrNotFound 

duplicate name 

name changed, uniqueness query returns existing 

ErrNameExists 

db error on update 

GetByID ok, Update returns error 

error propagated 

preserves DateCreated 

incoming has zero DateCreated 

existing DateCreated kept 

preserves DeletedAt 

existing has non-nil DeletedAt 

DeletedAt not overwritten 

RulesetID unchanged 

incoming has different RulesetID 

original RulesetID preserved 

#### TT-17022 API for Delete Rulesets
Context and preconditions
Context preceding the task and a problem to solve: any researches, feedback from customers, etc.
Should be filled by the reporter.


Product idea
Create backend api to delete rulesets


What we will do
Add DELETE /api/rulesets/{id}. Soft-delete only —> sets DeletedAt, document is retained. Includes audit diff with full snapshot of the deleted ruleset.

Service —> add to dashboard/service/ruleset/service.go
Source: copy-paste Delete from governance/internal/service/ruleset/service.go (lines 54–69). Import paths only.

Repository —> extend dashboard/repository/ruleset/ruleset.go
Add to interface:



type Repository interface {
    Add(ctx context.Context, ruleset *model.Ruleset) error
    GetByID(ctx context.Context, orgID, id string) (*model.Ruleset, error)
    List(ctx context.Context, orgID string, req ListRequest) (*ListResponse, error)
    Update(ctx context.Context, orgID string, ruleset *model.Ruleset) error
    Delete(ctx context.Context, orgID, id string) error
}
Delete —> soft delete only:



func (r *rulesetRepository) Delete(_ context.Context, orgID, id string) error {
    existing, err := r.GetByID(context.Background(), orgID, id)
    if err != nil {
        return err // ErrNotFound already wrapped by GetByID
    }
    now := time.Now()
    existing.DeletedAt = &now
    return r.storage.Update(model.DBM{"_id": existing.ID, "org_id": orgID}, existing)
}
Handler —> extend dashboard/handlers/ruleset/ruleset.go
Source: copy-paste Delete from governance/internal/handlers/rulesets/rulesets.go (lines 220–239). Mechanical changes: response writer, logger.

Delete flow:

Extract id from mux.Vars(r) and return status 400 if empty

Fetch existing via svc.Get(ctx, orgID, id) and return status 404 if not found (also used for audit diff)

Call svc.Delete(ctx, orgID, id)

Error mapping: ErrNotFound → 404, other → 500

Set audit diff:



putAuditDiffToHTTPReqCtx(existingRuleset, nil, r)
Return 200 + model.APIResponse{Status: "success", Message: "ruleset deleted"}

Update Mount:



func (h *Handler) Mount(r *mux.Router, addActionMw func(http.HandlerFunc, string) http.HandlerFunc) {
    r.HandleFunc("",      addActionMw(h.Create, "Create Ruleset")).Methods(http.MethodPost)
    r.HandleFunc("",      addActionMw(h.List,   "List Rulesets")).Methods(http.MethodGet)
    r.HandleFunc("/{id}", addActionMw(h.Get,    "Get Ruleset")).Methods(http.MethodGet)
    r.HandleFunc("/{id}", addActionMw(h.Update, "Update Ruleset")).Methods(http.MethodPut)
    r.HandleFunc("/{id}", addActionMw(h.Delete, "Delete Ruleset")).Methods(http.MethodDelete)
}
Swagger —> add delete under /api/rulesets/{rulesetId}

Swagger definition for the Delete api
Acceptance criteria
[ ] DELETE /api/rulesets/{id} returns 200 for a valid ruleset within the org

[ ] Accepts both MongoDB ObjectId hex and RulesetID string in path

[ ] Returns 404 if not found or belongs to a different org

[ ] Document is soft-deleted —> deleted_at is set, document is retained in MongoDB

[ ] Soft-deleted ruleset no longer appears in GET /{id} or GET /

[ ] Returns 401 unauthenticated, 403 for insufficient permissions

[ ] Audit record with action "Delete Ruleset" and full snapshot (existingRuleset → nil) written on success

[ ] Failed deletes do not write an audit diff

[ ] swagger.yml updated with DELETE path

[ ] All API tests pass

[ ] All unit tests pass

Testing
API tests —> add to tests/api/tests/dashboard_api/rulesets_test.py

e2e test for the delete ruleset api
Unit tests
Handler —> add to dashboard/handlers/ruleset/ruleset_test.go
TestRuleset_Delete:

Sub-test

Setup

Expected

missing id in path 

empty path var 

400 

ruleset not found 

svc.Get returns ErrNotFound 

404 

repo error on delete 

svc.Get ok, svc.Delete returns error 

500 

success 

both calls ok; verify audit diff existingRuleset → nil in context 

200 

Repository —> add to dashboard/repository/ruleset/ruleset_mock_test.go
TestRuleset_MockDelete:

Sub-test

Storage mock

Expected

success 

Query returns existing, Update returns nil 

no error; DeletedAt set and non-zero 

not found 

Query returns error in GetByID 

ErrNotFound 

db error on soft delete 

Query returns existing, Update returns error 

error propagated 

#### Ruleset Model & RBAC Setup
Create the shared Ruleset data model, Swagger schemas, RBAC permission group, and MongoDB index definitions that all subsequent tickets build on. No handler, no routes, no tests in this ticket.

Model —> dashboard/model_ruleset.go (new file)


package dashboard
import (
    "time"
    "github.com/TykTechnologies/tyk-analytics/dashboard/model"
    storagemodel "github.com/TykTechnologies/storage/persistent/model"
)
type RulesetResourceType string
const (
    RulesetResourceTypeOASAPI RulesetResourceType = "oas-api"
)
type RulesetAction string
const (
    RulesetActionNone RulesetAction = "none"
    RulesetActionWarn RulesetAction = "warn"
)
type Ruleset struct {
    ID           storagemodel.ObjectId `bson:"_id,omitempty"    json:"_id"`
    RulesetID    string                `bson:"ruleset_id"       json:"ruleset_id"`
    OrgID        string                `bson:"org_id"           json:"org_id"`
    Name         string                `bson:"name"             json:"name"`
    Description  string                `bson:"description"      json:"description"`
    Active       bool                  `bson:"active"           json:"active"`
    ResourceType RulesetResourceType   `bson:"resource_type"    json:"resource_type"`
    Action       RulesetAction         `bson:"action"           json:"action"`
    Categories   model.Categories      `bson:"categories"       json:"categories,omitempty"`
    Owner        string                `bson:"owner"            json:"owner"`
    Definition   string                `bson:"definition"       json:"definition"`
    Rules        []RulesetRule         `bson:"rules"            json:"rules"`
    IsTemplate   bool                  `bson:"is_template"      json:"is_template"`
    DateCreated  time.Time             `bson:"date_created"     json:"date_created"`
    LastUpdated  time.Time             `bson:"last_updated"     json:"last_updated"`
    DeletedAt    *time.Time            `bson:"deleted_at,omitempty" json:"-"`
}
type RulesetRule struct {
    ID         storagemodel.ObjectId `bson:"_id"        json:"id"`
    Name       string                `bson:"name"       json:"name"`
    Active     bool                  `bson:"active"     json:"active"`
    Definition string                `bson:"definition" json:"definition"`
}
func (r *Ruleset) GetObjectID() storagemodel.ObjectId   { return r.ID }
func (r *Ruleset) SetObjectID(id storagemodel.ObjectId) { r.ID = id }
func (r *Ruleset) TableName() string                    { return "tyk_rulesets" }
Field notes:

Field

Notes

ID 

MongoDB _id, auto-assigned on insert 

RulesetID 

UUID auto-generated server-side on creation, never changed. Stable cross-env identifier, allows the same ruleset to be referenced across dev/staging/prod. GetByID accepts either an ObjectId hex (→ queries _id) or a RulesetID string (→ queries ruleset_id). 

Owner 

Set server-side from request.Session(r).EmailAddress —> callers cannot set it 

DeletedAt 

Soft-delete field; excluded from json output. All queries filter "deleted_at": {"$eq": nil} 

Categories 

Reuses existing model.Categories ([]string); links rulesets to APIs via shared category tags 

RBAC —> config/config.go
Add a new ObjectGroup constant:



GovernanceGrp ObjectGroup = "governance"
Add GovernanceGrp to the IsValid() switch:



case BaseGrp, AnalyticsGrp, ..., AuditLogsGrp, GovernanceGrp:
RBAC —> dashboard/user_permissions.go
Add to PermissionGroups slice:



config.GovernanceGrp,
Add to PermissionMatchGroups slice:



{"\\/api\\/rulesets", config.GovernanceGrp},
isPermitted() automatically enforces:

HTTP method

Minimum level

POST / PUT / DELETE 

write or admin 

GET 

read, write, or admin 

MongoDB indexes
Defined in the repository constructor (Ticket). Documented here so the model ticket serves as the reference for reviewers.

Index name

Fields

Purpose

name 

name asc 

name lookups 

org_id 

org_id asc 

all queries are org-scoped 

org_id_name 

org_id + name compound 

name uniqueness per org 

ruleset_id 

ruleset_id asc 

migration-stable ID lookup 

resource_type 

resource_type asc 

filtering by resource type 

categories 

categories asc 

API↔ruleset linking 

storage.CreateIndex(ctx, row, index) —> implemented in the NewRepository constructor in Ticket to create a Rulesets.

Swagger schemas —> update swagger.yml
Add under components/schemas:: Swagger definition for gov
 

Acceptance criteria
dashboard/model_ruleset.go compiles with all fields above

Ruleset implements the DBObject interface (GetObjectID, SetObjectID, TableName)

GovernanceGrp added to config/config.go and included in IsValid()

GovernanceGrp added to PermissionGroups and PermissionMatchGroups

All four Swagger schemas added to swagger.yml

No handler, routes, or tests in this ticket

#### Investigate and implement how the backedn will parse MCP for rulset configration

Product idea
We will explain how governance rulesets apply to MCP API definitions registered in Tyk analytics, what the evaluation flow looks like, what is supported out of the box, and where the boundaries are.

 

Why it is compatible
MCP definitions are OAS 3.0 documents
An MCP API registered in Tyk Analytics is stored and exported as a standard OpenAPI 3.0 document. The Tyk-specific configuration lives in the x-tyk-api-gateway vendor extension, which is legal under the OAS specification and any tool that can parse OAS 3.0 can read the file without modification.

Governance rulesets use Spectral-compatible rule definitions. Spectral (and compatible runners such as Vacuum) operate on the parsed JSON/YAML document tree via JSONPath expressions. Because x-tyk-api-gateway is just another node in that  tree, rules can target MCP-specific fields with the same mechanism used for standard OAS fields.




MCP definition (OAS 3.0 + x-tyk-api-gateway)
        │
        │  passed as-is —> no transformation
        ▼
  Vacuum / Spectral runner
        │
        │  evaluates each rule's `given` (JSONPath) + `then` (function)
        ▼
  Violations list (severity: warn | error | info | hint)
The two addressable sections
An MCP definition has two distinct areas that rulesets can target:

1. Standard OAS section
Fields that exist in any OAS 3.0 document:

Field

JSONPath

API title 

$.info.title 

Servers 

$.servers[*].url 

Paths and operations 

$.paths[*][*] 

Operation IDs 

$.paths[*][*].operationId 

Security schemes 

$.components.securitySchemes 

Global security 

$.security 

2. Tyk extension section (x-tyk-api-gateway)
x-tyk-api-gateway is the standard Tyk OAS vendor extension present in all Tyk OAS API definitions, not just MCP APIs. It carries gateway configuration that Tyk Dashboard and Gateway read at runtime.

Fields common to all Tyk OAS APIs (oas-api and mcp-api alike):

Field

JSONPath

Authentication enabled 

$['x-tyk-api-gateway'].server.authentication.enabled 

Listen path 

$['x-tyk-api-gateway'].server.listenPath.value 

Upstream URL 

$['x-tyk-api-gateway'].upstream.url 

API active state 

$['x-tyk-api-gateway'].info.state.active 

Operation-level middleware 

$['x-tyk-api-gateway'].middleware.operations 

Fields specific to MCP APIs (only present when resource_type: mcp-api):

Field

JSONPath

MCP tool definitions 

$['x-tyk-api-gateway'].middleware.mcpTools 

Per-tool allow 

$['x-tyk-api-gateway'].middleware.mcpTools.*.allow.enabled 

Per-tool rate limit 

$['x-tyk-api-gateway'].middleware.mcpTools.*.rateLimit.enabled 

Per-tool rate limit per 

$['x-tyk-api-gateway'].middleware.mcpTools.*.rateLimit.per 

Per-tool request header transforms 

$['x-tyk-api-gateway'].middleware.mcpTools.*.transformRequestHeaders 

What can be validated (supported)
The following checks can be expressed as standard Spectral rules using built-in functions (truthy, falsy, defined, undefined, pattern, enumeration, schema, etc.):

Check

Applies to

Rule approach

Authentication is enabled 

oas-api, mcp-api 

given: $['x-tyk-api-gateway'].server.authentication → field: enabled → truthy 

Upstream URL is not empty 

oas-api, mcp-api 

given: $['x-tyk-api-gateway'].upstream → field: url → truthy 

Upstream URL is not a test/localhost value 

oas-api, mcp-api 

given: $['x-tyk-api-gateway'].upstream.url → pattern: notMatch: localhost\\|httpbin 

API is active 

oas-api, mcp-api 

given: $['x-tyk-api-gateway'].info.state → field: active → truthy 

Listen path is set 

oas-api, mcp-api 

given: $['x-tyk-api-gateway'].server.listenPath → field: value → truthy 

Operation IDs follow naming convention 

oas-api, mcp-api 

given: $.paths[*][*].operationId → pattern: match: ^[a-z][a-zA-Z0-9]+$ 

All MCP tools have rate limiting on 

mcp-api only 

given: $['x-tyk-api-gateway'].middleware.mcpTools[*].rateLimit → field: enabled → truthy 

All MCP tools are allowed 

mcp-api only 

given: $['x-tyk-api-gateway'].middleware.mcpTools[*].allow → field: enabled → truthy 

Each tool injects an identifying header 

mcp-api only 

given: $['x-tyk-api-gateway'].middleware.mcpTools[*] → field: transformRequestHeaders → defined 

Users can also write any valid Spectral rule with a custom definition field, the ruleset platform executes whatever rule definition is stored without modification.

What cannot be validated without custom Spectral functions
Some checks require correlating values across different parts of the document. Spectral's built-in functions evaluate each given path independently; they cannot join two node sets in a single rule.

Check

Why it cannot use built-in functions

Every operationId in paths has a corresponding entry in middleware.operations 

Requires cross-referencing two independent node sets 

Every tool name in mcpTools corresponds to a real MCP tool exposed by the upstream 

Requires knowledge outside the document 

Rate limit per value is a valid duration string (e.g. 20s, 1m) 

Pattern matching works, but semantic validation of the format requires a schema or custom function 

These checks can be implemented as custom Spectral functions. Users provide the function implementation as part of their rule definition. The ruleset platform executes it as-is  no platform changes are needed to support this.

 

Resource type
MCP API rulesets use resource_type: mcp. This allows the evaluation process to select only relevant rulesets when linting a given document, and allows teams to manage OAS and MCP rulesets independently even when they share the same category tags.

Acceptance criteria
Definition of done. How do we know the story is completed.
Should be filled by the reporter.


Testing
in this ticket we have 2 files attached: mcp-definition.yml and example-mcp-ruleset.yml which I used to test the analysis using vacuum, basically we just want to check that the mcp has authentication enabled. If you are willing to give it a try just download the files, and execute this in any terminal:

 vacuum lint --ruleset example-mcp-ruleset.yaml --skip-check --details mcp-definition.yml

 

So far the output should look like:



 INFO  Linting file 'mcp-definition.yml' against 1 rules: 
----------------------------------------------------------------------------------
Location                | Severity | Message                                                                             | Rule                        | Category   | Path     
mcp-definition.yml:65:7 | warning  | MCP API must have authentication enabled (x-tyk-api-gateway.server.authenticatio... | mcp-authentication-required | Validation | $.enabled
Category   | Errors | Warnings | Info
Validation | 0      | 1        | 0   
          Linting passed, but with 1 warnings and 0 informs  


#### TT-17023 UI for API Ruleset listing

Summary
Introduce a Governance section to the Dashboard and implement the Rulesets listing page with name search, category filter, and pagination.

Context
Tyk is expanding into API governance capabilities. The Dashboard currently has no governance section, no navigation entry, and no routes for governance rulesets.

This ticket is the starting point for governance functionality. It establishes:

The Governance navigation group in the Dashboard left sidebar

The Rulesets listing page, which is searchable, filterable, and paginated

The backend API required to power the listing page

The create, edit, and delete flows for rulesets are deferred to separate follow-up tickets. A "Create new ruleset" button will be present in the UI as a navigation entry point, but the destination page is out of scope here.

Current Behaviour
The Dashboard left navigation has no Governance section

There are no routes for any governance-related page

No backend API exists for governance rulesets

Expected Behaviour
Navigation
A Governance group appears in the Dashboard left sidebar (positioned after AI Protocols) with the following link:

Link

Path

Ruleset

/governance/rulesets

Rulesets listing page — /governance/rulesets
NavBar
Title: "Rulesets" with a help tooltip

Right side: "+ Create new ruleset" primary button (navigates to /governance/rulesets/add — page implementation is deferred)

Search
A "Table Search" text input filters rulesets by name

Search term reflected in URL query param q

Category filter
A "Category" dropdown filters rulesets by category

Default selection: "All" (no filter)

Options loaded on page mount from the existing categories API already used by the API listing page

Selected category reflected in URL query param category

Changing the category resets pagination to page 1

Table
Column

Description

Status

Active/Inactive icon (active = green circle-check, inactive = grey circle-minus)

Name

Ruleset name — clickable link (detail page is a follow-up ticket)

Category

Category the ruleset belongs to

Reports

Associated report type displayed as badge

Last updated

Date last modified, formatted DD-MM-YYYY

Pagination
Displays 1–10 of N total results

Rows per page: 10, 25, 50 (default: 10)

Page navigation

URL state persistence
All search, filter, and page params are stored in the URL

Refreshing the page restores the previous state

Empty states
No rulesets, no active filters → empty state with message

Search/filter returns no results → "No rulesets found" message

Backend Requirements
Data Model — Ruleset


{
  "id": "string (UUID)",
  "name": "string",
  "category": "string",
  "reports": "string",
  "status": "active | inactive",
  "updated_at": "ISO 8601 timestamp"
}
Endpoint 1 — GET /api/governance/rulesets
Returns a paginated, searchable, filterable list of rulesets for authenticated users.

Example:



GET /api/governance/rulesets?q=auth&category=Security&p=2&sort=-updated_at
Query parameters
Param

Type

Default

Description

q

string

—

Search term matched against name (case-insensitive, partial match)

category

string

—

Exact match filter on category; omit or empty = all categories

p

integer

1

Page number

sort

string

-updated_at

Sort field; prefix - for descending

Response 200 OK


{
  "data": {
    "rulesets": [
      {
        "id": "uuid",
        "name": "API Authentication Enforcement",
        "category": "Default",
        "reports": "Security & Risk",
        "status": "active",
        "updated_at": "2025-01-01T00:00:00Z"
      }
    ],
    "pages": 10
  }
}
Category options
 The category dropdown must use the existing categories API already used by the API listing page 

 No new categories endpoint should be introduced 

Backend implementation notes
 The rulesets endpoint requires a valid authenticated session 

 Results are scoped to the authenticated user's organisation 

q search must be case-insensitive and support partial matches on name 

 GET /api/governance/rulesets follows the q, p, sort convention of /api/mcps 

 The category dropdown reuses the existing categories response shape used elsewhere in the product 

What We Will Do
1. Navigation
Add a Governance group with icon shield-check, containing:

 Ruleset link → /governance/rulesets 

2. Routes
Register one route:

governance/rulesets → GovernanceRulesets (full listing page) 

3. Rulesets listing page
UI to view list of rulesets, with search, filter, and pagination capabilities

Backend Dependencies
 Add DB schema / storage for rulesets (org-scoped) 

 Implement GET /api/governance/rulesets with q, category, p, sort params 

 Reuse the existing categories API already used by the API listing page 

 Apply existing auth middleware 

Test Cases
Test Case 1: Navigation — Governance group appears
Log in to the Dashboard

Expected: Left sidebar contains a "Governance" group with a "Ruleset" link

Test Case 2: Rulesets listing loads with data
Click "Ruleset"

Expected:

 Table shows data 

 Category dropdown populated from existing categories API 

 Pagination visible 

Test Case 3: Name search
Type "auth"

Expected:

 API called with q=auth&p=1 

 URL updates to ?q=auth 

 Pagination resets 

Test Case 4: Category filter
Select "Security"

Expected:

 API called with category=Security&p=1 

 URL updates 

 Pagination resets 

Test Case 5: Combined search and filter
Search "rate" + category "Default"

Expected:

 API called with both params 

 Results match both 

Test Case 6: Pagination
Go to page 2

Expected:

 API called with p=2 

 URL updates 

Test Case 7: URL state restored
Refresh with params

Expected: State persists

Test Case 8: Empty state
No rulesets

Expected: Empty state shown using SVG or a message box

Test Case 9: No search results
No matches

Expected: "No rulesets found"

Test Case 10: Create button
Click button

Expected: Navigates to /governance/rulesets/add

Acceptance Criteria
 Governance group exists with Ruleset link 

/governance/rulesets loads data from API 

 Category dropdown uses existing categories API 

 Category filter updates URL and API calls 

 Search + filter work together 

 Pagination resets on filter/search change 

 Pagination works correctly 

 Empty state shown correctly with SVG + feature description + CTA(create…) + Learn more (link to docs)

 "No rulesets found" handled correctly 

 Create button navigates correctly 

 Endpoint is authenticated and org-scoped 

 All existing tests pass 

Out of Scope
 Create ruleset 

 Edit ruleset 

 Delete ruleset 

 Ruleset detail page 

 RBAC / permission gating for Governance 

#### TT-17024 UI for API Ruleset Creation

Summary
Implement a multi-step Ruleset creation wizard, allowing users to create a ruleset either from an existing template or from their own definition (via file upload or code editor), followed by basic info configuration.

Context
This ticket implements the "Create new ruleset" flow navigated to from the Rulesets listing page (/governance/rulesets) when the user clicks "+ Create new ruleset".

https://www.figma.com/design/jESbot2td2DuJB0SEBBVGT/Governance-Dashboard?node-id=696-27919&p=f&m=dev

Current Behaviour
No UI exists to create governance rulesets.

Expected Behaviour
Route
/governance/rulesets/add

NavBar
Title: "Create new ruleset" with help tooltip

No action buttons in the NavBar — navigation is handled by the wizard's built-in buttons at the bottom

Wizard form for creating a ruleset 4 steps (Step 3 conditional)
Step 1 — Choose Starting Point
Two option cards presented side by side:

Option

Description

Start from template 

Use an existing governance ruleset template as a starting point. Templates exist in the user's file system, picked up by the backend and made available via the API 

Start with your own definition 

Upload a file or paste your own ruleset definition 

One option must be selected to proceed

Selected option determines which steps follow (Step 3 only shown for template flow)


Screenshot 2026-04-28 at 12.57.35 PM.png
 

Step 2 — (Dynamic based on Step 1 selection)
If "Start from template" was selected:

A searchable, filterable list of ruleset templates fetched from the backend:

Search — text input to filter templates by name (debounced ~300ms)

Category filter — dropdown to filter by category

Template list — user selects one template to proceed

Fetched from GET /api/rulesets?is_template=true&p=-1

If "Start with your own definition" was selected:

Two sub-options:

Option

Description

Import from file 

User uploads a .json or .yaml file; content parsed and loaded 

Paste ruleset 

Monaco code editor with JSON syntax highlighting; user pastes or types content 

Content must be non-empty to proceed



Screenshot 2026-04-28 at 12.59.45 PM.png
Screenshot 2026-04-28 at 1.16.27 PM.png
 

Step 3 — Review & Edit (template flow only)
Selected template's JSON content loaded into a Monaco code editor

User can review and edit before proceeding

Content must be non-empty to proceed

Skipped entirely in the own-definition flow (would be redundant)

Screenshot 2026-04-28 at 1.01.16 PM.png
 

Step 4 — Basic Info (all flows)
Field

Required

Description

Name 

Yes 

Human-readable name; unique within the org 

Description 

No 

Long-form description 

Action 

No 

warn — violations flagged before deployment; none — no action (default). Maps to the deployment warning concept in the UI 

Link ruleset to APIs 

No 

Associate ruleset with API categories; options fetched from GET /api/apis/categories 

Finish triggers POST /api/rulesets

On success: toast shown; user navigated to /governance/rulesets/{ruleset_id}

On error: error toast shown; user stays on the wizard





Screenshot 2026-04-28 at 1.02.44 PM.png
Stepper navigation
Control

Behaviour

Continue 

Validate current step; advance if valid 

Back 

Return to previous step (no validation) 

Finish (Step 4) 

Submit POST /api/rulesets 

Cancel 

Navigate back to /governance/rulesets without saving 

Step validation prevents advancing if required fields are incomplete.

Backend Requirements
Endpoint 1 — GET /api/rulesets
When p=-1 the backend must return all matching templates without pagination — required by the template selection step.

Example:



GET /api/rulesets?is_template=true&p=-1
GET /api/rulesets?is_template=true&name=auth&p=1
Response shape is identical to the standard listing response.

Endpoint 2 — POST /api/rulesets
Creates a new governance ruleset for the authenticated organisation.

Request body:



{
  "name": "My Ruleset",
  "description": "Enforces authentication standards across all APIs",
  "action": "warn",
  "categories": ["Security", "Compliance"],
  "is_template": false,
  "rules": [
    {
      "name": "operation-success-response",
      "active": true,
      "definition": "..."
    }
  ]
}
Request fields:

Field

Type

Required

Description

name 

string 

Yes 

Unique within the org 

description 

string 

No 

Long-form description 

action 

string 

No 

warn or none (default: none) 

categories 

string[] 

No 

API categories to associate the ruleset with 

is_template 

boolean 

No 

Whether this ruleset is a template (default: false) 

rules 

Rule[] 

Yes 

Array of rule objects 

rules[].name 

string 

Yes 

Rule identifier 

rules[].active 

boolean 

Yes 

Whether the rule is active 

rules[].definition 

string 

Yes 

Rule definition in JSON format 

Response 201 Created:



{
  "Status": "success",
  "Message": "ruleset created",
  "Meta": "<created-ruleset-id>"
}
Error responses: 400 (validation), 401, 403, 409 (duplicate name), 500

Permissions: Requires write or admin on the governance permission group.

UX Dependencies
The following decisions need to be made before implementation:

Active/Inactive in wizard — Should the user be able to set the ruleset active status in the Basic Info step (Step 4), or should it default to a specific state on creation and be changed only from the details page?

Test Cases
Test Case 1: Template list renders API data correctly
GET /api/rulesets?is_template=true&p=-1 returns templates

Expected: Template names and categories rendered correctly in the Step 2 list

Test Case 2: Correct payload sent to POST /api/rulesets on Finish
User completes all steps and clicks Finish

Expected: POST /api/rulesets called with name, rules, categories, action, and is_template: false reflecting the wizard inputs

Test Case 3: File upload populates rules correctly
User uploads a valid .json file in the own-definition flow

Expected: File content correctly mapped to the rules field in the API payload

Test Case 4: 409 conflict error displayed
POST /api/rulesets returns 409

Expected: Error message shown to the user; no navigation away from Step 4

Acceptance Criteria
[ ] Wizard renders at /governance/rulesets/add with 4 steps (3 for own-definition flow)

[ ] Step 1 presents two option cards; one must be selected to proceed

[ ] Template flow Step 2 loads templates from GET /api/rulesets?is_template=true&p=-1

[ ] Template list is searchable by name and filterable by category

[ ] If no templates are available show a message box with appropriate copy and link to the documentation.

[ ] Own-definition flow Step 2 offers file upload (.json) and Monaco paste options

[ ] Step 3 (Review & Edit) only appears in template flow; Monaco pre-filled with selected template JSON

[ ] Step 3 does not appear in own-definition flow

[ ] Step 4 has Name (required), Description, Action (warn/none), and Categories multi-select

[ ] Categories multi-select populated from GET /api/apis/categories

[ ] Stepper Back button preserves previously entered state

[ ] Step validation prevents advancing when required fields are empty

[ ] Finish calls POST /api/rulesets with the full payload

[ ] Successful creation shows a toast and redirects to /governance/rulesets/{ruleset_id}

[ ] API error shows an error toast and keeps the user on Step 4

[ ] 409 duplicate name error shown without navigating away

[ ] UX Dependencies #1 and #2 resolved before final implementation

[ ] POST /api/rulesets is authenticated and org-scoped

Out of Scope
Editing an existing ruleset (follow-up ticket)

Deleting a ruleset (follow-up ticket)

Ruleset versioning

Marking a ruleset as a template from the UI

Real-time JSON validation / linting in the code editor

Related Information
OpenAPI spec: POST /api/rulesets — RulesetRequest schema, action: none|warn

Companion tickets: Governance — Rulesets Listing Page, Governance — Ruleset Details Page

Design reference: 4-step stepper wizard; template list in Step 2; Monaco editor in Steps 2b/3; form in Step 4

#### TT-17025 UI for API Ruleset view, edit, delete

Summary
Implement the Ruleset details/edit page at /governance/rulesets/:rulesetId with two tabs — 
1. Basic configs 
2. Raw definition configuration 

 Supporting  
- GET /api/rulesets/:rulesetId 
- DELETE /api/rulesets/:rulesetId (soft delete — document retained with deleted_at set)
- PUT /api/rulesets/:rulesetId.

Context
This ticket implements the detail/edit/delete page for an existing governance ruleset. Users navigate here by clicking a ruleset name in the Rulesets listing page.

The page is titled "Configure ruleset information" 

Current Behaviour
Clicking a ruleset name in the listing goes nowhere (the detail page does not exist).

Expected Behaviour
Route /governance/rulesets/:rulesetId renders the ruleset details page for view, update, delete operation
NavBar
Title: "Configure ruleset information" with help tooltip

Right side: Cancel (secondary) + Save (primary)

Cancel navigates back to /governance/rulesets without saving

Save triggers PUT /api/rulesets/:rulesetId

Tabs
Tab

Content

Basic configs (default) 

Active status, name, description, categories, action, deprecation

Raw definition configuration 

Monaco JSON editor showing the ruleset definition 

Tab 1 — Basic configs
Basic configs (Panel)
Ruleset status

Help tooltip: "When active, this ruleset will immediately evaluate all mapped APIs once saved. If you'd like to review or test it first, keep the status as Inactive."

Radio group:

Draft — active: false — "Does not evaluate APIs"

Active — active: true — "Evaluates all mapped APIs"

Ruleset name

Text input with help tooltip

Maps to name field (required, unique within org)

Description

Textarea with help tooltip

Maps to description field

Link ruleset to APIs — Category

Description: "You can link which APIs this ruleset can be evaluated against, results will show in the API overview page."

Multi-select with removable chips (e.g., "External ×")

Maps to categories field

Options loaded from categories endpoint 

Deployment warning 

Help tooltip: "When enabled, all mapped APIs are checked against this ruleset before deployment. Any violations are flagged."

Toggle: "Activate deployment warning"

Maps to action: warn (enabled) / action: none (disabled)


Advanced configs (Panel)
Deprecate ruleset

Button with trash icon, danger-outline styling

Description: "The ruleset will be permanently deprecated from use. It will remain visible in any reports where it was previously applied, but you won't be able to use it again or recover it."

Clicking opens a Confirm modal

On confirm: calls DELETE /api/rulesets/:rulesetId (soft delete — document retained with deleted_at set)

On success: toast shown; redirect to /governance/rulesets

Tab 2 — Raw definition configuration
Monaco code editor showing the ruleset's rules content as JSON

default editor language will be json

Editable; changes reflected in the update payload

Pre-filled from the fetched ruleset data

Backend Requirements
GET /api/rulesets/:rulesetId
Fetch a single ruleset by ruleset_id string

Response 200 OK — returns the full Ruleset object including rules[]:



{
  "_id": "6644b38535715ec496cbef3d",
  "ruleset_id": "my-oas-ruleset",
  "org_id": "default",
  "name": "OAS API Ruleset",
  "description": "Validates OAS APIs",
  "active": true,
  "resource_type": "oas-api",
  "action": "warn",
  "categories": ["payments"],
  "owner": "user@example.com",
  "is_template": false,
  "rules": [
    { "name": "no-x-headers", "active": true, "definition": "..." }
  ],
  "date_created": "2024-05-15T10:00:00Z",
  "last_updated": "2024-05-15T10:00:00Z"
}
Error responses: 401, 403, 404, 500

PUT /api/rulesets/:rulesetId
Update an existing ruleset. Accepts the same RulesetRequest body as POST.

ruleset_id is never overwritten — any ruleset_id in the request body is ignored

Accepts application/json or multipart/form-data

Response 200 OK — returns the updated Ruleset object, shows error state in toast message.

DELETE /api/rulesets/:rulesetId
Soft-delete a ruleset. The document is retained in the database with deleted_at set. This is the "Deprecate ruleset" action in the UI.

Response 200 OK:



{
  "Status": "success",
  "Message": "ruleset deleted",
  "Meta": null
}
Error responses: 401, 403, 404, 500

Note: The design refers to this as "Deprecate", but the API implements it as a soft delete. Once deleted, the ruleset cannot be re-activated.

Test Cases
Test Case 1: API data rendered correctly in Basic configs tab
GET /api/rulesets/:rulesetId returns a ruleset

Expected: name, description, active, categories, and action values pre-populate the correct fields; active: true → Active radio selected; action: warn → deployment warning toggle on

Test Case 2: API data rendered correctly in Raw definition tab
GET /api/rulesets/:rulesetId returns a ruleset with rules

Expected: Rules JSON is displayed in the Monaco editor on the Raw definition tab

Test Case 3: Correct payload sent to API on Save
User changes name and categories; clicks Save or user makes edit to the raw definition

Expected: PUT /api/rulesets/:rulesetId called with updated fields, and all other current field values

Test Case 4: Deprecate sends correct API call
User confirms the deprecate action

Expected: DELETE /api/rulesets/:rulesetId called; on Status: success response, user is redirected to /governance/rulesets

Acceptance Criteria

Page loads and pre-populates from GET /api/rulesets/:rulesetId

"Basic configs" tab is active by default

Status radio pre-set from active field; Draft = false, Active = true

Deployment warning toggle pre-set from action field 

Category multi-select pre-set from categories; options from categories endpoint

"Raw definition configuration" tab renders Monaco JSON editor pre-filled with rules

Save calls PUT /api/rulesets/:rulesetId; success toast shown; user stays on page and updated information is retained.

409 name conflict error handled gracefully

Cancel navigates to /governance/rulesets without saving

"Deprecate ruleset" click opens a confirmation modal Ref : https://www.figma.com/design/jESbot2td2DuJB0SEBBVGT/Governance-Dashboard?node-id=748-55067&m=devConnect your Figma account 

Confirming calls DELETE /api/rulesets/:rulesetId; redirect to listing

GET, PUT, DELETE /api/rulesets/:rulesetId all authenticated and org-scoped

DELETE is soft-delete (document retained with deleted_at)

Any error messages from the API are shown in the toast messages.
Out of Scope
Re-activating a soft-deleted ruleset

Test ruleset section (used to test ruleset against selected API’s)

#### TT-17026 BE - Vacuum update
Context and preconditions
In the governance PoC we forked the Vacuum library (github.com/buraksekili/vacuum) because of a panic. Upstream (github.com/daveshanley/vacuum) has shipped 50+ releases since (v0.17.14 → v0.26.1). Before any other governance story consumes Vacuum from tyk-analytics, we need an evidence-based decision on fork vs. upstream so downstream stories pin the right library.

Product idea
Produce a decision (with numbers) on whether to keep the fork or move to upstream, captured in an ADR that downstream tickets in the Ruleset epic and the Evaluation epic can reference. Actual library integration is out of scope here — it lives in the stories that consume the lib.

What we will do
.

Surface analysis — diff the public API of motor, rulesets, model between fork (buraksekili/vacuum@b857c8f) and upstream (daveshanley/vacuum v0.26.1). Classify each symbol match / added / removed / signature-changed, and emit a verdict (drop-in / minor-drift / breaking-drift).

Performance benchmark — measure end-to-end motor.ApplyRulesToRuleSet for both libs across 3 spec sizes × 5 rulesets (15 scenarios). Report ns/op, B/op, allocs/op, plus a Δ column. Reproducible.

Dependency analysis (operational + security) — enumerate the transitive module graph upstream v0.26.1 brings in, classify by role (core / rule-execution / serialiser / CLI-only / LSP-only / surprise), run govulncheck and an Advisory-DB sweep, audit licenses across the audited subset, and capture maintenance-health signals (release cadence, contributor concentration, bus factor) for Vacuum + the pb33f core. Flag the live security concerns the integrating stories must handle: customer-supplied JS rules executing in goja, remote $ref resolution as an SSRF vector, multiple YAML parsers / pre-1.0 deps in scope.

Publish the harness as a standalone, runnable repo so the numbers can be re-checked or re-run on bumped pins: github.com/tbuchaillot/vacuum-benchmark.

Write the ADR — context, findings (perf table + parity verdict + dep analysis), decision, consequences, alternatives considered. Land it at lib_confluence.md so downstream stories link to a single source of truth.

Library integration into tyk-analytics (CRUD wiring, motor.RuleSetExecution mapping, Dashboard violation surfacing, JS-rule sandboxing, $ref-fetch policy, full license-check job, etc.) is handled by the stories that consume the lib — see the Ruleset epic and Evaluation epic. This ticket only produces the decision and the constraints those stories must respect.

Acceptance criteria
Benchmark project published, reproducible (make fetch && make bench), with pinned spec SHAs and pinned vacuum versions for both fork and upstream.

Results report (results/results.md) covers 3 sizes × 5 rulesets and includes a Δ column.

Contract report (results/contracts.md) lists every public symbol in motor / rulesets / model with a status, plus a deterministic verdict.

Dependency analysis section in the ADR covers: transitive-module count and bucketed inventory, govulncheck result, GitHub Advisory DB sweep for vacuum + pb33f core + goja, license posture for the audited subset, maintenance-health signals (release cadence, contributor concentration), and an explicit list of integration-time constraints (custom-JS sandboxing, remote-$ref policy, license-check-job sweep, dependabot scope).

ADR  is published with: status, context, findings, dependency analysis (operational + security), decision, consequences, alternatives, references.

Decision is communicated to the team and linked from downstream tickets in the Ruleset and Evaluation epics, with the security/ops constraints called out so they aren't dropped on integration.

Testing
No testing needed

#### TT-17067 Operator Integration - TykRuleset CRD
Operator integration - TykRuleset CRD
Context and preconditions
Tyk Operator gives platform engineers a GitOps experience for Tyk: APIs, Security Policies, OperatorContext, and (since PR #172) MCP Proxy Definitions are all declared as Kubernetes resources and reconciled into the Dashboard. With the introduction of the Governance feature, the same audience will need to manage Rulesets in CI/CD pipelines and Argo/Flux workflows alongside the APIs they govern.

The ruleset epic ships full CRUD for rulesets in the Dashboard (POST/GET/PUT/DELETE /api/rulesets). Once those endpoints are merged, the operator should expose them as a CRD so that:

A ruleset definition can live in a Git repo next to the APIs it applies to.

Categories, action, and active state can be flipped in PRs and rolled back via git revert.

The Dashboard stops being the single source of truth for governance config in environments that already manage everything else through the operator.

PR #172 (MCP integrations) is the canonical template for adding a new CRD to tyk-operator-internal: it adds a TykMcpProxyDefinition CRD whose spec references a ConfigMap holding an OAS document, plus a controller that creates/updates/deletes the matching resource on the Dashboard. The shape we need for Rulesets is structurally identical, with one important difference: the Dashboard generates ruleset_id server-side and forbids the caller from setting OrgID or Owner, so the operator must observe the ID after Create rather than choose it upfront.

Preconditions:

The four CRUD ruleset tickets are merged into the Dashboard.

The OperatorContext referenced by the CR carries a Dashboard auth token with write or admin on GovernanceGrp.

Product idea
Provide a TykRuleset Custom Resource so users can write:



apiVersion: tyk.tyk.io/v1alpha1
kind: TykRuleset
metadata:
  name: payments-ruleset
  namespace: tyk
spec:
  contextRef:
    name: dashboard-context
    namespace: tyk
  name: Payments Ruleset
  description: Validates Payments OAS APIs for compliance
  active: true
  action: warn
  resourceType: oas-api
  categories: [payments, internal]
  ruleset:
    # Exactly one of `raw` or `objRef` must be set.
    raw: |
      extends: ["spectral:oas"]
      rules:
        operation-operationId:
          severity: error
          ...
    # objRef:
    #   kind: ConfigMap         # or Secret
    #   name: payments-ruleset-spec
    #   namespace: tyk
    #   keyName: ruleset.yaml
…and have the operator create the corresponding ruleset in the Dashboard, keep it in sync with edits to the CR (or any referenced object), and delete it when the CR is removed.

The Spectral JSON/YAML body can be supplied two ways:

ruleset.raw — body inline in the CR. Best for small/medium rulesets and tight git workflows. One-file, one-PR diffs.

ruleset.objRef — body in a separate ConfigMap or Secret. Best for large or sensitive rulesets, or when the same body is shared by multiple TykRuleset CRs. Secret is intended for rulesets that embed internal-only patterns (endpoints, customer naming) that shouldn't sit in a plain ConfigMap.

Exactly one of raw / objRef must be set, enforced by a CEL validation rule on the CRD. tykRef-style adoption of pre-existing Dashboard rulesets is not in v1; if needed it ships as a separate ownership: adopt mode in a follow-up.

Out of scope for this story:

Epic 2 (evaluation engine) — running rulesets against APIs is a separate CRD/action.

Rule-builder UI — operator-only feature.

Inline-in-spec ruleset bodies — defer until customer demand exists.

Templates (is_template = true) — admins curate org templates via the Dashboard, not GitOps.

What we will do
Add a TykRuleset CRD, a TykRulesetReconciler, and a Ruleset universal client. Mirror PR #172 file-for-file. Phase the work as Types → Read-only reconcile → Create/Update → Delete/finalizer.

Type definitions — api/v1alpha1/tykruleset_types.go (new file)
Source: copy-paste structure from api/v1alpha1/tykmcpproxydefinition_types.go.



type TykRulesetSpec struct {
    // Context overrides the OperatorContext used for this resource.
    Context *model.Target `json:"contextRef,omitempty"`
    // Name is the Dashboard-visible name of the ruleset. Required, unique
    // within the org. Maps 1:1 to analytics Ruleset.Name.
    Name string `json:"name"`
    // Description is the free-text description.
    Description string `json:"description,omitempty"`
    // Active toggles enforcement. Defaults to false.
    Active bool `json:"active,omitempty"`
    // Action governs what happens when a rule fires.
    // +kubebuilder:validation:Enum=none;warn
    Action string `json:"action,omitempty"`
    // ResourceType is what this ruleset applies to.
    // +kubebuilder:validation:Enum=oas-api
    ResourceType string `json:"resourceType,omitempty"`
    // Categories scopes this ruleset for evaluation.
    Categories []string `json:"categories,omitempty"`
    // IsTemplate marks the ruleset as an org template. Defaults to false.
    IsTemplate bool `json:"isTemplate,omitempty"`
    // Ruleset is the source of the Spectral JSON/YAML body. Exactly one of
    // raw / objRef must be set.
    Ruleset RulesetSource `json:"ruleset"`
}
// RulesetSource specifies where the Spectral body comes from.
//
// +kubebuilder:validation:XValidation:rule="(has(self.raw) ? 1 : 0) + (has(self.objRef) ? 1 : 0) == 1",message="exactly one of ruleset.raw or ruleset.objRef must be set"
type RulesetSource struct {
    // Raw is the Spectral JSON/YAML body inlined in the CR.
    Raw *string `json:"raw,omitempty"`
    // ObjRef points at a ConfigMap or Secret whose data[keyName] holds
    // the Spectral JSON/YAML body.
    ObjRef *RulesetObjectReference `json:"objRef,omitempty"`
}
// RulesetObjectReference points at a ConfigMap or Secret in the cluster.
type RulesetObjectReference struct {
    // Kind is the referenced object kind. Only ConfigMap and Secret are supported.
    // +kubebuilder:validation:Enum=ConfigMap;Secret
    Kind string `json:"kind"`
    // Name is the object name.
    Name string `json:"name"`
    // Namespace defaults to the CR's namespace when empty.
    Namespace string `json:"namespace,omitempty"`
    // KeyName is the data key holding the ruleset body.
    KeyName string `json:"keyName"`
}
type TykRulesetStatus struct {
    ObjectID          string          `json:"objectId,omitempty"`
    RulesetID         string          `json:"rulesetId,omitempty"`
    Owner             string          `json:"owner,omitempty"`
    OrgID             string          `json:"orgId,omitempty"`
    LatestTransaction TransactionInfo `json:"latestTransaction,omitempty"`
    LatestCRDSpecHash string          `json:"latestCRDSpecHash,omitempty"`
    // LatestSourceHash hashes the resolved body, regardless of source
    // (raw or objRef → ConfigMap/Secret).
    LatestSourceHash  string          `json:"latestSourceHash,omitempty"`
    LatestTykSpecHash string          `json:"latestTykSpecHash,omitempty"`
}
Print columns: Name, RulesetID, Active, Action, SyncStatus.

Categories: tyk, governance. Short name: tykrs.

Validation strategy — CEL, no precedence
The "exactly one of raw / objRef" constraint is enforced at admission time via the CEL rule on RulesetSource (above). Rationale:

CEL over a validating admission webhook. Same outcome (CR is rejected before it lands in etcd, reconciler never sees it), but no extra service, no cert rotation, no webhook-down failure mode. tyk-operator-internal does have webhook precedent (apidefinition_webhook.go); use a webhook only if a future rule needs cluster-side lookups, which the source-oneOf rule doesn't.

No reconciler-side precedence. Because admission rejects multi-source and zero-source CRs, the reconciler treats the "both set" / "neither set" states as impossible-by-construction. If it ever encounters one (state corruption, schema regression, direct API write bypassing validation), it must fail loud — Status.LatestTransaction.Status = "Failed" with an explicit "invalid source — admission validation should have caught this" message — and must not silently prefer one source over the other. Silent precedence (objRef > raw or vice-versa) turns an invalid spec into tolerated behaviour and hides the real bug.

Layered fallback is a different design, not in scope. If we ever want "raw is a default and objRef overrides it", that ships as a separate proposal with its own field shape — it's not the same feature as oneOf.

Wiring constants
api/v1alpha1/kinds.go — add KindTykRuleset = "TykRuleset".

pkg/keys/keys.go — add TykRulesetFinalizerName = "finalizers.tyk.io/tykruleset".

Universal interface — pkg/client/universal/ruleset.go (new file)


type Ruleset interface {
    Create(ctx context.Context, meta RulesetMetadata, body string) (RulesetIdentity, error)
    Get(ctx context.Context, id string) (RulesetResponse, error)
    Update(ctx context.Context, id string, meta RulesetMetadata, body string) error
    Delete(ctx context.Context, id string) error
    Exists(ctx context.Context, id string) bool
}
Add Ruleset() Ruleset to pkg/client/universal/universal_client.go.

Dashboard client — pkg/client/dashboard/ruleset.go (new file)
Source: copy-paste structure from pkg/client/dashboard/mcp.go. Endpoint /api/rulesets.

Method

HTTP

Notes

Create 

POST /api/rulesets 

multipart/form-data with metadata + ruleset parts. Caller supplies the resolved body string regardless of source. Read Meta from APIResponse and a follow-up GET to learn the stable ruleset_id. Return both. 

Get 

GET /api/rulesets/{id} 

Accepts ObjectID hex or RulesetID string. 

Update 

PUT /api/rulesets/{id} 

multipart/form-data. 

Delete 

DELETE /api/rulesets/{id} 

Soft-delete server-side. 

Exists 

GET probe 

Treat 404 as not-exists. 

Register Ruleset() on dashboard.Client in pkg/client/dashboard/client.go.

Gateway client — pkg/client/gateway/ruleset.go (new file)
Stubs that return ErrNotSupported. Rulesets are a Dashboard-only feature.

Register Ruleset() on gateway.Client in pkg/client/gateway/client.go.

Mock — pkg/client/mock/ruleset.go (new file)


//go:generate mockgen -destination=./ruleset.go -package mock . Ruleset
Register MockRuleset on the mock client in pkg/client/mock/mock.go.

Klient pass-through — extend pkg/client/klient/universal_client.go
Add Ruleset() and Universal-level helpers, identical pattern to MCP().

Reconciler — internal/controller/tykruleset_controller.go (new file)
Source: copy-paste skeleton from internal/controller/tykmcpproxydefinition_controller.go.



// +kubebuilder:rbac:groups=tyk.tyk.io,resources=tykrulesets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=tyk.tyk.io,resources=tykrulesets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=tyk.tyk.io,resources=tykrulesets/finalizers,verbs=update
// +kubebuilder:rbac:groups="",resources=configmaps,verbs=get;list;watch
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch
Reconcile flow:

Fetch CR; ignore not-found.

Resolve OperatorContext via HttpContext helper.

If DeletionTimestamp set → reconcileDelete.

If finalizer missing → add and requeue.

Resolve the body via resolveRulesetSource(ctx, cr.Spec.Ruleset):

If Raw is set → use it directly.

If ObjRef is set → fetch the referenced object (ConfigMap or Secret) in ObjRef.Namespace (default: CR namespace), read data[KeyName]. For Secret, decode base64. Return error → LatestTransaction.Failed.

If status.RulesetID == "" or !klient.Universal.Ruleset().Exists(...) → Create with the resolved body → store ObjectID, RulesetID, OrgID, Owner from response.

Else → compute hashes (spec, source, tyk). If unchanged, skip. Else Update with the resolved body.

Update status with hashes (LatestCRDSpecHash, LatestSourceHash, LatestTykSpecHash) and LatestTransaction.

reconcileDelete:

If finalizer present → klient.Universal.Ruleset().Delete(ctx, status.RulesetID); ignore IsNotFound.

Remove finalizer and Update.

(Future, Epic 2: refuse delete if Status.LinkedByEvaluations is non-empty, mirroring MCP's LinkedByPolicies guard.)

Source watches:

Two field indexes — one per supported object kind — both keyed off spec.ruleset.objRef. The mapping function emits the indexed value only when ObjRef.Kind matches; CRs using ruleset.raw index to nothing and aren't woken by ConfigMap/Secret churn.



const (
    rulesetToConfigmapIdx = "spec.ruleset.objRef.configmap"
    rulesetToSecretIdx    = "spec.ruleset.objRef.secret"
)
Watch both &v1.ConfigMap{} and &v1.Secret{} with mapping functions findRulesetsFromCmRefs / findRulesetsFromSecretRefs, identical pattern to PR #172's MCP/ConfigMap wiring.

Main wiring — cmd/main.go
Register TykRulesetReconciler{Client, Scheme, Env} on startup, immediately after the MCP block.

Generated artefacts (run make manifests / make helm)
config/crd/bases/tyk.tyk.io_tykrulesets.yaml

Add to config/crd/kustomization.yaml

config/rbac/role.yaml

helm/crds/crds.yaml, helm/templates/all.yaml

Append a Ruleset kind entry to PROJECT (kubebuilder scaffolding metadata)

Samples — config/samples/ruleset/ (new directory)
File

Purpose

00-tykruleset-raw.yaml 

TykRuleset with inline ruleset.raw body. 

01-ruleset-configmap.yaml 

ConfigMap holding a Spectral body. 

02-tykruleset-configmap.yaml 

TykRuleset using ruleset.objRef → ConfigMap. 

03-ruleset-secret.yaml 

Secret holding a Spectral body. 

04-tykruleset-secret.yaml 

TykRuleset using ruleset.objRef → Secret. 

05-tykruleset-with-context.yaml 

Demonstrates contextRef. 

06-ruleset-configmap-updated.yaml 

Drift scenario for tests. 

07-tykruleset-categories.yaml 

Shows categories array. 

08-tykruleset-invalid-source.yaml 

Both raw and objRef set — expected to be rejected by CEL validation. 

Docs — docs/development.md
Add a "Governance Rulesets" quickstart section: required Dashboard permissions, the four-step apply sequence, and how to verify with kubectl get tykruleset and the Dashboard UI.

Identity & idempotency notes
The Dashboard returns ObjectID (Meta) on create and exposes ruleset_id on a follow-up GET. Operator stores both, but uses RulesetID for all subsequent calls so the binding survives an analytics-side migration.

If a Dashboard admin manually deletes a ruleset, the next reconcile sees Exists == false and recreates it with a new RulesetID. Status is rewritten. This is the same drift behaviour as TykOasApiDefinition.

OrgID and Owner are observed from the response, never set from the spec.

Phasing
Phase

Scope

A 

Types + Dashboard client + mocks (no controller). Unit-tested. 

B 

Read-only reconciler (Get only, no mutation). Confirms wiring and status reporting. 

C 

Create / Update + ConfigMap watch + hash-based skip. 

D 

Delete + finalizer. 

E (Epic 2) 

LinkedByEvaluations and dependency-blocking deletion. 

Build hygiene: after type changes run make manifests, make helm, gofmt -s -w ., go vet ./..., golangci-lint run.

Acceptance criteria
[ ] Applying a TykRuleset with ruleset.raw produces a ruleset on the Dashboard whose Definition matches the inline body byte-for-byte.

[ ] Applying a TykRuleset with ruleset.objRef → ConfigMap produces a ruleset whose Definition matches data[keyName].

[ ] Applying a TykRuleset with ruleset.objRef → Secret produces a ruleset whose Definition matches the base64-decoded data[keyName].

[ ] Submitting a CR with both raw and objRef set, or with neither, is rejected by CEL validation at admission time.

[ ] Submitting a CR with objRef.Kind other than ConfigMap / Secret is rejected by the Enum validator.

[ ] All other metadata fields (name, description, active, action, resource_type, categories) round-trip correctly.

[ ] Status.ObjectID and Status.RulesetID are populated after the first successful reconcile.

[ ] Status.Owner and Status.OrgID are read back from the Dashboard response, never taken from the spec.

[ ] Editing the inline ruleset.raw triggers a PUT /api/rulesets/{id} that updates the Tyk-side body; LatestSourceHash and LatestCRDSpecHash advance.

[ ] Editing the referenced ConfigMap or Secret triggers a PUT; LatestSourceHash advances; LatestCRDSpecHash is unchanged.

[ ] Editing only spec metadata (e.g. categories or active) triggers a PUT with new metadata; LatestCRDSpecHash advances; body unchanged.

[ ] No PUT is issued when all three hashes (spec, source, tyk) are equal.

[ ] Switching a CR from raw → objRef (or vice-versa) without changing the resolved body content triggers no PUT (LatestSourceHash stable).

[ ] kubectl delete tykruleset/<name> issues a DELETE /api/rulesets/{id}, then removes the finalizer.

[ ] If the Dashboard returns 404 on delete, the finalizer is still removed (IsNotFound is treated as success).

[ ] If the Dashboard ruleset is deleted out-of-band, the next reconcile recreates it and writes the new RulesetID into status.

[ ] If the referenced ConfigMap/Secret is missing, Status.LatestTransaction.Status = "Failed" and reconcile retries on object creation (watches fire).

[ ] Dashboard 4xx/5xx errors surface as Status.LatestTransaction.Status = "Failed" with the error message in Status.LatestTransaction.Error.

[ ] action defaults to none and resource_type defaults to oas-api when not set in the CR; invalid values are rejected by the kubebuilder Enum validation.

[ ] Existing CRs continue to reconcile correctly after operator restart (no in-memory state).

[ ] Re-creating an OperatorContext with the same Dashboard endpoint does not duplicate the ruleset.

[ ] CHANGELOG.md updated.

[ ] make manifests, make helm, golangci-lint run, go vet ./..., gofmt -s -w . all clean.

[ ] docs/development.md contains a Governance Rulesets quickstart covering both source modes.

[ ] Sample manifests 00–08 apply (or fail in the expected way for 08) against a Kind cluster.

Testing
Unit tests
Reconciler — internal/controller/tykruleset_controller_test.go (new file)
Source: copy-paste structure from internal/controller/tykmcpproxydefinition_controller_test.go.

TestTykRulesetReconciler_Reconcile:

Sub-test

Setup

Expected

CR not found 

Get returns NotFound 

result {}, err nil 

context resolution fails 

HttpContext returns error 

result {}, error propagated 

finalizer added 

CR has no finalizer 

requeue 1s, finalizer present on CR 

create flow — raw 

inline body, klient.Exists = false 

Ruleset.Create called with the inline string; status fields and hashes set; LatestTransaction Successful 

create flow — objRef ConfigMap 

ConfigMap present 

Ruleset.Create called with data[keyName]; same status updates 

create flow — objRef Secret 

Secret present 

Ruleset.Create called with base64-decoded data[keyName] 

update flow — spec drift 

LatestCRDSpecHash differs 

Ruleset.Update called 

update flow — source drift (raw) 

inline body changes 

Ruleset.Update called; LatestSourceHash advances 

update flow — source drift (configmap) 

ConfigMap edited 

Ruleset.Update called; LatestSourceHash advances 

update flow — source drift (secret) 

Secret edited 

Ruleset.Update called; LatestSourceHash advances 

update flow — tyk drift 

LatestTykSpecHash differs 

Ruleset.Update called 

source mode swap, content equal 

switch raw → objRef with identical bytes 

LatestSourceHash unchanged; Ruleset.Update NOT called 

no-op 

all three hashes equal 

Ruleset.Update NOT called; status unchanged 

dashboard error 

Ruleset.Create returns error 

LatestTransaction Failed; error string set 

objRef missing — ConfigMap 

Get on ConfigMap returns NotFound 

LatestTransaction Failed 

objRef missing — Secret 

Get on Secret returns NotFound 

LatestTransaction Failed 

objRef key missing 

object exists but data[keyName] absent 

LatestTransaction Failed 

objRef invalid kind 

Kind = "Pod" (bypassing CEL via test fixture) 

reconcile returns error; never calls Create/Update 

TestTykRulesetReconciler_ReconcileDelete:

Sub-test

Setup

Expected

happy path 

DeletionTimestamp set, finalizer present, Exists = true 

Ruleset.Delete called; finalizer removed 

remote already gone 

Exists = false 

finalizer removed; no Delete call 

dashboard 404 on delete 

Delete returns IsNotFound 

finalizer removed; treated as success 

dashboard error 

Delete returns 500 

error propagated; finalizer kept 

TestTykRulesetReconciler_findRulesetsFromCmRefs: ConfigMap event maps to all CRs whose spec.ruleset.objRef points at it (Kind ConfigMap); CRs using raw or pointing at a Secret are not woken.

TestTykRulesetReconciler_findRulesetsFromSecretRefs: symmetric for Secrets; ConfigMap-referencing CRs are not woken.

TestRulesetSource_CELValidation (envtest with the CRD installed):

Sub-test

Spec

Expected

both unset 

no raw, no objRef 

apply rejected with CEL message 

both set 

raw and objRef set 

apply rejected with CEL message 

only raw 

inline body 

apply accepted 

only objRef ConfigMap 

Kind: ConfigMap 

apply accepted 

only objRef Secret 

Kind: Secret 

apply accepted 

objRef bad kind 

Kind: Pod 

apply rejected by Enum validator 

Dashboard client — pkg/client/dashboard/ruleset_test.go (new file)
Sub-test

HTTP fixture

Expected

Create success 

201 with Meta = "abc123" 

ObjectID = "abc123"; follow-up GET returns ruleset_id; both returned 

Create dashboard 409 

response 409 

ErrAlreadyExists 

Create network error 

transport error 

error propagated 

Get by ObjectID 

200 

ruleset returned 

Get not found 

404 

ErrNotFound 

Update success 

200 

no error 

Update 409 

response 409 

ErrNameExists 

Delete success 

200 

no error 

Delete 404 

response 404 

IsNotFound true on returned error 

Exists 200/404 

probe responses 

true / false respectively 

Mock
Generated; smoke-test that the gomock interface compiles.

Integration tests
integration/tykruleset_test.go (new file)
Source: copy-paste structure from integration/tykmcpproxydefinition_test.go.

Suite covers a real Kind cluster + Dashboard:

create flow — raw — apply TykRuleset with inline ruleset.raw; assert ruleset visible via /api/rulesets and Status fields populated.

create flow — ConfigMap — apply ConfigMap then TykRuleset; assert Tyk-side Definition matches data[keyName].

create flow — Secret — apply Secret then TykRuleset; assert Tyk-side Definition matches the base64-decoded value.

update via raw — edit inline body in CR; reconcile produces matching Definition on the Dashboard.

update via ConfigMap — edit ConfigMap; assert Tyk-side body matches new content.

update via Secret — edit Secret; assert Tyk-side body matches new content.

update via spec — edit categories; assert PUT issued; new categories visible on Dashboard; body unchanged.

source mode swap — change a CR from raw → objRef (ConfigMap with identical bytes); assert no PUT and no churn in Status.

idempotency — re-apply unchanged CR; assert no PUT (verified via Dashboard audit log or hash equality).

CEL validation — kubectl apply on a manifest with both raw and objRef set is rejected by the API server.

missing source — apply TykRuleset before its referenced ConfigMap/Secret exists; assert LatestTransaction.Failed; create the object; assert next reconcile succeeds (watches wake).

delete flow — delete CR; assert ruleset is soft-deleted on the Dashboard and finalizer is gone.

drift recovery — manually delete on Dashboard; assert the next reconcile recreates the ruleset.

operator restart — kill the manager pod mid-edit; assert the next reconcile catches up.

context override — apply two CRs with different contextRef; assert each lands in the right Dashboard.

CI
Add the new integration test to the standard run_tests.yaml job.

If the analytics endpoints aren't yet in a tagged Dashboard release, gate the test behind the test-master CI job (the same pattern PR #172 used for unreleased MCP support).

Lint / generate
make manifests and make helm produce clean diffs (no untracked changes after run).

golangci-lint run, go vet ./..., gofmt -s -w . pass.

Manual verification (smoke)
Apply samples 00 ->04 in order against a Kind + Tyk Dashboard environment. After each step, kubectl get tykruleset should reflect the expected status, and the Dashboard UI should show the ruleset with the correct fields.


## Epic 2: Ruleset Evaluation (The Execution)

Epic Name


Ruleset Evaluation (The Execution). Governance in Dashboard
Description

Goal
Provide the execution engine and CI/CD tooling to evaluate APIs against governance rulesets.

Description
This epic covers applying and evaluating rulesets. It focusses healivy on integrating the Vacuum engine and building the CI/CD bridge so developers can run their APIs against rulesets in UI and pipelines

 

Figma Designs: Governance Dashboard – Figma 

Features
Rule Execution Engine: Execute rulesets using the Vacuum engine

Ruleset Testing: A sandbox environment to test rulesets against API definitions and view execution results before saving. This needs to be configured in the Ruleset screen and in the API screen, both in the UI and CI/CD

Category-based Execution: Run rulesets against all APIs in a specific category in the UI and in CI/CD pipelines. Provide

Rule Execution & Feedback: Execute rulesets using the Vacuum engine. Output uses Severity only (Error / Warn / Info) — Errors make the API non-compliant; Warn and Info are advisory.

Severity & Priority Mapping: Map rules to Severity (Error / Warn / Info). No separate "violation" concept — what was previously called a violation is now a rule with severity Error.

Remediation Guidance: Detailed issue views providing remediation guidance for specific rule violations

Define custom remediation guidance per rule within the ruleset definition

Display “Issue Details” with actionable guidance when a rule violation occurs

 

Backend Stories
Category-based Evaluation job: Async background job to evaluate a ruleset against all APIs within its assigned categories. Async job

Ability to test rulesets per API in the dashboard and in the CI/CD. An endpoint that can post an API spec and a rulesetId, to run Vacuum and return the JSON results.

Remediation Priority: Assign each rule a remediation priority. Configure this. 

Implement "Error/Warn/Info" severity. 

 

Front End Stories
Ruleset Test Sandbox: UI to select an API and run a ruleset against it, and view the results. Both in the Ruleset screen and Governance Tab per API section.

 

Open Questions and Doubts
We have rule severity (error/warn/info) when you configure a rule.

Performance Analysis when you run ruleset into APIs

The current idea is not to block customers on their deployments, just a warning. TBR after customer interviews. 
We could allow them to decide if they want to block or not.

#### TT-17075 Async evaluation

#### TT-17073 Evaluation API

#### TT-17096 UI: Governance tab in API details page

Summary
Add a "Governance" tab to the existing API details page that surfaces the per-API compliance posture: an overall governance status banner, a paginated grid of associated rulesets with per-ruleset compliance scores, a filterable issues table listing individual violations and warnings, and a utility section to trigger a manual evaluation run.

Context
Rulesets define governance rules (e.g. "all APIs must require authentication"). A ruleset becomes associated with an API when the ruleset's categories field matches one of the API's category tags. Once associated, the ruleset's rules are evaluated against that API's definition and the results determine its compliance status.

This tab surfaces those evaluation results directly inside the API details page, allowing API owners to understand their compliance posture and act on violations without navigating away from their API.

The tab is brand new — it does not replace any existing tab.

Current Behaviour
The API details page has the following tabs: Settings, Endpoints, Access, Versions. No governance or compliance information is shown anywhere on the API details page.

Expected Behaviour
Route / Tab placement
The "Governance" tab is appended as the last tab on the API details page. It carries a "New!" pill badge to draw attention to the new capability.

The tab does not introduce a new URL, it follows the existing tab-switching pattern of the API details page.

Page layout
The tab renders four sections vertically, in order:

Governance Status Banner

Rulesets associated with this API

Issues for this API

Test ruleset

1. Governance Status Banner
A full-width bordered card rendered at the very top of the tab.

Element

Description

Status icon 

An orange/red warning shield icon. Colour changes: orange/amber for warnings-only, red for any violations 

"Governance status" label 

Smaller muted text: static string "Governance status" 

Status text 

Bold large text: "Compliant" (when zero violations across all associated rulesets) or "Non-compliant" (when any violation exists) 

Warnings count 

Yellow triangle icon (⚠) + large bold number + "Warnings" label — sum of all warnings across all associated rulesets for this API 

Errors count 

Red alert triangle icon + large bold number + "Errors" label — sum of all violations across all associated rulesets for this API 

The status and counts are derived entirely from the backend response and are never computed on the frontend.

Screenshot 2026-04-29 at 11.06.05 AM.png
2. Rulesets associated with this API
Section heading: Rulesets associated with this API

Section sub-text: "Check the compliance status of your applied rulesets, review any warnings, and take action to resolve issues."

A 3-column responsive card grid, each card representing one ruleset that has been associated with this API through category matching. The grid is paginated at 6 cards per page.

Ruleset Compliance Card anatomy
Element

Value / notes

Status badge (top-left) 

✅ Compliant (green icon + green text) or ❌ Non-compliant (red icon + red text). Non-compliant = at least one violation in this ruleset for this API 

Ruleset name 

Bold text. Clickable link — navigates to /governance/rulesets/:ruleset_id 

Violations count 

Prohibition circle icon (⊘) + count, e.g. ⊘ 13 violations. Zero is shown as ⊘ 0 violations 

Warnings count 

Warning triangle icon (⚠) + count, e.g. ⚠ 4 warnings. Zero is shown as ⚠ 0 warnings 

Compliance level label 

Right-aligned text below the counts: Compliance level 92/100 

Progress bar 

Horizontal bar below the label, fill width = compliance_level / 100. Compliant cards: dark/full-looking bar; non-compliant cards: shorter / lighter bar to indicate lower effective score 

The compliance level is a 0–100 score returned by the API endpoint — it is never computed on the frontend

Standard pagination control rendered below the card grid, right-aligned.

Active page is highlighted. Page size is 6. pages count comes from the backend response.

 

Screenshot 2026-04-29 at 11.06.43 AM.png
3. Issues for this API
Section heading: Issues for this API

Section sub-text: "Review governance violations for this API. Resolve issues to prevent security, compliance, and operational issues."

Filter bar
The filter bar is split into two groups, left and right:

Left — Severity filter pills (tab-style):

Pill

State when active

All issues (N) 

Default selected; shows all rows regardless of severity 

Violations (N) 

Filters to severity: "violation" rows only 

Warnings (N) 

Filters to severity: "warning" rows only 

The (N) counts are derived from the backend response and update when filters change.

Right — Dropdown filters:

Control

Default

Behaviour

Filter by ruleset label + dropdown 

All — no filter applied (dropdown shows all associated rulesets as options) 

Selecting a specific ruleset shows only issues produced by that ruleset 

Remediation priority label + dropdown 

All — no filter applied 

Selecting High, Medium, or Low filters rows by the issue's priority field 

Changing any filter parameter sends a new request to GET /apis/:apiId/governance with the updated query params — filtering is server-side.

Issues table
Six columns, in this order:

#

Column header

Content

1 

Ruleset 

Name of the ruleset that produced the issue (plain text) 

2 

Rule violated 

Rule identifier (e.g. authentication_required, enforce_https). Rendered as an underlined clickable link — see Gap #4 for what "View issue" / link target does 

3 

Severity 

Badge — ⊘ Violation (red badge) or ⚠ Warning (yellow badge). The column header carries a tooltip icon (ⓘ); hovering shows: "Violations make the ruleset non-compliant and must be fixed. Warnings are advisory and don't affect compliance status." 

4 

Remediation priority 

Priority badge — ⚠ High (amber icon + text), ⚠ Medium, or ⚠ Low. Icon and label only, no colour difference on the icon per the design 

5 

Problem if not fixed 

Short human-readable description of the risk if this issue is not resolved (e.g. "Unauthorized users can create fraudulent orders."). May be truncated with ellipsis if too long 

6 

Actions 

🔍 View issue button (icon + label). See Gap #4 

The table is paginated at 10 rows per page.

Screenshot 2026-04-29 at 11.09.25 AM.png



4. Test ruleset
Full expected flow TBD

Section heading: Test ruleset

Section sub-text: "Select an API and run the ruleset to see the results."

On evaluation success:

All three sections above (banner, rulesets grid, issues table) refresh by re-fetching GET /apis/:apiId/governance

A toast notification confirms: "Ruleset evaluation completed"

On evaluation error:

An error toast is shown; all existing data on the page remains unchanged

Note on dropdown pre-selection: When the tab loads on the API details page, the "Select API" dropdown defaults to the current API (the one whose details page the user is viewing). Changing the selection and clicking "Run ruleset" evaluates that other API's rulesets — the result sections will update to show that other API's compliance data. This is intentional: the section acts as a standalone evaluation utility, not locked to the current page's API.

Backend Requirements
Endpoint 1 — GET /apis/:apiId/governance
Fetches the full governance compliance summary for a given API. Called on tab load and after every evaluation run.

Path parameter:

Param

Description

apiId 

The API's api_id string (same as used in all other /apis/:apiId endpoints) 

Response 200 OK:



{
  "api_id": "payment-api",
  "status": "non-compliant",
  "total_warnings": 10,
  "total_errors": 5,
  "rulesets": {
    "items": [
      {
        "ruleset_id": "a1b2c3",
        "name": "Secure Access Standards",
        "status": "compliant",
        "violations": 0,
        "warnings": 4,
        "compliance_level": 92
      },
      {
        "ruleset_id": "d4e5f6",
        "name": "Traffic Control Guidelines",
        "status": "non-compliant",
        "violations": 13,
        "warnings": 4,
        "compliance_level": 55
      }
    ],
  },
  "issues": {
    "items": [
      {
        "ruleset_id": "a1b2c3",
        "ruleset_name": "API Authentication enforcement",
        "rule_id": "rule-001",
        "rule_name": "authentication_required",
        "severity": "violation",
        "priority": "high",
        "problem_if_not_fixed": "Unauthorized users can create fraudulent orders."
      }
    ],
    "total": 13,
    "violations": 8,
    "warnings": 4,
  }
}
Field-by-field UI mapping:

Error responses: 401, 403, 404 (API not found), 500

Endpoint 2 — POST /apis/:apiId/governance/evaluate
Triggers a governance evaluation run for the given API against all of its associated rulesets. Called when the user clicks "Run ruleset" in Section 4.

Path parameter:

Param

Description

apiId 

The API's api_id. Note: this is the API selected in the "Select API" dropdown in Section 4, which may differ from the page's original API 

Response 200 OK:



{
  "Status": "success",
  "Message": "evaluation completed"
}
On receiving a 200 success, the UI immediately re-fetches GET /apis/:apiId/governance (with the same :apiId that was evaluated) to refresh all three display sections.

Error responses: 401, 403, 404 (API not found), 500

Permissions: Requires read or above on the governance permission group.

Test Cases
Test Case 1: Non-compliant API renders correctly
GET /apis/:apiId/governance returns status: "non-compliant" with total_errors: 5, total_warnings: 10

Expected: Banner shows red shield + "Non-compliant" + "10 Warnings" + "5 Errors". Affected ruleset cards show ❌ badge. Issues table populated with correct rows

Test Case 2: Fully compliant API renders correctly
GET /apis/:apiId/governance returns status: "compliant", total_errors: 0, total_warnings: 0

Expected: Banner shows green shield + "Compliant". All ruleset cards show ✅. Issues table shows empty state

Test Case 3: Severity filter tab filters the table
User clicks "Violations (8)" pill

Expected: Table filters out only violations

Test Case 4: "Filter by ruleset" dropdown filters the table
User selects "API Authentication enforcement" from the ruleset dropdown

Expected: Issues table re-rendered with ruleset_id=<id>; only issues from that ruleset shown; pill counts update accordingly

Test Case 5: "Remediation priority" dropdown filters the table
User selects "High" from the remediation priority dropdown

Expected: Issues table re-rendered with priority=high; only high-priority rows shown

Test Case 6: Rulesets grid pagination works
rulesets.pages = 2; user clicks page 02

Expected: Re-fetches with rulesets_page=2; second set of ruleset cards rendered; pagination control shows 02 active

Test Case 7: "Run ruleset" triggers evaluation and refreshes all sections
User clicks "Run ruleset"

Expected: POST /apis/:apiId/governance/evaluate called; button shows loading state; on 200 success, GET /apis/:apiId/governance re-fetched; all three display sections update; success toast shown

Test Case 8: No rulesets associated with the API
GET /apis/:apiId/governance returns status: "no-rulesets"

Expected: Banner is hidden or shows a neutral state; rulesets grid shows empty state: "No rulesets are associated with this API. Link a ruleset's category to this API's category tags to begin governance evaluation."; issues table shows empty state

Acceptance Criteria
[ ] "Governance" tab is appended as the last tab on the API details page, with a "New!" badge

[ ] Tab calls GET /apis/:apiId/governance on mount

[ ] Governance status banner shows correct status label, icon colour, warning count ("X Warnings"), and error count ("X Errors")

[ ] Rulesets card grid renders cards with correct status badge (✅/❌), violations count, warnings count, compliance level label, and progress bar

[ ] Issues table renders 6 columns: Ruleset, Rule violated (as link), Severity (badge), Remediation priority (badge), Problem if not fixed, Actions ("View issue" button)

[ ] Severity column header has a tooltip explaining Violation vs Warning

[ ] "All issues / Violations / Warnings" filter tabs update the table

[ ] "Filter by ruleset" dropdown populates from ruleset_name values in issues.items; selecting filters the table (server-side via ruleset_id param)

[ ] "Remediation priority" dropdown filters the table (server-side via priority param)

[ ] "Run ruleset" calls POST /apis/:apiId/governance/evaluate

[ ] Button shows a loading/spinner state during evaluation

[ ] After evaluation success: all display sections re-fetch and update; success toast shown

[ ] After evaluation error: error toast shown; page data unchanged

[ ] status: "no-rulesets" shows appropriate empty state with instructional copy

[ ] All API calls are authenticated and org-scoped

[ ] Permissions: requires read or above on the governance permission group

Out of Scope
Editing which rulesets are associated with an API 

Running evaluation automatically on every API save (infrastructure concern)

Historical compliance data / audit trail of past evaluation runs

Inline rule editing directly from the issues table

Bulk-resolving issues or marking issues as acknowledged

TBD
UI / UX for view issue click.

Do we need API level pagination here, if we’re not expecting heavy data we can do pagination on FE.

Test ruleset section on this page is confusing, maybe it should just say re-run ?

Empty state if no ruleset configured 

#### TT-17097 UI: Ruleset runner in ruleset details page

#### TT-17098 Compliance stats on API listing page


## Epic 3: Governance Ruleset Results & Visibility
Epic Name


Governance Ruleset Results & Visibility. Governace In Dashboard
Description

Goal
Surface governance evaluation results, track compliance, and enforce standards before deployment.

Description
This epic covers what you get from the evaluation. It handles receiving and storing linting feedback (whether generated externally by CI/CD pipelines or internally by Dashboard jobs), displaying them in the new Governance tab per API, and warning users if they try to deploy non-compliant APIs.

 

Figma Designs: Governance Dashboard – Figma

Features
Per-API Governance Status: View the overall compliance status (Compliant / Non-Compliant) for a specific API.

Applied Rulesets & Issues: List the rulesets evaluated against the API and detail the specific issues by severity (Error / Warn / Info).

CI/CD Status Retrieval: API endpoints to retrieve the governance status and issues of an API

Issue Details: Configuration on how to fix (Remediation guidance) an API through the API goverannce tab and through the Test ruleset functionality in the Ruleset page.

 

Backend Stories
Endpoint to get the governance status and issues of a specific API. (Complaint/Non-Compliant)

Endpoint to automatically send the final linting feedback to the Dashboard after API evaluation

BE logic to map rule issues (Error / Warn / Info) to their specific “How to fix”

Any other work needed for the new Governance tab per API section [UI Warnings].

 

Frontend Stories
Implement the new “Governance” tab in the API configuration view, displaying the compliance status, rulesets, and detailed issues

Display Issue Details, Affected area, and “How to fix” (Remediation Guidance) sorted by Remediation Priority

UI warnings when saving or changing an API to active if it's non-compliant

Show an “In progress” indicator when a ruleset is currently running on an API

Open Questions & Doubts
How do we calculate the compliance score? 

What makes an API Non-Compliant? A rule failing at severity Error makes the API non-compliant. Warn and Info do not affect compliance status.

How does the “Warning” visual date happen in CI/CD to have that extra step before you move an API to active if non-compliant?

 

Further Ideas
Customers (NorthWestern Mutual & Barclays) would like RBAC to be expanded. A read-only shared link of the dashboard for Federated teams. Something like a snapshot you can share.