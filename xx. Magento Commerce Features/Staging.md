# Magento 2 Staging

## High level staging implementation overview

- Future editions of products/categories etc. are saved as a copy with a timestamped from-to columns.
- Current staging version is saved as a flag, used in all collections to add WHERE condition
- You can browse (preview) frontend site seeing future versions passing version and signature GET parameters


## Demonstrate an understanding of the events processing flow

Influence of Staging on the event processing.

What is a modification of the event processing mechanism introduced by the staging module?


\Magento\Staging\Model\Event\Manager:

- ban some events in preview mode - none by default
- ban some observers in preview mode

  - `catalog_category_prepare_save`
  - `catalog_category_save_after`
  - `catalog_category_save_before`
  - `catalog_category_move_after`


## DB collection flag `disable_staging_preview`

- connection->select()
- \Magento\Framework\DB\Adapter\Pdo\Mysql::select
- Magento\Framework\DB\SelectFactory::create
- Magento\Framework\DB\Select\SelectRenderer
- // work with select...
- \Magento\Framework\DB\Select::assemble
- \Magento\Framework\DB\Select\SelectRenderer::render
  - \Magento\Framework\DB\Select\FromRenderer
  - \Magento\Staging\Model\Select\FromRenderer


```
from = [
    flag = [
        joinType = 'from'
        tableName = 'flag'
    ]
]
```

```
$select->where($alias . '.created_in <= ?', $versionId);
$select->where($alias . '.updated_in > ?', $versionId);
```

Only known staged tables are affected - \Magento\Staging\Model\StagingList::getEntitiesTables - `*_entity`


## Staging modification to the Magento database operations (row_id, entity managers)

Flow modifications introduced by staging (triggers, row_id, data versions).

- deletes foreign keys to removed entity_id column
- creates foreign key to row_id
- replaces indexes

**Triggers** created differently! Attributes no longer have entity_id for `_cl` tables, so one more
extra call to load entity_id by row_id is added to each update by schedule trigger.

\Magento\Framework\Mview\View\Subscription::buildStatement
vs
\Magento\CatalogStaging\Model\Mview\View\Attribute\Subscription::buildStatement


trg_catalog_product_entity_int_after_insert:
```SQL
-- before
INSERT IGNORE INTO `catalog_product_flat_cl` (`entity_id`) VALUES (NEW.`entity_id`);
-- after
SET @entity_id = (SELECT `entity_id` FROM `catalog_product_entity` WHERE `row_id` = NEW.`row_id`);
INSERT IGNORE INTO `catalog_product_flat_cl` (`entity_id`) values(@entity_id);
```

trg_catalog_product_entity_int_after_update:
```SQL
-- before
IF (NEW.`value_id` <=> OLD.`value_id` OR NEW.`attribute_id` <=> OLD.`attribute_id` OR NEW.`store_id` <=> OLD.`store_id` OR NEW.`entity_id` <=> OLD.`entity_id` OR NEW.`value` <=> OLD.`value`) THEN INSERT IGNORE INTO `catalog_product_flat_cl` (`entity_id`) VALUES (NEW.`entity_id`); END IF;
-- after
SET @entity_id = (SELECT `entity_id` FROM `catalog_product_entity` WHERE `row_id` = NEW.`row_id`);
IF (NOT(NEW.`value_id` <=> OLD.`value_id`) OR NOT(NEW.`attribute_id` <=> OLD.`attribute_id`) OR NOT(NEW.`store_id` <=> OLD.`store_id`) OR NOT(NEW.`value` <=> OLD.`value`) OR NOT(NEW.`row_id` <=> OLD.`row_id`)) THEN INSERT IGNORE INTO `catalog_product_flat_cl` (`entity_id`) values(@entity_id); END IF;
```

trg_catalog_product_entity_int_after_delete:
```SQL
-- before
INSERT IGNORE INTO `catalog_product_flat_cl` (`entity_id`) VALUES (OLD.`entity_id`);
-- after
SET @entity_id = (SELECT `entity_id` FROM `catalog_product_entity` WHERE `row_id` = OLD.`row_id`);
INSERT IGNORE INTO `catalog_product_flat_cl` (`entity_id`) values(@entity_id);
```


## sequence tables

Staging migration scripts do this: `row_id` becomes auto_increment, and `entity_id` (`page_id` etc.) column
is no longer auto incremented, but still required!

Requests like `INSERT INTO cms_page (identified) VALUES("test_page")` would result in error because page_id
is normally auto_increment and not passed in request.

Magento fixes this by manually generating auto increments for entity_id in separate sequence tables.

Exmple:

- \Magento\Framework\Model\ResourceModel\Db\CreateEntityRow::prepareData
- `$output[$metadata->getIdentifierField()] = $metadata->generateIdentifier();`
- \Magento\Framework\EntityManager\Sequence\Sequence::getNextValue
- insert and return next ID into `sequence_*` table

`sequence_cms_page`, `sequence_catalog_category`, `sequence_catalogrule`, ...

In community edition (no staging), `$metadata->generateIdentifier()` because sequenceTable is not passed.


- \Magento\Framework\EntityManager\MetadataPool::getMetadata
- \Magento\Framework\EntityManager\MetadataPool::createMetadata
- new \Magento\Framework\EntityManager\EntityMetadata
  - `sequence` argument: \Magento\Framework\EntityManager\Sequence\SequenceFactory::create:
    - no staging - sequence = null
    - staging defines `sequenceTable` - `new \Magento\Framework\EntityManager\Sequence\Sequence(connection, sequenceTable)`



## Common staging plugins

- FrontControllerInterface::beforeDispatch - validate `___version`, `___timestamp`, `___signature`
- Magento\PageCache\Model\Config::afterIsEnabled - disabled in preview mode
- Magento\Store\Model\BaseUrlChecker - disabled in preview mode
- Magento\Framework\Stdlib\DateTime\Timezone::isScopeDateInInterval - interval never validated in preview mode
- Magento\Store\Model\StoreResolver::getCurrentStoreId - use `___store` in preview mode
- Magento\Customer\Model\Session::regenerateId, destroy - never regenerate/destroy in preview mode
- getUrl - adds `___version`, `___store`, possibly `__timestamp`, `__signature` in preview mode
- disables block_html caching


## Staging-related modifications of the indexing process

- plugin before Magento\Catalog\Controller\Category\View
- \Magento\CatalogStaging\Model\Plugin\Controller\View::beforeExecute
- \Magento\CatalogStaging\Model\Indexer\Category\Product\Preview::execute
- \Magento\Catalog\Model\Indexer\Category\Product\AbstractAction::reindex

Reindex using temporary tables with suffix `_catalog_staging_tmp`,
e.g. `catalog_category_product_index_store1_catalog_staging_tmp`


## Save staging update

\Magento\SalesRuleStaging\Controller\Adminhtml\Update\Save::execute
\Magento\Staging\Model\Entity\Update\Save::execute

```
staging[mode]: save
staging[update_id]: 
staging[name]: 3 September
staging[description]: Everything is new
staging[start_time]: 2020-09-03T07:00:00.000Z
staging[end_time]: 
```

\Magento\Staging\Model\Entity\Update\Action\Pool::getAction(
  entityType = 'Magento\SalesRule\Api\Data\RuleInterface',
  namespace = 'save',
  actionType = mode = 'save'
)

- \Magento\Staging\Model\Entity\Update\Action\Save\SaveAction:
  - \Magento\Staging\Model\Entity\Update\Action\Save\SaveAction::createUpdate
  - \Magento\Staging\Model\EntityStaging::schedule
    - \Magento\CatalogRuleStaging\Model\CatalogRuleStaging::schedule
      - validates intersecting updates
      - sets `created_in`, saves entity model (catalog rule)
- Magento\CatalogRuleStaging\Model\Rule\Hydrator
- \Magento\CatalogRuleStaging\Model\Rule\Retriever::getEntity - $this->ruleRepository->get($entityId)


## Preview

All links are automatially updated:
- \Magento\Framework\Url::getUrl
- \Magento\Framework\Url\RouteParamsPreprocessorComposite::execute
- \Magento\Staging\Model\Preview\RouteParamsPreprocessor::execute
- `[_query][___version]=`, `[_query][___store]=`
- `__timestamp=`, `__signature=`

Example:

vendor/magento/module-catalog/view/frontend/layout/default.xml:
```XML
<block class="Magento\Catalog\Block\FrontendStorageManager" name="frontend-storage-manager" before="-"
       template="Magento_Catalog::frontend_storage_manager.phtml">
    <arguments>
        <!-- ... -->
        <item name="updateRequestConfig" xsi:type="array">
            <item name="url" xsi:type="serviceUrl" path="/products-render-info"/>
        </item>
```

- \Magento\Store\Model\Argument\Interpreter\ServiceUrl::evaluate
- https://m24ee.local/rest/default/V1/?___store=default&___version=1599422820&__signature=b5cd66d9ea383ea57356f28cf2dd0f8edeecbc516b4d7982f8aef2ce4bcd4d32&__timestamp=1599165421products-render-info


URL examples:

- REST - `?___version=`
- category - `?update_id=`


Preview from admin panel:

- https://m24ee.local/admin/staging/update/preview/key/.../?preview_store=default&preview_url=https:/m24ee.local/&preview_version=1599116400
  - header and iframe
- https://m24ee.local/?___version=1599116400&___store=default&__timestamp=1599077245&__signature=a8f3a05adb92374873a3f9c4e1d36c1bddcc9748b32b046af4264b496243f95d


\Magento\Staging\Plugin\Framework\App\FrontController::beforeDispatch - validate signature:

```
__signature = hash(sha256, "__version,__timestamp", secret key from env.php)
```


## Set current version

\Magento\CatalogStaging\Model\Category\DataProvider::getCurrentCategory

```
$updateId = (int) $this->request->getParam('update_id');
$this->versionManager->setCurrentVersionId($update->getId());
```


## checkout-staging

- disable place order in preview
- any quote, saved in preview mode, is saved in db table `quote_preview` and deleted by cron
