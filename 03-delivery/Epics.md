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



#### TT-17020 API for Read Rulesets
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
GetByID  accepts MongoDB ObjectId hex or stable RulesetID string:



func (r *rulesetRepository) GetByID(_ context.Context, orgID, id string) (*model.Ruleset, error) {
    var ruleset model.Ruleset
    var filter model.DBM
    if model.IsObjectIdHex(id) {
        filter = model.DBM{"_id": model.ObjectIdHex(id), "org_id": orgID, "deleted_at": model.DBM{"$eq": nil}}
    } else {
        filter = model.DBM{"ruleset_id": id, "org_id": orgID, "deleted_at": model.DBM{"$eq": nil}}
    }
    if err := r.storage.Query(filter, &ruleset); err != nil {
        return nil, ErrNotFound
    }
    return &ruleset, nil
}
List  all matching, handler paginates:



go
var allowedSortFields = map[string]string{
    "name":         "name",
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

Extract id from mux.Vars(r)  (accepts ObjectId hex or RulesetID string)

Call svc.Get(ctx, orgID, id) if ErrNotFound → 404

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

name 

string 

case-insensitive regex match 

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

[ ] Accepts both MongoDB ObjectId hex and RulesetID string in path

[ ] Returns 404 if not found or belongs to a different org

[ ] Soft-deleted rulesets are never returned by Get or List

[ ] GET /api/rulesets returns paginated list

[ ] List supports ?name= (regex), ?active=, ?is_template=, ?categories=, ?resource_type= filters

[ ] No audit diff written for read operations

[ ] Returns 401 unauthenticated, 403 for deny on GovernanceGrp

[ ] swagger.yml updated with GET list and GET single paths + RulesetList schema

[ ] All API tests pass

[ ] All unit tests pass

Testing
API tests —> add to tests/api/tests/dashboard_api/rulesets_test.py

API test for get rulesets
Unit tests
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

valid hex ID 

200 

success by ruleset id 

non-hex string ID 

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

?name=foo passed through to repo 

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

success by object id 

Query with _id filter returns ruleset 

ruleset returned 

success by ruleset id 

Query with ruleset_id filter returns ruleset 

ruleset returned 

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

error propagated 


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

#### TT-17025 UI for API Ruleset view, edit, delete

#### TT-17026 BE - Vacuum update
Context and preconditions
Context preceding the task and a problem to solve: any researches, feedback from customers, etc.
Should be filled by the reporter.
In the original governance PoC, we forked the Vacuum lib (the lib used for ruleset linting and analysis) GitHub - buraksekili/vacuum: vacuum is the worlds fastest OpenAPI 3, OpenAPI 2 / Swagger linter and quality analysis tool. Built in go, it tears through API specs faster than you can think. vacuum is compatible with Spectral rulesets and generates compatible reports. because of panic.
The lib has +50 releases from back then ( from v0.17.14 to v0.26.1). We need to understand what has changed and its performance implications. 

Product idea
Proposed solution from a product perspective, i.e. wireframes, Figma prototypes and etc.
Should be filled by the reporter.
Analyze if we still need to fork this library and do a performance benchmark comparison of the old version vs the new, so we can have an overall idea of how long it takes to analyze OAS APIs with rulesets.

TT-17041 Ruleset Model & RBAC Setup

TT-17054 Investigate and implement how the backend will parse MCP for ruleset configuration


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

Rule Execution & Feedback: Execute rulesets using the Vacuum engine. Provide violations, warnings.

Severity & Priority Mapping: Map rules to Severity (error/warn/info)

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

Applied Rulesets & Issues: List the rulesets evaluated against the API and detail the specific violations and warnings.

CI/CD Status Retrieval: API endpoints to retrieve the governance status and issues of an API

Issue Details: Configuration on how to fix (Remediation guidance) an API through the API goverannce tab and through the Test ruleset functionality in the Ruleset page.

 

Backend Stories
Endpoint to get the governance status and issues of a specific API. (Complaint/Non-Compliant)

Endpoint to automatically send the final linting feedback to the Dashboard after API evaluation

BE logic to map rule violations/warnings to their specific “How to fix”

Any other work needed for the new Governance tab per API section [UI Warnings].

 

Frontend Stories
Implement the new “Governance” tab in the API configuration view, displaying the compliance status, rulesets, and detailed issues

Display Issue Details, Affected area, and “How to fix” (Remediation Guidance) sorted by Remediation Priority

UI warnings when saving or changing an API to active if it's non-compliant

Show an “In progress” indicator when a ruleset is currently running on an API

Open Questions & Doubts
How do we calculate the compliance score? 

What makes an API Non-Compliant?. An error [A rule is not met] makes APIs non-compliant.

How does the “Warning” visual date happen in CI/CD to have that extra step before you move an API to active if non-compliant?

 

Further Ideas
Customers (NorthWestern Mutual & Barclays) would like RBAC to be expanded. A read-only shared link of the dashboard for Federated teams. Something like a snapshot you can share.