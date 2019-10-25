## 10.1 Demonstrate ability to customize My Account

Describe how to customize the “My Account” section. 

*How do you add a menu item?*
- Create A theme or use an existing one
- Create a folder in the theme Magento_Customer
- Create a folder inside the theme Magento_Customer called layout 
- Create a file inside the theme/Magento_Customer/layout/customer_account.xml
- Add similar xml
```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */
-->
<page  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="customer_account_navigation">
            <block class="Magento\Customer\Block\Account\SortLinkInterface" name="customer-account-navigation-address-link">
                <arguments>
                    <argument name="label" xsi:type="string" translate="true">Russell Special</argument>
                    <argument name="path" xsi:type="string">special/link</argument>
                    <argument name="sortOrder" xsi:type="number">165</argument>
                </arguments>
            </block>
        </referenceBlock>
    </body>
</page>
```

*How would you customize the “Order History” page?*
