# Demonstrate ability to use EAV entity load and save

Types of resource models:
- flat resource - `Magento\Framework\Model\ResourceModel\AbstractResource`
- EAV entity resource - `Magento\Eav\Model\Entity\AbstractEntity`
- version controll aware EAV entity resource - `Magento\Eav\Model\Entity\VersionControl\AbstractEntity`.
  Used only by customer and customer address.

New Entity Manager stands aside, you don't need to extend your resource model from it,
but rather manually call its methods in your resource - load, save, delete.
\Magento\Framework\EntityManager\EntityManager is suitable for both flat and EAV tables.

## Entity Manager

Entity manager is a new way of saving both flat tables and EAV entities.
Separates individual operations into own class - read, create, update, delete.
This give a lot of freedom for extensions.

Each operation still has a lot to do - save EAV data and run extensions. Default operations
split these activities into smaller actions:
- save main table - both for flat and EAV
- save EAV attributes - only for EAV entities. see *attribute pool*
- run extensions - see *extension pool*

Entity manager is concept separate from regular flat and EAV resource models saving, it doesn't
call them under the hood. It does the same things but in own fashion with extensibility in mind.

This means that you can still build your code on regular resource models:
- flat resource Magento\Framework\Model\ResourceModel\Db\AbstractDb
- EAV entity resource Magento\Eav\Model\Entity\AbstractEntity

Entity manager is currently used by:
- bundle option
- bundle selection
- catalog rule
- product
- category
- cms block
- cms page
- sales rule
- gift card amount
- and some others (staging etc.)

Terms:
- *entity manager* - calls appropiate operations. Holds type resolver and operations pool
- *metadata pool* - DI injectable. Register Api Data interface and [`entityTableName`, `identifierField`]
    ```xml
    <item name="Magento\Cms\Api\Data\PageInterface" xsi:type="array">
        <item name="entityTableName" xsi:type="string">cms_page</item>
        <item name="identifierField" xsi:type="string">page_id</item>
        <!-- eavEntityType - flag is eav -->
        <!-- connectionName -->
        <!-- sequence -->
        <!-- entityContext -->
    </item>
    ```

- *operation pool* - `checkIfExists`/`read`/`create`/`update`/`delete` operations per entity type. Operaton does ALL work.
    *Default operations - can extend in DI:*
    - checkIfExists - Magento\Framework\EntityManager\Operation\CheckIfExists - decides if entity is new or updated
    - read - Magento\Framework\EntityManager\Operation\Read
    - create - Magento\Framework\EntityManager\Operation\Create
    - update - Magento\Framework\EntityManager\Operation\Update
    - delete - Magento\Framework\EntityManager\Operation\Delete

- *attribute pool* - read/create/update attributes in separate tables per entity type. DI injectable, has default, register only to override.
    ```xml
    <item name="Magento\Catalog\Api\Data\CategoryInterface" xsi:type="array">
        <item name="create" xsi:type="string">Magento\Catalog\Model\ResourceModel\CreateHandler</item>
        <item name="update" xsi:type="string">Magento\Catalog\Model\ResourceModel\UpdateHandler</item>
    </item>
    ```
    *Default attribute pool actions - Module_Eav/etc/di.xml:*
    - read - Magento\Eav\Model\ResourceModel\ReadHandler
    - create - Magento\Eav\Model\ResourceModel\CreateHandler
    - update - Magento\Eav\Model\ResourceModel\UpdateHandler

- *extension pool* - custom modifications on entity read/create/update. Put your extensions here, good for extension attributes
  + product gallery read/create
  + product option read/save
  + product website read/save
  + configurable product links read/save
  + category link read/save
  + bundle product read/save - set options
  + downloadable link create/delete/read/update
  + gift card amount read/save
  + cms block, cms page - store read/save


### *Entity manager.save:*
- resolve entity type - implementing data interface `\Api\Data` when possible, or original class
- check `has(entity)` - detect create new or update
  * operation pool.getOperation(`checkIfExists`)
- if new, operation pool.getOperation(`create`).execute
- if existing, operation pool.getOperation(`update`).execute
- run callbacks

### `create` operation
- event `entity_manager_save_before`
- event `{$lower_case_entity_type}_save_before`
- apply sequence - \Magento\Framework\EntityManager\Sequence\SequenceApplier::apply
  * entity[id] = sequence.getNextValue
- *create main*
  * EntityManager\Db\CreateRow::execute - insert into main table by existing column
- *create attributes*
  * attribute pool actions `create`[].execute
  * default action handler `Magento\Eav\Model\ResourceModel\CreateHandler`:
    + only if entity is EAV type, insert all attributes
      - \Magento\Eav\Model\ResourceModel\AttributePersistor::registerInsert
      - \Magento\Eav\Model\ResourceModel\AttributePersistor::flush
- *create extensions* - \Magento\Framework\EntityManager\Operation\Create\CreateExtensions
  * extension pool actions `create`[].execute - save many-to-many records, custom columns etc.
- event `{$lower_case_entity_type}_save_after`
- event `entity_manager_save_after`


## Object converters
\Magento\Framework\Reflection\DataObjectProcessor::buildOutputDataArray(object, interface)
\Magento\Framework\Reflection\CustomAttributesProcessor::buildOutputDataArray
\Magento\Framework\Reflection\ExtensionAttributesProcessor::buildOutputDataArray


\Magento\Framework\Api\\`SimpleDataObjectConverter`:
- toFlatArray:
  * data object processor.buildOutputDataArray
  * ConvertArray::toFlatArray
- convertKeysToCamelCase - used by Soap API, supports `custom_attributes`

\Magento\Framework\Api\\`ExtensibleDataObjectConverter`:
- toNestedArray(entity, skip custom attributes list, interface) - used by entity repositories.save:
  * data object processor.buildOutputDataArray
  * add `custom_attributes`
  * add `extension_attributes`
- toFlatArray:
  * toNestedArray
  * ConvertArray::toFlatArray
- (static) convertCustomAttributesToSequentialArray


\Magento\Framework\Reflection\DataObjectProcessor::buildOutputDataArray:
- \Magento\Framework\Reflection\MethodsMap::getMethodsMap - method name and getter return types
- filter only getters: is..., has..., get...
- get data using interface getter, e.g. $object->getSku() - based on Interface definition
- deduce field name by getter name, e.g. 'sku'
- process custom_attributes \Magento\Framework\Reflection\CustomAttributesProcessor::buildOutputDataArray
- process extension_attributes \Magento\Framework\Reflection\ExtensionAttributesProcessor::buildOutputDataArray
- process return object: build return value objects with their return type annotation
- process return array: cast each element to type, e.g. int[] => (int) each value
- cast element to type

Product.save:
- buildOutputDataArray
- product data = call all getters + original data
- product = initializeProductData. create new product/get existing by SKU. set product data
- process links, unless product data `ignore_links_flag`
- abstract entity.validate
- product resource.save
- entity manager.save
