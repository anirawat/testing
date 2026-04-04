# Ayurvedic CRM + Leads Dashboard — PHASE 1 Architecture

## STEP 1: Plugin purpose and system overview

### Naming Registry (Locked for all later phases)
- **Plugin slug:** `ayurvedic-crm-leads-dashboard`
- **Text domain:** `ayurvedic-crm-leads-dashboard`
- **Main PHP constant prefix:** `ACLD_`
- **Class prefix:** `ACLD_`
- **Function prefix:** `acld_`
- **Hook/action/filter prefix:** `acld_`
- **AJAX action prefix:** `acld_`
- **REST namespace:** `acld/v1`
- **DB table base names:**
  - `{$wpdb->prefix}acld_leads`
  - `{$wpdb->prefix}acld_lead_activity`
  - `{$wpdb->prefix}acld_assignment_history`
  - `{$wpdb->prefix}acld_lead_meta` (extensibility table)
- **Option keys:**
  - `acld_version`
  - `acld_db_version`
  - `acld_activation_flag`
  - `acld_migration_lock`
  - `acld_settings`
  - `acld_status_config`
  - `acld_notification_settings`
- **Transient/lock keys:** `acld_migration_lock`, `acld_export_lock_{user_id}`
- **Nonce keys:**
  - `acld_admin_nonce`
  - `acld_lead_action_nonce`
  - `acld_export_nonce`
  - `acld_ajax_nonce`
  - `acld_public_capture_nonce`
- **Screen ID prefix:** `toplevel_page_acld_dashboard` and `acld_page_*`
- **CSS prefix:** `.acld-`
- **JS global object:** `window.ACLD`
- **Cron hooks:**
  - `acld_run_deferred_migrations`
  - `acld_send_followup_reminders`
  - `acld_cleanup_logs`

### Purpose
A production-grade, WordPress-native CRM plugin for Ayurvedic clinics and wellness lead ops teams, handling lead intake, assignment, pipeline tracking, follow-ups, conversion monitoring, and analytics through a secure and scalable admin dashboard.

### System overview
- **Core runtime layers:**
  1. Bootstrap kernel (safe loading, flags, version checks)
  2. Service registry/container
  3. DB/migration layer
  4. CRM domain services (lead capture, scoring, assignment, duplicate checks)
  5. Admin UI controllers + templates
  6. AJAX/REST adapters
  7. Notifications/integrations modules
- **Design principle:** lightweight request path, deferred heavy tasks, strongly modular service boundaries.

## STEP 2: Folder and file structure

```text
/ayurvedic-crm-leads-dashboard
    ayurvedic-crm-leads-dashboard.php
    uninstall.php
    readme.txt
    /includes
        /core
            class-acld-kernel.php
            class-acld-loader.php
            class-acld-container.php
            class-acld-activator.php
            class-acld-deactivator.php
            class-acld-upgrader.php
            class-acld-scheduler.php
        /db
            class-acld-db-migrator.php
            class-acld-db-schema.php
            class-acld-db-leads-repository.php
            class-acld-db-activity-repository.php
            class-acld-db-assignment-repository.php
            class-acld-db-meta-repository.php
        /crm
            class-acld-lead-service.php
            class-acld-lead-validator.php
            class-acld-duplicate-service.php
            class-acld-assignment-service.php
            class-acld-pipeline-service.php
            class-acld-followup-service.php
            class-acld-export-service.php
            class-acld-activity-service.php
        /admin
            class-acld-admin-bootstrap.php
            class-acld-admin-menu.php
            class-acld-admin-assets.php
            class-acld-admin-dashboard-controller.php
            class-acld-admin-leads-controller.php
            class-acld-admin-lead-detail-controller.php
            class-acld-admin-assignments-controller.php
            class-acld-admin-exports-controller.php
            class-acld-admin-settings-controller.php
            class-acld-admin-notifications-controller.php
            class-acld-admin-integrations-controller.php
            class-acld-admin-tools-controller.php
        /ajax
            class-acld-ajax-router.php
            class-acld-ajax-leads.php
            class-acld-ajax-assignments.php
            class-acld-ajax-notes.php
            class-acld-ajax-export.php
        /public
            class-acld-public-bootstrap.php
            class-acld-shortcode-lead-form.php
            class-acld-rest-controller.php
            class-acld-webhook-controller.php
        /notifications
            class-acld-notification-service.php
            class-acld-email-template-service.php
        /integrations
            class-acld-integrations-manager.php
            class-acld-integration-google-sheets.php
            class-acld-integration-whatsapp.php
            class-acld-integration-webhook.php
        /helpers
            class-acld-sanitizer.php
            class-acld-formatter.php
            class-acld-date-helper.php
            class-acld-utm-helper.php
        /security
            class-acld-capabilities.php
            class-acld-nonce.php
            class-acld-audit-logger.php
        /modules
            /future
                class-acld-sla-module.php
                class-acld-lead-scoring-module.php
    /assets
        /css
            admin.css
            leads.css
            dashboard.css
        /js
            admin.js
            leads.js
            dashboard.js
            lead-detail.js
    /templates
        /admin
            dashboard.php
            leads-list.php
            lead-detail.php
            assignments.php
            exports.php
            settings.php
            notifications.php
            integrations.php
            tools.php
        /public
            lead-capture-form.php
    /languages
        ayurvedic-crm-leads-dashboard.pot
```

## STEP 3: CRM data flow

1. **Capture entry points:** shortcode form, REST endpoint, webhook endpoint, manual admin entry, future integrations.
2. **Input sanitation/validation:** `ACLD_Lead_Validator` + `ACLD_Sanitizer` normalize payload.
3. **Duplicate check pass:** by normalized phone/email + `duplicate_hash`; result tagged (`is_duplicate`, `duplicate_of`).
4. **Lead create/update transaction:** insert lead in `acld_leads`, write `lead_activity` event, optionally assignment history entry.
5. **Business enrichment:** pipeline default status (`new`), score bucket, source classification, follow-up defaults.
6. **Notifications:** async-safe trigger (mail immediate lightweight, heavy providers deferred).
7. **Admin consumption:** dashboard aggregate queries, paginated leads listing, detail views, activity timeline.
8. **Actions lifecycle:** quick status/assignment/note updates through AJAX → service layer → repositories → activity log.
9. **Export:** filtered ID set generated, batched CSV streaming with lock key and capability+nonce checks.

## STEP 4: Database schema plan

### 1) `{$wpdb->prefix}acld_leads`
- PK: `id BIGINT UNSIGNED AUTO_INCREMENT`
- Identity/contact: `full_name`, `phone`, `email`, `city`, `age`, `gender`
- Clinical/interest: `concern`, `answers_payload LONGTEXT`, `assessment_score DECIMAL(10,2)`, `result_category`
- Attribution: `source`, `source_detail`, `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content`, `landing_page`, `referrer`, `device_type`
- Pipeline/ownership: `status`, `sub_status`, `priority`, `assigned_to BIGINT`, `assigned_doctor BIGINT`, `lead_quality`, `lead_score INT`
- Engagement: `followup_date DATETIME NULL`, `last_contacted_at DATETIME NULL`, `next_action VARCHAR(255)`
- Notes/meta-lite: `notes TEXT`, `internal_comments LONGTEXT`, `tags TEXT`
- Duplicate handling: `duplicate_hash CHAR(64)`, `is_duplicate TINYINT(1)`, `duplicate_of BIGINT NULL`
- Timestamps: `created_at DATETIME`, `updated_at DATETIME`
- Indexes:
  - `KEY idx_phone (phone)`
  - `KEY idx_email (email)`
  - `KEY idx_status (status)`
  - `KEY idx_assigned_to (assigned_to)`
  - `KEY idx_followup_date (followup_date)`
  - `KEY idx_created_at (created_at)`
  - `KEY idx_source (source)`
  - `KEY idx_utm_campaign (utm_campaign)`
  - `KEY idx_duplicate_hash (duplicate_hash)`

### 2) `{$wpdb->prefix}acld_lead_activity`
- PK: `id`
- `lead_id BIGINT UNSIGNED`
- `activity_type VARCHAR(80)` (created, note_added, status_changed, assignment_changed, followup_set, duplicate_flagged, exported, etc.)
- `activity_message TEXT`
- `meta_json LONGTEXT`
- `created_by BIGINT UNSIGNED NULL`
- `created_at DATETIME`
- Indexes: `idx_lead_id`, `idx_activity_type`, `idx_created_at`, composite `(lead_id, created_at)`

### 3) `{$wpdb->prefix}acld_assignment_history`
- PK: `id`
- `lead_id`, `from_user_id`, `to_user_id`, `assigned_role`, `assigned_by`, `assigned_at`, `reason`, `meta_json`
- Indexes: `idx_lead_id`, `idx_to_user_id`, `idx_assigned_at`

### 4) `{$wpdb->prefix}acld_lead_meta`
- PK: `id`
- `lead_id`, `meta_key`, `meta_value LONGTEXT`, `updated_at`
- Unique key: `(lead_id, meta_key)`
- Used for extensibility without immediate schema changes.

### Versioning
- Plugin version option: `acld_version`
- DB version option: `acld_db_version`
- Migration target constants e.g. `ACLD_DB_VERSION = '1.0.0'`.

## STEP 5: Hooks/actions/filters map

### Core WP hooks
- `plugins_loaded` → kernel bootstrap registration
- `init` → register public shortcodes, lightweight routes
- `admin_menu` → register CRM pages
- `admin_enqueue_scripts` → load scoped assets only on ACLD screens
- `wp_ajax_acld_*` → protected admin actions
- `wp_ajax_nopriv_acld_capture_lead` (only if public form needs AJAX)
- `rest_api_init` → register `acld/v1` routes
- `cron_schedules` + `acld_send_followup_reminders`

### Internal custom hooks
- `do_action('acld_lead_created', $lead_id, $lead_data)`
- `do_action('acld_lead_updated', $lead_id, $changes)`
- `do_action('acld_lead_assigned', $lead_id, $to_user_id, $from_user_id)`
- `do_action('acld_lead_status_changed', $lead_id, $old_status, $new_status)`
- `do_action('acld_duplicate_detected', $lead_id, $duplicate_of)`
- `do_action('acld_followup_due', $lead_id)`

### Filters
- `apply_filters('acld_default_statuses', $statuses)`
- `apply_filters('acld_lead_capture_payload', $payload)`
- `apply_filters('acld_assignment_candidates', $users, $lead)`
- `apply_filters('acld_export_columns', $columns)`
- `apply_filters('acld_notification_recipients', $emails, $context)`

## STEP 6: Admin pages plan

1. **Dashboard (`acld_dashboard`)**
   - KPI cards: total/today/week/month, converted, due follow-ups, overdue.
   - Chart blocks: source-wise, status-wise, UTM campaign performance.
   - Panels: recent activity, assigned vs unassigned, lead quality buckets.

2. **All Leads (`acld_leads`)**
   - Server-side pagination.
   - Filters: status/date/source/assigned/city/quality/score.
   - Quick actions via AJAX: status, assignment, notes.
   - Bulk actions: assign, status change, export selected.

3. **Lead Detail (`acld_lead_detail`)**
   - Primary info and attribution cards.
   - Timeline (activity + notes + status transitions).
   - Assignment panel, follow-up scheduler, internal comments.

4. **Assignments (`acld_assignments`)**
   - Assignment queue and reassignment workflows.
   - Workload view by doctor/counselor/staff.

5. **Exports (`acld_exports`)**
   - Filter builder + column selector.
   - CSV streaming exports with batched query cursor.

6. **Settings (`acld_settings`)**
   - General settings, pipeline config, duplicate rules.

7. **Notifications (`acld_notifications`)**
   - Toggle new lead/assignment emails and templates.

8. **Integrations (`acld_integrations`)**
   - API keys/webhook endpoints/future provider toggles.

9. **Advanced Tools (`acld_tools`)**
   - Safe migration trigger, data health checks, logs cleanup.

## STEP 7: Lead lifecycle logic

1. **Create lead** → status `new`, assignment optional.
2. **Qualification flow**: `new` → `contacted` → `interested` → `qualified`.
3. **Outcome flow**:
   - positive: `consultation_booked` → `converted`
   - pending: `followup_pending`
   - negative: `not_responding` or `lost`
4. **Sub-status support** via configurable option map `acld_status_config`.
5. **Status change rules**:
   - each transition logged in activity
   - optional transition validation matrix filter
6. **Follow-up logic**:
   - if status is pending/contacted/interested, enforce/encourage `followup_date`
   - due + overdue tracked via date indexes
7. **Lead quality segmentation**: hot/warm/cold based on score thresholds in settings.

### Assignment logic
- Manual assign/reassign from leads list/detail.
- Captures `assigned_by`, `assigned_at`, previous assignee.
- Writes both lead row update + assignment history event.
- Future rule engine plug-in point: `acld_assignment_candidates` filter + module loader.

### Duplicate prevention logic
- Normalize phone/email.
- Build `duplicate_hash = sha256(lower(email)|digits(phone))` with guards for missing values.
- Check recent match window + full history fallback indexed by hash.
- If duplicate:
  - set `is_duplicate = 1`
  - set `duplicate_of` candidate lead id
  - log activity `duplicate_flagged`
  - show admin warning ribbon and list filter.

### Notification flow
- On lead create: if enabled, send concise email to configured recipients.
- On assignment change: notify assigned user and optional manager group.
- Uses centralized notification service with template resolver.
- Fail-safe: mail failures logged to activity/log table, no fatal throw.

### Export flow
- User submits filtered export request with nonce.
- Build query constraints from sanitized filter object.
- Stream CSV in chunks (e.g., 500 rows) to avoid memory spikes.
- Use lock `acld_export_lock_{user_id}` to prevent concurrent heavy exports.
- Record export activity event.

## STEP 8: Security and permissions model

- **Capabilities**
  - `acld_manage_crm` (full)
  - `acld_view_leads`
  - `acld_edit_leads`
  - `acld_assign_leads`
  - `acld_export_leads`
  - `acld_manage_settings`
- Map default admin roles on activation; customizable later.
- **Nonce checks** for every mutation (AJAX, forms, export, settings).
- **Sanitization/escaping** centralized helper usage per field type.
- **Prepared SQL only** in repositories.
- **REST permission callbacks** mandatory.
- **Audit logging** for critical operations: delete, status bulk update, assignment bulk update, export.

## STEP 9: Stability / timeout / activation-safety strategy

### Activation safety
- Activation only sets:
  - `acld_version`
  - `acld_activation_flag = 1`
  - schedule single event `acld_run_deferred_migrations`
- No heavy schema generation directly in activation.

### Deferred migrations
- On admin-init-safe checkpoint or cron event:
  - verify lock `acld_migration_lock`
  - compare `acld_db_version`
  - run idempotent migration steps once
  - update DB version
  - clear lock
- Includes timeout guards and stepwise migration map.

### Bootstrap safety
- Context-aware loading:
  - admin classes only in admin
  - public capture only when needed
  - AJAX route classes only during `DOING_AJAX`
  - cron tasks only during `DOING_CRON`
- No heavy queries during bootstrap.

### Update-safe loading
- Strict include order: constants → loader/container → services.
- `file_exists` + `class_exists` defensive checks for optional modules.
- No output in includes.
- No schema routines on every page request.

### Timeout prevention
- Batch processing for exports/reminders/sync.
- Aggregates query with indexes and limits.
- No full-table scans for dashboard (use grouped indexed ranges).
- Optional transient cache (short TTL) for dashboard cards.

## STEP 10: Scalability / future modules strategy

- **Module contract:** each future module registers via integrator class and hooks only.
- **Repository pattern:** domain services isolated from SQL details.
- **Lead meta table:** custom fields and external IDs without schema churn.
- **Events-first architecture:** internal actions allow drop-in automation modules.
- **Feature flags in settings:** modules can be enabled per clinic branch.
- **Branch readiness:** add `branch_id` in v1.1 migration with compatible defaults.
- **Analytics readiness:** activity table captures state changes needed for funnel and stage-aging reports.

## STEP 11: File-by-file build roadmap

1. **Foundation phase**
   - main plugin file, kernel, loader, container, activator/deactivator, scheduler
   - done first to enforce safe lifecycle boundaries.
2. **DB phase**
   - schema, migrator, repositories
   - build with migration lock + versioning before feature logic.
3. **CRM core phase**
   - lead service, validator, duplicate, assignment, pipeline, followup, activity, export services.
4. **Security/helpers phase**
   - capabilities, nonce utility, sanitizer, audit logger.
5. **Admin phase**
   - menu, assets, dashboard/leads/detail/controllers + templates.
6. **AJAX phase**
   - router + lead/assignment/note/export handlers.
7. **Public/Integration phase**
   - shortcode form, REST controller, webhook controller, integration manager.
8. **Notifications phase**
   - notification service + template service + settings wiring.
9. **Hardening phase**
   - performance tests, pagination checks, capability audits, migration retry tests.
10. **Future module stubs phase**
   - module registry + placeholders for SLA/scoring/WhatsApp sheets sync.

---
Stop after PHASE 1 by design. PHASE 2 (implementation plan with creation order and checkpoints) starts only on next instruction.
