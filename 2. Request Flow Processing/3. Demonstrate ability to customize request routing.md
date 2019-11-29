# 2.3 Demonstrate ability to customize request routing

## Describe request routing and flow in Magento.

[Frontend routers](https://devdocs.magento.com/guides/v2.2/extension-dev-guide/routing.html):

- robots (10)
- urlrewrite (20)
- standard (30) - module/controller/action
  * get modules by front name registered in routes.xml
  * find action class
  * if action not found in all modules, searches "noroute" action in *last* module

- cms (60)
- default (100) - noRouteActionList - 'default' noroute handler

Adminhtml routers:

- admin (10) - extends base router - module/controller/action, controller path prefix "adminhtml"
- default (100) - noRouteActionList - 'backend' noroute handler

Default router (frontend and backend):

- noRouteHandlerList. process
  + backend (10)
    *Default admin 404 page* "adminhtml/noroute/index" when requested path starts with admin route.

  + default (100)
    *Default frontend 404 page* "cms/noroute/index" - admin config option `web/default/no_route`.

- always returns forward action - just mark request not dispatched - this will continue match loop


### When is it necessary to create a new router or to customize existing routers?

Create new router when URL structure doesn't fit into module/controller/action template,
e.g. fixed robots.txt or dynamic arbitraty rewrites from DB.

If you want to replace controller action, you can register custom module in controller lookup sequence -
reference route by ID and add own module "before" original module.

### How do you handle custom 404 pages?

1. If [front controller](https://github.com/magento/magento2/blob/2.3/lib/internal/Magento/Framework/App/FrontController.php#L61-L65) catches [\Magento\Framework\Exception\NotFoundException](https://github.com/magento/magento2/blob/2.3/lib/internal/Magento/Framework/Exception/NotFoundException.php), it changes action name *"noroute"* and continues loop.
   E.g. catalog/product/view/id/1 throws NotFoundException. catalog/product/noroute is checked.

1. If standard router recognizes front name but can't find controller, it tries to find *"noroute"*
   action from last checked module.
   E.g. catalog/brand/info controller doesn't exist, so catalog/brand/noroute will be checked.
   
   [\Magento\Framework\App\Router\Base::getNotFoundAction](https://github.com/magento/magento2/blob/2.3/lib/internal/Magento/Framework/App/Router/Base.php#L237)

1. If all routers didn't match, default controller provides two opportunities:
  - set default 404 route in admin config `web/default/no_route` (see: [\Magento\Framework\App\Router\NoRouteHandler::process](https://github.com/magento/magento2/blob/2.3/lib/internal/Magento/Framework/App/Router/NoRouteHandler.php#L34))
  - register custom handler in [noRouteHandlerList](https://github.com/magento/magento2/blob/2.3/lib/internal/Magento/Framework/App/Router/NoRouteHandlerList.php):
    * backend (sortOrder: 10) [Magento\Backend\App\Router\NoRouteHandler](https://github.com/magento/magento2/blob/2.3/app/code/Magento/Backend/App/Router/NoRouteHandler.php#L44) -> `adminhtml/noroute/index`
    * default (sortOrder: 100) [Magento\Framework\App\Router\NoRouteHandler](https://github.com/magento/magento2/blob/2.3/lib/internal/Magento/Framework/App/Router/NoRouteHandler.php)
