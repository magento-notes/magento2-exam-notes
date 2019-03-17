# Utilize ACL to set menu items and permissions

## Describe how to set up a menu item and permissions.

menu.xml - `urn:magento:module:Magento_Backend:/etc/menu.xsd` - flat structure
```xml
<menu>
    <add id="Module_Alias::item_name" title="Title" translate="title" module="Module_Alias" sortOrder="30"
      parent="Magento_Backend::system_design" action="adminhtml/system_design" resource="Magento_Backend::schedule"
      dependsOnConfig="" dependsOnModule="" sortOrder="" target="" toolTip=""
    />
    <update id="Module_Alias::item_name" [all normal params] />
    <remove id="Module_Alias::item_name" />
</menu>
```

*How would you add a new tab in the Admin menu?*
- don't specify parent and action

*How would you add a new menu item in a given tab?*
- just set one of top level parents, e.g. 'Magento_Backend::system'

*How do menu items relate to ACL permissions?*

## Describe how to check for permissions in the permissions management tree structures. 
*How would you add a new user with a given set of permissions?*

- System > Permissions > User Roles has all the differnet roles and associated permissions.  Each role can be scoped to Website level and granular permissions based on resource options. 
- System > Permissions > All Users to view and create new users and associate to a role.  There are two tabs 1 is user info the other is User Role where you define the Role for this user.  You can only select 1 role per user.
    
*How can you do that programmatically?*
- You can leverage [\Magento\Authorization\Model\Acl\AclRetriever](https://github.com/magento/magento2/blob/2.2-develop/app/code/Magento/Authorization/Model/Acl/AclRetriever.php).  That as a few methods that will help

```php
    /**
     * Get a list of available resources using user details
     *
     * @param string $userType
     * @param int $userId
     * @return string[]
     * @throws AuthorizationException
     * @throws LocalizedException
     */
    public function getAllowedResourcesByUser($userType, $userId)
    {
        if ($userType == UserContextInterface::USER_TYPE_GUEST) {
            return [self::PERMISSION_ANONYMOUS];
        } elseif ($userType == UserContextInterface::USER_TYPE_CUSTOMER) {
            return [self::PERMISSION_SELF];
        }
        try {
            $role = $this->_getUserRole($userType, $userId);
            if (!$role) {
                throw new AuthorizationException(
                    __('We can\'t find the role for the user you wanted.')
                );
            }
            $allowedResources = $this->getAllowedResourcesByRole($role->getId());
        } catch (AuthorizationException $e) {
            throw $e;
        } catch (\Exception $e) {
            $this->logger->critical($e);
            throw new LocalizedException(
                __(
                    'Something went wrong while compiling a list of allowed resources. '
                    . 'You can find out more in the exceptions log.'
                )
            );
        }
        return $allowedResources;
    }

    /**
     * Get a list of available resource using user role id
     *
     * @param string $roleId
     * @return string[]
     */
    public function getAllowedResourcesByRole($roleId)
    {
        $allowedResources = [];
        $rulesCollection = $this->rulesCollectionFactory->create();
        $rulesCollection->getByRoles($roleId)->load();
        $acl = $this->aclBuilder->getAcl();
        /** @var \Magento\Authorization\Model\Rules $ruleItem */
        foreach ($rulesCollection->getItems() as $ruleItem) {
            $resourceId = $ruleItem->getResourceId();
            if ($acl->has($resourceId) && $acl->isAllowed($roleId, $resourceId)) {
                $allowedResources[] = $resourceId;
            }
        }
        return $allowedResources;
    } 
```

However the actual code to set privilage permission may look like this in the core code
vendor/magento/magento2-base/setup/src/Magento/Setup/Fixtures/AdminUsersFixture.php

In particular this section:

```php
            $adminUser = $this->userFactory->create();
            $adminUser->setRoleId($role->getId())
                ->setEmail('admin' . $i . '@example.com')
                ->setFirstName('Firstname')
                ->setLastName('Lastname')
                ->setUserName('admin' . $i)
                ->setPassword('123123q')
                ->setIsActive(1);
            $adminUser->save();
```            

```php
<?php
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */

namespace Magento\Setup\Fixtures;

use Magento\Authorization\Model\Acl\Role\Group;
use Magento\Authorization\Model\RoleFactory;
use Magento\Authorization\Model\RulesFactory;
use Magento\Authorization\Model\UserContextInterface;
use Magento\Framework\Acl\RootResource;
use Magento\User\Model\ResourceModel\User\CollectionFactory as UserCollectionFactory;
use Magento\User\Model\UserFactory;

/**
 * Generate admin users
 *
 * Support the following format:
 * <!-- Number of admin users -->
 * <admin_users>{int}</admin_users>
 */
class AdminUsersFixture extends Fixture
{
    /**
     * @var int
     */
    protected $priority = 5;

    /**
     * @var UserFactory
     */
    private $userFactory;

    /**
     * @var RoleFactory
     */
    private $roleFactory;

    /**
     * @var UserCollectionFactory
     */
    private $userCollectionFactory;

    /**
     * @var RulesFactory
     */
    private $rulesFactory;

    /**
     * @var RootResource
     */
    private $rootResource;

    /**
     * @param FixtureModel $fixtureModel
     * @param UserFactory $userFactory
     * @param UserCollectionFactory $userCollectionFactory
     * @param RoleFactory $roleFactory
     * @param RulesFactory $rulesFactory
     * @param RootResource $rootResource
     */
    public function __construct(
        FixtureModel $fixtureModel,
        UserFactory $userFactory,
        UserCollectionFactory $userCollectionFactory,
        RoleFactory $roleFactory,
        RulesFactory $rulesFactory,
        RootResource $rootResource
    ) {
        parent::__construct($fixtureModel);
        $this->userFactory = $userFactory;
        $this->roleFactory = $roleFactory;
        $this->userCollectionFactory = $userCollectionFactory;
        $this->rulesFactory = $rulesFactory;
        $this->rootResource = $rootResource;
    }

    /**
     * {@inheritdoc}
     */
    public function execute()
    {
        $adminUsersNumber = $this->fixtureModel->getValue('admin_users', 0);
        $adminUsersStartIndex = $this->userCollectionFactory->create()->getSize();

        if ($adminUsersStartIndex >= $adminUsersNumber) {
            return;
        }

        $role = $this->createAdministratorRole();

        for ($i = $adminUsersStartIndex; $i <= $adminUsersNumber; $i++) {
            $adminUser = $this->userFactory->create();
            $adminUser->setRoleId($role->getId())
                ->setEmail('admin' . $i . '@example.com')
                ->setFirstName('Firstname')
                ->setLastName('Lastname')
                ->setUserName('admin' . $i)
                ->setPassword('123123q')
                ->setIsActive(1);
            $adminUser->save();
        }
    }

    /**
     * {@inheritdoc}
     */
    public function getActionTitle()
    {
        return 'Generating admin users';
    }

    /**
     * {@inheritdoc}
     */
    public function introduceParamLabels()
    {
        return [
            'admin_users' => 'Admin Users'
        ];
    }

    /**
     * Create administrator role with all privileges.
     *
     * @return \Magento\Authorization\Model\Role
     */
    private function createAdministratorRole()
    {
        $role = $this->roleFactory->create();
        $role->setParentId(0)
            ->setTreeLevel(1)
            ->setSortOrder(1)
            ->setRoleType(Group::ROLE_TYPE)
            ->setUserId(0)
            ->setUserType(UserContextInterface::USER_TYPE_ADMIN)
            ->setRoleName('Example Administrator');
        $role->save();

        /** @var \Magento\Authorization\Model\Rules $rule */
        $rule = $this->rulesFactory->create();
        $rule->setRoleId($role->getId())
            ->setResourceId($this->rootResource->getId())
            ->setPrivilegies(null)
            ->setPermission('allow');
        $rule->save();

        return $role;
    }
}

```
