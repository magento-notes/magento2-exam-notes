# Customer Segments

Conditions:

- customer address attributes
- customer attributes
- cart item, total qty, totals
- products in cart, in wishlist, viewed products, ordered products, viewed date, ordered date
- order address, totals, number of orders, average total amount, order date, order status - only logged in

Listens all model evetns, matches conditions by \Magento\CustomerSegment\Model\Segment\Condition\Order\Address::getMatchedEvents


## Database

- magento_customersegment_segment - name, conditions, actions
- magento_customersegment_website - many-to-many
- magento_customersegment_customer - website, added/updated date
- magento_customersegment_event - after saving segment rule, holds events that related to selected conditions


## Admin conditions

namespace  Magento\CustomerSegment;

Admin form - old widget style tabs:

- Block\Adminhtml\Customersegment\Edit\Tab\General
- Block\Adminhtml\Customersegment\Edit\Tab\Conditions

Rendering conditions form:

render conditions -> renderer \Magento\Rule\Block\Conditions -> segment.getConditions -> root Model\Segment\Condition\Combine\Root


Condition classes - hold matching rules:

- Model\Condition\Combine\AbstractCombine
- Model\Segment\Condition\Order\Address - order_address join sales_order join customer_entity
- Model\Segment\Condition\Sales\Combine
  - Model\Segment\Condition\Sales\Ordersnumber - COUNT(*)
  - Model\Segment\Condition\Sales\Salesamount - sum/avg sales_order.base_grand_total
  - Model\Segment\Condition\Sales\Purchasedquantity - sum/avg sales_order.total_qty_ordered
- Model\Segment\Condition\Customer\Address
  - Model\Segment\Condition\Customer\Address\DefaultAddress


## Refresh Segment Data

Some conditions don't work for guest visitors, despite description! Example - cart grand total amount:

- click "Refresh" inside segment admin
- Controller\Adminhtml\Report\Customer\Customersegment\Refresh::execute
- Model\Segment::matchCustomers
- Model\ResourceModel\Segment::aggregateMatchedCustomers
- Model\ResourceModel\Segment::processConditions
- Model\Segment\Condition\Combine\Root::getSatisfiedIds
- // individual conditions
- Model\Segment\Condition\Shoppingcart\Amount::getSatisfiedIds

```SQL
SELECT `quote`.`customer_id` FROM `quote`
WHERE (quote.is_active=1) AND (quote.store_id IN('1')) AND (quote.base_grand_total >= '100') AND (customer_id IS NOT NULL)
```


## Viewed product index

Condition by recently viewed events **SEEMS BROKEN** when full page cache is enabled.
This condition relies on `report_viewed_product_index` table, which is handled by report module.

Configuration:

- `reports/options/enabled`
- `reports/options/product_view_enabled`

\Magento\Reports\Model\ReportStatus::isReportEnabled

\Magento\Reports\Model\Product\Index\Viewed

`catalog_controller_product_view` -> \Magento\Reports\Observer\CatalogProductViewObserver
- save into  `report_viewed_product_index`
- save into `report_event`, take customer from session

This does not work when full page cache is enabled and Varnish returns HTML directly.


## Listening to events and updating matched customers

Segments needs to listen to some events related to conditions.

After segment with conditions is saved, table `magento_customersegment_event` holds events that it listens to:

- Model\Segment::beforeSave
- Model\Segment::collectMatchedEvents
- recursively for each added conditions children:
  - Model\Condition\Combine\AbstractCombine::getMatchedEvents


Segment rule table holds rendered `condition_sql`:

- Model\Segment::beforeSave
- Model\Rule\Condition\Combine\Root::getConditionsSql
- recursively for each added conditions children:
  - Model\Condition\Combine\AbstractCombine::getConditionsSql
  - `_prepareConditionsSql`


`condition_sql` column placeholders:
- :customer_id
- :website_id
- :quote_id
- :visitor_id

Updating customer segments on events:

- event `customer_login`
- Observer\ProcessEventObserver::execute
- Model\Customer::processEvent
- Model\Customer::getActiveSegmentsForEvent
- Model\ResourceModel\Segment\Collection::addEventFilter - join table `magento_customersegment_event`
- Model\Customer::_processSegmentsValidation
- Model\Segment::validateCustomer
  * visitor_id, quote_id passed for guest
- Model\Segment\Condition\Combine\Root::isSatisfiedBy
  * all conditions[].isSatisfiedBy
  * all conditions[].Model\Condition\Combine\AbstractCombine::getConditionsSql


Result:

- Matched segments are added to `magento_customersegment_customer`
- http context \Magento\CustomerSegment\Helper\Data::CONTEXT_SEGMENT = 'customer_segment' is updated with new segments
- customerSession.setCustomerSegmentIds


Event -> interested segment rules.


## Impact on performance

1. Customer segments are added to full page cache keys

    \Magento\CustomerSegment\Helper\Data::CONTEXT_SEGMENT = 'customer_segment'

    \Magento\PageCache\Model\App\Response\HttpPlugin::beforeSendResponse
    \Magento\Framework\App\Response\Http::sendVary
    \Magento\Framework\App\Http\Context::getVaryString

    Example:

    ```php
    sha1($this->serializer->serialize([
        'customer_group' => '1',
        'customer_logged_in' => true,
        'customer_segment' => ["1"]
    ]))
    ```

    This lowers chances of hitting cached page, increasing load on the server.

1. Listens to many object save events, calculates matching conditions


## DepersonalizePlugin

Does 2 things:

1. Makes sure customerSession.CustomerSegmentIds is not deleted by Customer module DepersonalizePlugin
1. Sets full page cache key httpContext->setValue(Data::CONTEXT_SEGMENT)


\Magento\Framework\Pricing\Render\Layout::loadLayout:
```
        $this->layout->getUpdate()->load();
          // 1. \Magento\Customer\Model\Layout\DepersonalizePlugin::beforeGenerateXml:
          // - remember customerGroupId and formKey from session
          // 2. \Magento\CustomerSegment\Model\Layout\DepersonalizePlugin::beforeGenerateXml:
          // - remember customerSegmentIds from customer session
        $this->layout->generateXml();
        $this->layout->generateElements();
          // 3. \Magento\PageCache\Model\Layout\DepersonalizePlugin::afterGenerateElements:
          // - event `depersonalize_clear_session`
          // - session_write_close();
          // - clear message session
          // 4. \Magento\Customer\Model\Layout\DepersonalizePlugin::afterGenerateElements:
          // - clear customer session
          // - restore session._form_key
          // - restore customer session.customerGroupId, empty customer object with customer group
          // 5. \Magento\Catalog\Model\Layout\DepersonalizePlugin::afterGenerateElements:
          // - clear catalog session
          // 6. \Magento\Persistent\Model\Layout\DepersonalizePlugin::afterGenerateElements:
          // - clear persistent session
          // 7. \Magento\CustomerSegment\Model\Layout\DepersonalizePlugin::afterGenerateElements:
          // - httpContext->setValue(Data::CONTEXT_SEGMENT)
          // - restore customerSession->setCustomerSegmentIds
          // 8. \Magento\Checkout\Model\Layout\DepersonalizePlugin::afterGenerateElements:
          // - clear checkout session
```

## Create segment programmatically

Doesn't have API interfaces, repository.

```PHP
/** @var \Magento\CustomerSegment\Model\Segment $segment */
$segment = $this->segmentFactory->create();
$segment->setName($name);
$segment->setDescription(null);
$segment->setIsActive(1);
$segment->setWebsiteIds(array_keys($this->storeManager->getWebsites()));
$segment->setApplyTo(\Magento\CustomerSegment\Model\Segment::APPLY_TO_VISITORS_AND_REGISTERED);
// @see \Magento\Rule\Model\AbstractModel::loadPost
$segment->getConditions()->setConditions([])->loadArray([
    'type' => \Magento\CustomerSegment\Model\Segment\Condition\Combine\Root::class,
    'aggregator' => 'all',
    'operator' => '',
    'value' => '1',
    'conditions' => [
        [
            'type' => \Magento\CustomerSegment\Model\Segment\Condition\Shoppingcart\Amount::class,
            'attribute' => 'grand_total',
            'operator' => '>=',
            'value' => '100',
        ]
    ],
]);
$segment->save();
```


## Catalog frontend action

Magento has alternative implementation of recently viewed products, handled by Catalog module.
It works with full page cache, but:

- is **not integrated** into customer segment conditions
- frontend synchronization must be enabled, adding extra calls on every page

When sync is enabled, it makes 2 AJAX calls with every page opening:
- synchronize recently viewed products from local storage
- synchronize recently compared products from local storage

Configuration:

- `catalog/recently_products/synchronize_with_backend` = 0
- `catalog/recently_products/recently_viewed_lifetime` = 1000
- `catalog/recently_products/scope` - store view, store, website

Other:

- db table `catalog_product_frontend_action`
- cron `catalog_product_frontend_actions_flush` - every minute \Magento\Catalog\Cron\FrontendActionsFlush:
- deletes compared, viewed older than (default 1000) seconds
- frontend controller 'catalog/product_frontend_action/synchronize'

Notable classes:

- Magento\Catalog\Model\FrontendStorageConfigurationPool
- Magento\Catalog\Model\Widget\RecentlyViewedStorageConfiguration


\Magento\Catalog\Block\FrontendStorageManager:

- recently_viewed_product
- recently_compared_product
- product_data_storage
- vendor/magento/module-catalog/view/frontend/web/js/storage-manager.js

customer data section:

- recently_viewed_product
- recently_compared_product
- product_data_storage

Triggering recently viewed update on frontend:

- catalog_product_view.xml
- Magento\Catalog\Block\Ui\ProductViewCounter
- vendor/magento/module-catalog/view/frontend/templates/product/view/counter.phtml
- vendor/magento/module-catalog/view/frontend/web/js/product/view/provider.js

\Magento\Catalog\Model\ProductRender - DTO

\Magento\Catalog\Ui\DataProvider\Product\ProductRenderCollectorComposite::collect:

- Url - url, addToCartButton, addToCompareButton
- AdditionalInfo - isSalable, type, name, id
- Image - images
- Price - priceInfo
- ... wishlist, review, tax, gift card, bundle price


widget `catalog_recently_compared` template `product/widget/compared/list.phtml`:

- UI component `widget_recently_compared` - listing, columns
- Magento_Catalog/js/product/provider-compared.js
- Magento_Catalog/js/product/storage/storage-service.js
- Magento_Catalog/js/product/storage/data-storage.js - get product data from customer section. recently viewed, compared
