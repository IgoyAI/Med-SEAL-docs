# Extending OpenEMR

OpenEMR is the clinical EMR in the Med-SEAL stack. This page explains the supported extension points and how to make customisations without breaking the upstream update path.

---

## Extension Philosophy

Med-SEAL follows OpenEMR's official customisation model:

- **Custom modules** -- placed in `openemr/interface/modules/custom_modules/`
- **Custom SQL patches** -- placed in `openemr/sql_upgrade.php` additions
- **Configuration overrides** -- via `openemr/sites/default/`
- **Hooks and events** -- via OpenEMR's `EventDispatcher`

Avoid modifying core OpenEMR PHP files directly. Instead, use the override and hook system so your changes survive OpenEMR version upgrades.

---

## Project Layout

Custom OpenEMR files in Med-SEAL live under:

```
openemr/
├── interface/
│   └── modules/
│       └── custom_modules/
│           └── medseal/        # Med-SEAL custom module
├── sites/
│   └── default/
│       └── sqlconf.php         # DB config (auto-generated)
└── library/
    └── medseal/                # Shared Med-SEAL PHP utilities
```

---

## Adding a Custom Module

1. Create a directory: `openemr/interface/modules/custom_modules/my_module/`
2. Add the module entry point: `my_module/index.php`
3. Register the module in `openemr/interface/modules/custom_modules/my_module/info.php`:

```php
<?php
$module_info = array(
    'id'          => 'my_module',
    'name'        => 'My Module',
    'version'     => '1.0.0',
    'description' => 'Custom extension for Med-SEAL',
    'acl_req'     => array('patients', 'auth_med'),
);
```

4. Activate the module in OpenEMR Admin: **Admin > Modules > Manage Modules**.

---

## Using the OpenEMR EventDispatcher

OpenEMR fires events at key points in the clinical workflow. Subscribe to them from your module:

```php
<?php
use OpenEMR\Events\PatientDemographics\RenderEvent;
use Symfony\Component\EventDispatcher\EventDispatcherInterface;

/** @var EventDispatcherInterface $eventDispatcher */
$eventDispatcher->addListener(RenderEvent::EVENT_SECTION_LIST_RENDER_AFTER, function (RenderEvent $event) {
    echo '<div class="medseal-section">AI Insights for this patient</div>';
});
```

Common event hooks:

| Event | When fired |
|---|---|
| `RenderEvent::EVENT_SECTION_LIST_RENDER_AFTER` | Patient demographics page render |
| `PatientDocumentsRenderEvent` | Documents tab render |
| `MedicalProblemEvent` | Problem list update |
| `AppointmentSetEvent` | Appointment creation |

---

## Calling the Med-SEAL AI Service from OpenEMR

Use the shared PHP client in `openemr/library/medseal/AiServiceClient.php`:

```php
<?php
use MedSeal\AiServiceClient;

$client = new AiServiceClient(getenv('AI_SERVICE_URL'));
$summary = $client->getPrevisitSummary($patientId, $appointmentId);

echo htmlspecialchars($summary['sections'][0]['content']);
```

The client handles authentication via the SSO token stored in the OpenEMR session.

---

## Database Changes

If your module requires additional tables or columns:

1. Create a migration SQL file: `openemr/contrib/migrations/001_my_module.sql`
2. Apply it in the module's `setup.php`:

```php
<?php
sqlStatement("CREATE TABLE IF NOT EXISTS medseal_my_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    patient_id VARCHAR(64) NOT NULL,
    data JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)");
```

3. Always use `IF NOT EXISTS` to make migrations idempotent.

---

## Updating OpenEMR

To update OpenEMR in the Docker image:

1. Change the image tag in `docker-compose.yml`:
   ```yaml
   openemr:
     image: openemr/openemr:7.0.3  # bumped version
   ```
2. Test that your custom modules still load correctly.
3. Check the [OpenEMR release notes](https://www.open-emr.org/wiki/index.php/OpenEMR_Release_Notes) for breaking changes.
4. Rebuild: `docker compose up -d --build openemr`.

---

## ACL (Access Control)

Register new ACL rules for your module in `openemr/acl/acl.php`:

```php
// Grant access only to users with 'admin' role
if (!acl_check('admin', 'super')) {
    die('Access Denied');
}
```

Use the principle of least privilege: request only the ACL roles your module genuinely needs.
