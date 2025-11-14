# Prompt Engineer & Context Studio

Base: `https://github.com/zeronsh/chat`

## 0. High level

Goal: Refactor `zeronsh/chat` into a focused app that:

* Lets users create and maintain **context profiles** (stored as JSON, via LLM helpers).
* Lets users run an interactive **Prompt Engineer** that:

  * uses selected context profiles
  * takes a task or goal
  * asks clarifying questions if needed
  * outputs an execution ready prompt
* Lets users store and manage **saved prompts** linked to context profiles.
* Supports sign up/login so users can store their own stuff.
* Uses **one LLM provider**, **bring your own key**, with a simple onboarding popup.
* Keeps everything self hosted and as local as possible for privacy.

You have an existing file `system_prompts_rows.csv` with three system prompts:

* `json_generator`
* `json_updater`
* `prompt_generator`

Those must be wired into the app as the three core LLM roles.

Base repo provides: chat UI, LLM streaming, database and infra. Your job is to strip what is not needed and rebuild the domain around context profiles and prompt engineering.

---

## 1. Base repo and constraints

Repository: `zeronsh/chat`
Stack (from repo, do not change unless required):

* Frontend: React with TanStack Start.
* Backend: Zero runtime with drizzle ORM.
* LLM: Vercel AI SDK pattern.
* DB: drizzle supported SQL (likely Postgres or SQLite).
* Auth: if present, reuse. If not present, implement simple auth.

You must:

* Keep the existing infra: routing, streaming, drizzle, deployment style.
* Remove or disable:

  * multi provider switching
  * Exa search tool and research tools
  * any unrelated agent tools not used for this spec

Result should be a clean three section application:

* Context Profiles
* Prompt Engineer
* Prompt Library

---

## 2. Core domain concepts

You will introduce these domain entities:

1. `User`
2. `SystemPrompt`
3. `ContextProfile`
4. `Prompt` (saved prompt template)
5. `PromptSession` (interactive chat that results in a prompt)
6. `AppSettings` (holds the shared LLM API key)

### 2.1 User

If the base repo already has a user model, extend that. If not, create a simple one.

Required fields:

* `id` (uuid)
* `email` (unique)
* `password_hash`
* `created_at`
* `updated_at`

Auth requirements:

* Basic email + password sign up and login.
* Sessions via cookies or JWT, whatever matches repo.

All other entities are per user.

### 2.2 SystemPrompt

Store the three system prompts from `system_prompts_rows.csv` here.

Fields:

* `id` (uuid)
* `key` (string, unique)

  * expected values: `json_generator`, `json_updater`, `prompt_generator`
* `content` (text)

  * full system prompt text from the CSV
* `created_at`
* `updated_at`

On first run, seed this table from a static seed or direct code. You can embed the CSV contents or convert to a static array.

These are read only at runtime.

### 2.3 ContextProfile

Represents a reusable block of context for a person, project, role, system or other.

Fields:

* `id` (uuid)
* `user_id` (fk → users.id)
* `name` (string)
* `type` (string enum, store as text): `person | project | role | system | other`
* `tags` (string array or json)
* `json` (jsonb or text)

  * the full context definition generated and maintained by the LLM
* `created_at`
* `updated_at`

The `json` structure is determined by the `json_generator` system prompt. You do not need to hardcode its schema, but you must treat it as complete and self contained.

### 2.4 Prompt

A saved, reusable prompt template ready to paste into ChatGPT, Claude etc.

Fields:

* `id` (uuid)
* `user_id` (fk → users.id)
* `title` (string)
* `raw_prompt` (text)

  * full ready to paste prompt
* `goal` (text)

  * optional description of what this prompt does
* `context_profile_ids` (json array of context_profile ids as strings)
* `tags` (string array or json)
* `created_at`
* `updated_at`

### 2.5 PromptSession

Each multi turn interaction with the Prompt Engineer to produce one prompt.

Fields:

* `id` (uuid)
* `user_id` (fk → users.id)
* `status` (enum stored as text): `in_progress | completed | abandoned`
* `selected_context_profile_ids` (json array of profile ids as strings)
* `llm_conversation_log` (jsonb or text)

  * array of messages like `{ role: "system" | "user" | "assistant", content: string }`
* `draft_prompt` (text or null)

  * latest `<prompt>` body returned by the LLM for this session
* `final_prompt_id` (fk → prompts.id | null)
* `created_at`
* `updated_at`

### 2.6 AppSettings

Single row table to hold the LLM API key for this deployment.

Fields:

* `id` (uuid)

  * can be a constant like `"singleton"`
* `llm_provider` (string)

  * for now can be `"openai"` or `"anthropic"` etc
* `llm_api_key_encrypted` (text)

  * encrypted API key, see Section 5
* `created_at`
* `updated_at`

You can assume only one record exists at a time.

---

## 3. LLM roles and usage

There are three LLM roles, each backed by a system prompt from `system_prompts`:

1. **Context Generator**

   * key: `json_generator`
   * use: create new context profiles from user description.

2. **JSON Updater**

   * key: `json_updater`
   * use: modify existing context profile JSON according to a natural language instruction.

3. **Prompt Production Engineer**

   * key: `prompt_generator`
   * use: multi turn conversation to design an execution ready prompt from context profiles and a task.

Very important:

* `json_generator` and `json_updater` return **full JSON objects** wrapped in `<json>...</json>` tags. You must parse this and treat it as the complete updated object.
* `prompt_generator` returns a completed prompt wrapped in `<prompt>...</prompt>` and a name in `<name>...</name>`. You must parse both.

### 3.1 LLM client abstraction

Create a service module, for example `llmClient.ts`, with at least these methods:

* `generateContextProfile(input: { userId; type; freeText; }): Promise<any>`

  * Looks up system prompt with key `json_generator`.
  * Builds a request that passes the user description and desired type.
  * Calls the LLM using the configured provider and API key.
  * Parses the `<json>...</json>` block from the response.
  * Returns a parsed JSON object.

* `updateContextProfile(input: { userId; existingJson; instruction; }): Promise<any>`

  * Looks up system prompt with key `json_updater`.
  * Sends existing JSON and a natural language instruction.
  * Parses `<json>...</json>` and returns updated JSON.

* `runPromptEngineerTurn(input: { sessionId; conversationLog; contextProfilesJson; taskMeta; }): Promise<{ assistantMessage; promptName?; promptBody?; }>``

  * Looks up system prompt with key `prompt_generator`.
  * Sends:

    * system prompt
    * an appropriate user message that includes:

      * selected context profiles JSON
      * task goal and constraints
      * full conversation history so far
  * Returns the assistant message text.
  * If the assistant message contains `<prompt>` and `<name>`:

    * parse those and return `promptName` and `promptBody`.

All LLM calls must use the same provider and API key from `AppSettings`.

---

## 4. UX and flows

You must rework the UI into three main areas:

* Context Profiles
* Prompt Engineer
* Prompt Library

Re-use existing layout and chat UI from the base repo where possible.

### 4.1 Global layout and navigation

Create a simple top navigation:

* Logo or app name.
* Tabs or links:

  * `Context Profiles`
  * `Prompt Engineer`
  * `Prompt Library`
* Account menu with:

  * user email
  * logout
  * link to `Settings` (for API key)

All main routes must be behind auth.

### 4.2 Auth flow

* On first visit:

  * If no user session:

    * show login/signup screen.
* Sign up:

  * email
  * password
* Login:

  * email
  * password
* After login:

  * if no API key configured in `AppSettings`, redirect or show modal for API key input (section 5).

Use whatever auth pattern the base repo supports. If there is none, implement minimal server side auth.

### 4.3 AppSettings: API key onboarding

Requirement: one LLM provider, bring your own key, with a popup or onboarding.

Flow:

1. After login, check if `AppSettings.llm_api_key_encrypted` exists.
2. If not:

   * show modal:

     * “Paste your LLM API key” (OpenAI or Anthropic etc).
     * Provider select if multiple supported, otherwise a label.
   * On submit:

     * encrypt API key with a server side symmetric key.
     * store in `AppSettings`.
3. If key exists:

   * hide modal.
4. All LLM calls must read the decrypted key from `AppSettings`.

Do not send the API key to the frontend beyond initial capture. Encryption can be simple (for example AES using a secret from env variables).

---

## 5. Context Profiles UI and flows

### 5.1 Context Profiles list

Route: `/contexts`

Features:

* Show a list of context profiles for the current user:

  * name
  * type
  * tags
  * updated_at
* Controls:

  * search by name and tags
  * filter by type
  * button: `New context profile`

### 5.2 Create context profile (assisted)

Flow:

1. User clicks `New context profile`.
2. Show modal or page with:

   * Name (string)
   * Type (select: person, project, role, system, other)
   * Tags (comma separated input)
   * Short description textarea:

     * prompt: “Describe what this profile should represent. Who or what is it, what is it for, what matters?”
3. On submit:

   * backend calls `llmClient.generateContextProfile` with:

     * userId
     * type
     * freeText = user description
   * receives JSON from `<json>` tag.
   * store in `context_profiles` with:

     * `name` from user input (or from JSON if field exists)
     * `type` as selected
     * `tags` as parsed
     * `json` as full JSON
4. Redirect to context profile detail page.

### 5.3 Context profile detail and JSON updater

On route like `/contexts/:id`:

* Show:

  * name, type, tags at top editable inline.
  * a human readable view of JSON below, broken into sections.

Human readable view:

* Derive basic sections from JSON keys if they exist, for example:

  * Goals
  * Constraints
  * Tone and style
  * Audience
  * Background
* Render them as labeled text blocks.
* Also provide a `View raw JSON` toggle.

Natural language quick edit:

* At bottom, show a textarea:

  * placeholder: “Describe what you want to change or add. Example: add my new offer, remove mention of X, make the tone more direct.”
* On submit:

  * backend uses `llmClient.updateContextProfile` with:

    * existingJson (current stored JSON)
    * instruction (user text)
  * receives new JSON from `<json>` tag.
  * over write `context_profiles.json` and set `updated_at`.
  * reload detail page with updated view.

Optional: allow raw JSON editing in an advanced tab. If you do this, validate JSON before saving.

---

## 6. Prompt Engineer UI and flows

This is the core interactive experience. You will reuse the chat UI from `zeronsh/chat` but change semantics.

Route: `/engineer`

### 6.1 Start new prompt session

On `/engineer`, show:

* Left pane or top panel:

  * Multi select of context profiles (name list with checkboxes).
  * Textarea: “What task or goal should this prompt achieve?”
  * Optional textarea: “Constraints or preferences” (for example output format, tone, length).
  * Button: `Start session`.

When user clicks `Start session`:

1. Validate:

   * at least one context profile selected
   * goal not empty
2. In backend:

   * fetch selected context profile JSONs.
   * create a new `prompt_sessions` row:

     * `user_id` current user
     * `status` = `in_progress`
     * `selected_context_profile_ids` = ids
     * `llm_conversation_log` initialised with:

       * system message = content from `SystemPrompt` with key `prompt_generator`
       * user message containing a JSON or structured object with:

         * goal
         * constraints
         * list of context profiles as JSON
3. Call `llmClient.runPromptEngineerTurn`.
4. Save assistant message into `llm_conversation_log`.
5. Render chat UI showing the assistant response and an input box for user reply.

### 6.2 Multi turn chat loop

Frontend chat behavior:

* When user sends a message:

  * append to `llm_conversation_log` as a user message.
  * send full log, selected context profile JSONs, and task meta back to backend endpoint.
* Backend endpoint:

  * loads session and context profiles.
  * calls `llmClient.runPromptEngineerTurn`.
  * appends assistant message to log.
  * checks if assistant message contains `<prompt>` and `<name>`.

    * if not:

      * return assistant message to frontend.
    * if yes:

      * parse `promptBody` and `promptName`.
      * update `prompt_sessions.draft_prompt` = `promptBody`.
      * keep `status` as `in_progress` until saved.
* Frontend continues to display messages until a `<prompt>` appears.
* Once `<prompt>` is detected:

  * show a separate panel with:

    * Prompt title (from `<name>`, editable).
    * Prompt body (from `<prompt>`, large textarea).
    * Buttons:

      * `Save prompt`
      * `Copy prompt`

### 6.3 Save prompt from session

When user clicks `Save prompt`:

1. Backend:

   * create a `prompts` row:

     * `user_id` = current user
     * `title` = user edited or `<name>`
     * `raw_prompt` = prompt body
     * `goal` = goal from session start
     * `context_profile_ids` = session.selected_context_profile_ids
   * update `prompt_sessions`:

     * `final_prompt_id` = new prompt id
     * `status` = `completed`
2. Frontend:

   * show confirmation and link to view prompt in the Prompt Library.

Copy prompt:

* Use clipboard API to copy the full `raw_prompt` content.

---

## 7. Prompt Library UI and flows

Route: `/prompts`

### 7.1 List view

* Show a list of prompts for current user:

  * title
  * tags
  * created_at
  * names of associated context profiles (if any)
* Search input to filter by title or goal substring.
* Filter by tags if tags exist.

### 7.2 Detail view

On clicking a prompt:

* Show:

  * title (editable)
  * goal (editable)
  * associated context profiles (list, clickable)
  * tags (editable)
  * raw prompt in a textarea
* Actions:

  * `Save` (updates the prompt row)
  * `Duplicate` (clones into a new prompt with new id)
  * `Delete` (with confirmation)
  * `Copy prompt` (clipboard)

You do not need to involve the LLM in this view. This is strictly managing stored data.

### 7.3 Manual prompt creation

Add a button `New prompt (manual)` that allows:

* Set title
* Associate context profiles
* Input raw prompt
* Set tags and goal

This uses no LLM calls, but still writes into `prompts` table.

---

## 8. Privacy and export

To keep it local and portable:

* The whole app is intended to be self hosted by the user or small team.
* No third party analytics.
* No external logging of prompt content beyond database.
* Provide export and optionally import per user.

### 8.1 Export

Add an endpoint and UI action `Export my data` that returns JSON:

```json
{
  "context_profiles": [...],
  "prompts": [...],
  "prompt_sessions": [...]
}
```

Use exact DB rows or a safe subset. Return as a file download.

### 8.2 Import (optional for MVP, but preferred)

Add an import flow that:

* accepts a JSON file in the format above
* validates schema
* either merges or replaces existing data for that user

This can be a later step, but design the export structure with import in mind.

---

## 9. Implementation plan for Codex

Use this as an operating sequence.

### 9.1 Setup

1. Clone and run `zeronsh/chat` following its README.
2. Confirm the app builds and the default chat UI works.
3. Inspect the code to identify:

   * LLM provider integration
   * database setup
   * existing auth, if any
   * main page routing and layout

### 9.2 Strip unused features

4. Remove or disable:

   * multiple provider selector in the UI
   * Exa search tool and any other search or research tools
   * any extra agent tools not related to context or prompts

Goal: one provider, one model, minimal tool set.

### 9.3 Schema changes

5. Add drizzle schemas and migrations for:

   * `system_prompts`
   * `context_profiles`
   * `prompts`
   * `prompt_sessions`
   * `app_settings`
   * `users` if not already present

6. If user model already exists, reuse it and connect foreign keys. If not, implement a simple `users` table and hook up auth.

7. Add seeding logic for `system_prompts` using the contents of `system_prompts_rows.csv`. Map keys exactly:

   * `json_generator`
   * `json_updater`
   * `prompt_generator`

### 9.4 LLM client abstraction

8. Implement `llmClient` module that:

   * loads the LLM API key from `AppSettings` and decrypts it
   * reads the correct `SystemPrompt.content` by `key`
   * exposes methods:

     * `generateContextProfile`
     * `updateContextProfile`
     * `runPromptEngineerTurn`

9. Implement robust parsing for `<json>`, `<prompt>`, and `<name>` tags. If tags are missing or malformed, return descriptive errors.

### 9.5 Auth and API key onboarding

10. Ensure a basic auth flow is in place:

    * sign up
    * login
    * logout
    * protect app routes behind auth

11. Implement `AppSettings` logic:

    * on server start or first use, ensure there is one row.
    * create settings page or modal to set the API key and provider.
    * encrypt key before storing.

12. Block access to Prompt Engineer and Context Profiles until an API key is configured.

### 9.6 Context profiles UI and endpoints

13. Create backend endpoints for:

    * list context profiles for user
    * create context profile (calls `generateContextProfile`)
    * get single context profile
    * update metadata (name, type, tags)
    * apply natural language update (calls `updateContextProfile`)
    * optional: raw JSON update

14. Implement frontend for `/contexts`:

    * list, search, filter
    * `New context profile` flow
    * detail page with human friendly view and quick edit box

### 9.7 Prompt Engineer UI and endpoints

15. Create backend endpoints for:

    * create new prompt session (initial call to `runPromptEngineerTurn`)
    * post user reply in prompt session:

      * append to conversation log
      * call `runPromptEngineerTurn` again
      * detect and store `<prompt>` and `<name>` if present

16. Implement frontend for `/engineer`:

    * panel to select context profiles and define goal and constraints
    * chat UI that uses the above endpoints
    * final prompt panel with save and copy actions

17. On save, create `prompts` row and update `prompt_sessions`.

### 9.8 Prompt library UI and endpoints

18. Create backend endpoints for:

    * list prompts
    * get prompt by id
    * create manual prompt
    * update prompt
    * delete prompt

19. Implement frontend for `/prompts`:

    * list with search and tags
    * detail editor
    * copy prompt, duplicate, delete

### 9.9 Export

20. Implement `GET /api/export` (or equivalent) per user that returns JSON with context profiles, prompts, and prompt sessions.

21. Add an `Export my data` button in settings or profile.

### 9.10 Final checks

22. Test flows end to end:

* sign up
* set API key
* create context profile
* update context profile with a natural language edit
* start prompt session with that profile and a goal
* answer clarifying questions
* receive final prompt, save it, copy it
* view and edit saved prompt in library

23. Verify no blocked calls to removed tools or providers, and no references to Exa or other unused features remain.

