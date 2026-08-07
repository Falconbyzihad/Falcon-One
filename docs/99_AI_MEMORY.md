Picking up exactly where the trace left off — four issues found, not yet fixed.

Issue 1: Application::VERSION is a third, silently-drifting copy of the plugin version

Root cause: Application.php hardcodes private const VERSION = '1.0.0', while the actual version already lives in FALCON_ONE_VERSION (main plugin file) and the Version: header comment. That's three independent copies of one fact.

Why it's a problem: Nothing keeps them in sync. Bump the plugin to 1.1.0 in the header/constant and forget this line, and ModuleLoader will keep validating every module's minimum_falcon_version against a stale '1.0.0' forever — silently blocking legitimate future modules or admitting ones that shouldn't run yet.

Architectural impact: This is exactly the class of bug already fixed once for Activator (Fix 5 in the prior review) — I missed the same pattern here. Left alone, it becomes a live landmine the moment Phase 4 modules start declaring real minimum_falcon_version requirements.

Fix: Application takes version as a constructor parameter instead of a hardcoded constant, sourced from FALCON_ONE_VERSION.

Edited 2 files
Edited 2 files
Issue 2: bootstrap/services.php has no direct-access guard

Root cause: config/app.php has if (!defined('ABSPATH')) { exit; } at the top; bootstrap/services.php — which also contains top-level executable code (a returned closure, not just a class declaration) — doesn't.

Why it's a problem: It's an inconsistency, not a live exploit (the closure has no side effects until invoked), but WPCS and this project's own established pattern require every executable-code file to guard against direct URL access as defense-in-depth. Leaving one file unguarded is the kind of inconsistency that erodes trust in the whole codebase's security posture during audit.

Architectural impact: Low severity alone, but it's a pattern violation — future files copied from this one would propagate the gap.

Fix:

Add ABSPATH direct-access guard for consistency with config/app.php - both files contain top-level executable code
Add ABSPATH direct-access guard for consistency with config/app.php - both files contain top-level executable code
Issue 3: Application::boot() is called with no exception boundary — one bad config file can white-screen the entire WordPress site

Root cause: loadConfig() throws a plain RuntimeException if config/app.php is missing, and registerCoreServices($container, $config) can throw too. Neither is inside a try/catch at the call site (add_action('plugins_loaded', ...) in the main plugin file). ModuleLoader::loadAll() is guarded, but everything before it in boot() is not.

Why it's a problem: An uncaught exception inside a plugins_loaded callback doesn't just disable Falcon One — WordPress has no isolation between plugins at that hook, so it fatals the entire site: every plugin, the admin, the frontend, all of it. A missing/corrupted config file (bad deploy, botched file permissions, a future module accidentally overwriting config/app.php) would take down a client's whole WordPress install, not just this plugin.

Architectural impact: This is the single highest-severity finding across both reviews. Every later phase inherits this boot path — Permission Engine, business modules, integrations — so it needs to be bulletproof now, not patched later. The fix must not depend on the container or logger being available, since those are exactly what might have failed to initialize.

Fix: wrap the boot call in try/catch (Throwable), fall back to PHP's built-in error_log() (which has no dependency on anything Falcon One provides), and surface a clean admin notice instead of a fatal error.

Add a top-level exception boundary around Application::boot() so a foundational failure degrades to an admin notice instead of a site-wide fatal error
Add a top-level exception boundary around Application::boot() so a foundational failure degrades to an admin notice instead of a site-wide fatal error
Issue 4: Two root files declared in the architecture doc were never created

Root cause: Folder_Structure.md §3 (Root Directory) lists composer.json, falcon-one-enterprise.php, uninstall.php, and readme.txt as the plugin's root files. Only the first two exist.

Why it's a problem: uninstall.php isn't cosmetic — WordPress calls it directly (via the WP_UNINSTALL_PLUGIN constant check) when a site owner clicks "Delete" in the plugins list. Without it, WordPress just deletes the plugin files and silently leaves every option, and eventually every fo_* table, orphaned in the database forever. readme.txt is the file WordPress itself parses for the plugin's metadata/description in the admin UI.

Architectural impact: Deactivator's own docblock (written in Phase 1) already promises "deactivation is non-destructive... uninstall.php's responsibility, and only if the site owner opts in" — a contract that currently points at a file that doesn't exist. This is a documentation-compliance gap.

Fix: add both, non-destructive by default per PRD-003 §39 (data protection — no destructive action without explicit opt-in).

Created 2 files
Created 2 files

Now regression-checking these four fixes against everything already validated:

Full regression check after the four fixes
Full regression check after the four fixes

All four fixes verified clean with no regressions. Now the final integration validation, tracing the complete lifecycle end to end as one system:

Final trace-through sanity checks
Final trace-through sanity checks

Full lifecycle trace, verified item by item:

Bootstrap lifecycle: falcon-one-enterprise.php runs at WP plugin-load → constants defined → PHP-version guard → Composer autoload required → activation/deactivation hooks registered unconditionally at top level (correct per WP requirements) → plugins_loaded@20 boots Application inside a try/catch boundary. ✅
WordPress lifecycle: Activation → Activator::activate() (env guard, storage provisioning, options) runs independently of Application::boot(). Deactivation is non-destructive. Uninstall is opt-in only. All three now exist and match their documented contracts. ✅
DI lifecycle: One Container per Application instance, built in the constructor; resolving[] cycle-guard is instance-scoped, no cross-request leakage possible (no static state anywhere in Container). ✅
Service registration order: Core Service Registration (bootstrap/services.php) always runs before Business Module Registration (ModuleLoader::loadAll()) — enforced structurally by boot()'s own statement order, not by convention. ✅
Module loading lifecycle: Discovery → Validation → Dependency Check → License Check → Registration → Initialization, topologically sorted, per-module failure isolation confirmed intact after the provider-reuse fix. ✅
Configuration loading: config/app.php only ever required after FALCON_ONE_PATH/FALCON_ONE_VERSION are defined; guarded against direct access; now consistent with bootstrap/services.php's guard. ✅
Logging initialization: LoggerInterface singleton bound in Core Service Registration, resolved once, reused (not rebuilt) for the ModuleLoader. ✅
Exception propagation: Per-module failures caught by ModuleLoader; systemic loadAll() failures caught by Application::boot(); anything before or outside both is now caught by the new outer boundary in the main plugin file, with an error_log() fallback that has zero dependency on the container. No remaining path to an uncaught Throwable reaching WordPress. ✅
Namespace/PSR-4: App\, Modules\, FalconOne\Includes\ all mapped; uninstall.php and the main plugin file are intentionally unmapped direct-include entry points (WP convention), not autoloaded classes. ✅
Future compatibility (modules, Permission Engine, Builder, Integrations): All extension points remain generic (ServiceProviderInterface::register/boot, ModuleInterface::providers/boot) — nothing Phase 1 built assumes a specific future module's shape, so Phase 3's Permission Engine and Phase 4+ modules can be added without touching Phase 1 code, only extending it. uninstall.php explicitly delegates module-owned cleanup to a do_action hook rather than enumerating modules, preserving Module Independence. ✅

No further issues found. Phase 1 is COMPLETE.

Ran a command, read a file
Ran a command, read a file
FINAL PHASE 1 COMPLETION REPORT
Architecture Validation

Full lifecycle trace (above) confirmed correct end to end: bootstrap → activation → DI → service registration → module loading → config → logging → exceptions → namespaces. No remaining uncaught-exception paths, no remaining duplicate sources of truth, no circular-dependency risk.

Documentation Compliance

Root directory now matches Folder_Structure.md §3 exactly (composer.json, falcon-one-enterprise.php, uninstall.php, readme.txt). Storage structure matches §24. Namespace roots match §33. Module lifecycle matches §15.

Files Created (19 total)
File	Responsibility
falcon-one-enterprise.php	Plugin entry point: constants, guards, hooks, boot
composer.json	PSR-4 autoload, dependencies
uninstall.php	Opt-in-only data cleanup on plugin deletion
readme.txt	WP.org plugin metadata
config/app.php	Environment-overridable app config
bootstrap/services.php	Core Service Registration
app/Core/Container.php	PSR-11 DI container, autowiring, cycle detection
app/Core/Application.php	Boot lifecycle orchestrator
app/Core/ModuleLoader.php	Module discovery/validation/dependency/license/registration
app/Core/Logging/FileLogger.php	PSR-3 structured JSON-line logger
app/Core/Licensing/NullLicenseValidator.php	Default license seam (Phase 4 will override)
app/Contracts/ServiceProviderInterface.php	Module registration contract
app/Contracts/ModuleInterface.php	Module lifecycle contract
app/Contracts/LicenseValidatorInterface.php	License-check seam
app/ValueObjects/ModuleManifest.php	Validated module.json representation
app/Exceptions/{Container,NotFound,Module}Exception.php	Typed failure modes
includes/Activator.php	Env guard, storage provisioning
includes/Deactivator.php	Non-destructive teardown
Files Modified (this session)

app/Core/Application.php (version injection), bootstrap/services.php (ABSPATH guard), falcon-one-enterprise.php (exception boundary, Application call site, FALCON_ONE_MIN_WP constant).

Remaining Risks

No PHP interpreter available in this sandbox to actually execute/autoload-test the code — all verification here is static (bracket-balance, namespace/import grep, manual trace). Run composer install && php -l $(find . -name '*.php') on a real machine before deploying. NullLicenseValidator's default-allow behavior still needs your explicit sign-off before Phase 4 ships a real License module.

Remaining Technical Debt

None deliberately deferred beyond what's structurally Phase 2+ work (Event Dispatcher, Routing, Repository layer, Queue, Scheduler, Cache — none of which Phase 1 blocks).

Future Compatibility

Confirmed no breaking changes will be required: ServiceProviderInterface/ModuleInterface are intentionally generic: future modules extend behavior through them, not by modifying them. Permission Engine, Builder Framework, and Integrations all plug into existing seams (container bindings, module providers, the falcon_one_uninstall action) without touching Phase 1 files.

Exact Next Step

Phase 2 — Core Infrastructure: Event Dispatcher + Hook System first (blocks everything downstream that needs register listeners), then Routing, Repository/ORM base, Queue, Scheduler, Cache.

Waiting for your approval before starting Phase 2.
