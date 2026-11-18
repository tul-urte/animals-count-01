# plan-materializer.md — THE MODEL MATERIALIZER (the standing plan)

**Status**: ASPIRATIONAL (2026-11-16, standing). The present work
(page-xml.js + pages.xml + the island, S109) is THIS project's first
concrete instance — captured here as the seed instance, not the boundary.
This document guides today's choices so they do not close tomorrow's
space; nothing else in this file is built yet.

## 1. THE PROJECT, IN ONE SENTENCE (the user's words)

A NEW PROJECT that focuses entirely on **materializing model objects
from XML** — entirely abstract; the imagebank page model (the browser
faces) is only ever a CONCRETE EXAMPLE. The model xml materializes to a
specified:

- **python object + playwright bindings** (the suites' loader, rebuilt
  as a materialization target)
- **javascript object + native DOM bindings** (the reference page's
  island render; later, bindings over page-internal surfaces)
- **whatever mappy object, given some bindings API** (any dict-like
  carrier + a binding surface the caller names)

## 2. THE SEED INSTANCE (what exists today, what taught what)

`06-deployment/imagebank/web/page-xml.js` + `pages.xml` parse the model
into an object tree with: structural attrs carrying the $-face,
PER-ELEMENT accessor keys (key=/title=), text-as-desc, slots
(param/trap/law/note/readme), me-wins includes (from=, first-listed
wins, trace), json-valued payloads, the `$`-root entry. Reference
instance two: the island-injected `pages-model.html`. The parity gate
(test_page_xml.js) held both trees deep-equal during the migration —
that gate is the materializer's own testing doctrine in miniature: A
CONVERSION IS NOT THE CONVERSION until its parity test holds.

WHAT TODAY'S INSTANCE TEACHES THE ABSTRACT DESIGN (each a law the
general project must keep):
- every materialized node is TYPED (the walks that consume the tree
  iterate plain keys and detect types; no per-consumer guessing)
- accessor keys belong to ELEMENTS (declared), not to tags — tags are
  grouping words; the same element tag may carry any key
- payload values stay STRINGS unless the vocabulary says otherwise
  (expression-first: the consumer's bindings interpret); number
  conversions are vocabulary-specific, never sniffs
- provenance is a first-class output (the include trace; a
  materialized tree can answer "which file supplied this node")
- the host supplies IO + DOM: the parser is environment-neutral
  (resolve()/xmlParse() are ITS dependency edges; CommonJS vs window is
  only the module wrapper)

## 3. THE NAMESPACE FOR SPECIALIZED OBJECTS

The model xml grows a NAMESPACE for specialized objects the parser
knows how to materialize: an element whose (namespace, local name) maps
to a registered MATERIALIZER CLASS. First candidates (the vocabulary
the user names):

- `h:lambda key="..."` — a lambda node whose text is an EXPRESSION the
  consumer EVALUATES through its bindings API
  (`<h:lambda key="exists">$self.getWebElement( $self.locators()
  ).exists()</h:lambda>`)
- `h:flow` — a STATE MACHINE node (title/args/initialState; states
  carry predicates; actions carry from/to + a body expression); the
  user's preview:
  `<h:flow title="Login" args="{'username': null, 'password': null,
  'maxWaitSeconds': 5.0}" initialState="notLoggedIn">` with
  `<state title="Not Logged In" predicate="$self.loginControl.exists()"/>`
  and `<action title="Log In" from="notLoggedIn" to="loggedIn">…$self.login(…)</action>`
- `h:notes` / annotations (the extension space the flip opened:
  notes + mini-state machines as TOP-LEVEL design, not comment debris)

THE PARSE LAW (already true in the seed instance, generalizes): the
parser PRESERVES what it does not materialize — an unregistered
namespaced element parses to its plain node (attrs → map, text →
string) and never breaks. Materializers register; unknown vocabulary
rides as data. NOTHING IS DROPPED.

## 4. THE ARGUMENT MODEL (the user's preview, formalized)

**All states and actions share the same args.** `$self` refers to the
current object that contains the flow element. The args block
(args="{'username': null, 'password': null, 'maxWaitSeconds': 5.0}") is
the flow's OWN parameter schema; predicates (state) and bodies (action)
are EXPRESSIONS over those args + `$self`.

The expression glue (the user's java/selenium precedent): an EL-like
mini-language — property paths (username), $self-calls
($self.login({…})), literals + maps in args/values. TODAY the parser
carries these VERBATIM (strings); the materializer project owns the
INTERPRETER: typed args per the declared schema, expression
evaluations at the bindings API. The expression vocabulary grows ONLY
when a real consumer needs it — never speculative.

## 5. THE MATERIALIZER CONTRACT (the abstract shape)

A materialization is: model xml → (materializer, bindings) → object
tree. The contract:

- **the parser** (one, shared): xml → the typed node tree (today's
  page-xml.js core; the registry + namespace dispatch added here —
  still ONE parser, no per-target forks)
- **the materializer** (per target): walks the typed tree, constructs
  the target's objects (python class with the model's per-node shape /
  JS object / Map), and attaches the BINDINGS: the accessor API the
  target names (locator composition + verbs for playwright; finders +
  helpers for DOM; plain maps for a mappy carrier)
- **the bindings API** (per consumer surface): TODAY's shape is
  `pw = PM.page(name, params).pw(pwPage); page.loc('a.b.c', params)` —
  the CONTRACT generalizes: a materialized model must be attachable to
  whatever the consumer's binding surface is, and the attachment is
  declared, not inherited
- **the parity doctrine**: every materializer carries its battery (the
  seed instance's 31-check shape); a bindings change that cannot hold
  its parity test is a regression

## 6. THE FLOWS: STATE MACHINES AS MODEL CITIZENS (the destination)

The h:flow sketch (section 3) is the destination's first REAL shape:
the estate's triage graph (new → assessed → claimed → published /
rejected / GT, the g-lanes) becomes MODEL-DECLARED, executable as a
state machine — the browser walks it per the flow's predicates and
actions. The QA suites then drive flows, not just read faces. What the
flows demand of the contract:

- typed states + actions (the materializer's registry handles h:flow /
  h:state / h:action)
- predicates are EXPRESSIONS over $self + shared args (the interpreter)
- transitions are route-honest (the estate's own edge table remains
  the authority; the flow VALIDATES against it, never replaces it)
- the machine's runtime (an executor) is a SEPARATE deliverable from
  the model (the model declares; the consumer executes)

## 7. WHAT GUIDES TODAY'S WORK (the plan's hand on the present)

- the imagebank page model's xml MUST stay the seed instance's shape:
  element attrs are payload + structural; keys are declared; text is
  prose; NO imagebank-specific vocabulary gets folded into the parser
  core (the parser stays generic; imagebank vocabulary — if any grows —
  registers, it does not branch)
- any NEW node vocabulary the model wants (notes, lambdas, flows)
  lands FIRST as a registered materializer in this project's terms,
  even while the materializer project is only this document
- the expression mini-language is NOT begun inside the imagebank
  parser; when a flow/lambda vocabulary arrives, its interpreter is a
  registered capability, not parser sugar
- keep the parser dual-surface (node + browser) — the island pattern
  (the model xml embedded in the reference page) is the concrete proof
  of the "embed anywhere" claim, keep it working as the instance grows

## 8. DELIVERABLES (when this project actually begins)

1. the parser promoted to a STANDALONE package (its own repo home;
   the imagebank artifact carries a pinned copy until then)
2. the registry + namespace dispatch (h: vocabulary pluggable)
3. the three target materializers (python+playwright / JS+DOM /
   mappy+any), each with its battery
4. the expression interpreter (args-typed, $self-bound), behind a
   flag until a consumer needs it live
5. the h:flow runtime (states/predicates/actions) with the
   route-honest validation
6. the schema (the user's promise: xmlns="http://imagebank/page-model",
   an XSD later) — the namespace's own document

## 9. THE NON-GOALS (as important as the goals)

- NO DSL for the estate's SQL/queues (the queries stay sql/*.json)
- NO generic workflow engine inside the imagebank server (the model
  declares; the product executes; the trigger/v-table DB machinery is
  not a materializer target)
- NO speculative vocabulary (the user's own law: we only add what we
  need and want — the h: namespace grows by registered demand)
- NO second parser (the xml parser is ONE; materializers are walk
  layers over its tree, never re-parsers)
