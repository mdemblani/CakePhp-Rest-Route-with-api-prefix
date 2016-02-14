# CakePhp-Rest-Route-with-api-prefix
A small hack in cakephp v2.X to support restful routes with api prefix

I had my api ready and faced the problem of converting them into rest routes with api and version prefixs using the cakephp Router::mapResources function that was provided by default by CakePhp, 

My api was intially,  
http://www.xyz.com/controller/   
supporting basic rest routes and I wanted it to be in the form  
http://www.xyz.com/api/version_number/controller/action   
that would support basic rest routes with prefix including api and version number.

To achieve this there are two ways:
1. Create routes using **Router::connect** for each function and list them down manually in your routes.php file.
2. The second options (which I used) is a small hack was made in the core cakephp2 lib *Router* class *mapResource* function located at  
**Path to CakePhp App Folder/lib/Cake/Routing/Router.php** as follows

Intially the function was as follows:
```php
/**
 * Creates REST resource routes for the given controller(s). When creating resource routes
 * for a plugin, by default the prefix will be changed to the lower_underscore version of the plugin
 * name. By providing a prefix you can override this behavior.
 *
 * ### Options:
 *
 * - 'id' - The regular expression fragment to use when matching IDs. By default, matches
 *    integer values and UUIDs.
 * - 'prefix' - URL prefix to use for the generated routes. Defaults to '/'.
 * - 'controller' - If an alias for the controller is being provided, then controller name(in plural and lowercase) should be specified 
 * - 'plugin' - If the controller specified is a plugin, then the name of the plugin should be provided.
 * 
 * @param string|array $controller A controller name or array of controller names (i.e. "Posts" or "ListItems")
 * @param array $options Options to use when generating REST routes
 * @return array Array of mapped resources
 */
	public static function mapResources($controller, $options = array()) {
		$hasPrefix = isset($options['prefix']);
		$options += array(
			'connectOptions' => array(),
			'prefix' => '/',
			'id' => self::ID . '|' . self::UUID
		);

		$prefix = $options['prefix'];
		$connectOptions = $options['connectOptions'];
		unset($options['connectOptions']);
		if (strpos($prefix, '/') !== 0) {
			$prefix = '/' . $prefix;
		}
		if (substr($prefix, -1) !== '/') {
			$prefix .= '/';
		}

		foreach ((array)$controller as $name) {
			list($plugin, $name) = pluginSplit($name);
			$urlName = Inflector::underscore($name);
			$plugin = Inflector::underscore($plugin);
			if ($plugin && !$hasPrefix) {
				$prefix = '/' . $plugin . '/';
			}

			foreach (self::$_resourceMap as $params) {
				$url = $prefix . $urlName . (($params['id']) ? '/:id' : '');

				Router::connect($url,
					array(
						'plugin' => !isset($options['plugin']) ? $plugin : $options['plugin'],
						'controller' => !isset($options['controller']) ? $urlName : $options['controller'],
						'action' => $params['action'],
						'[method]' => $params['method']
					),
					array_merge(
						array('id' => $options['id'], 'pass' => array('id')),
						$connectOptions,
                                                $options['param']
					)
				);
			}
			self::$_resourceMapped[] = $urlName;
		}
		return self::$_resourceMapped;
	}
```
	
to the following:  
	
```php
/**
 * Creates REST resource routes for the given controller(s). When creating resource routes
 * for a plugin, by default the prefix will be changed to the lower_underscore version of the plugin
 * name. By providing a prefix you can override this behavior.
 *
 * ### Options:
 *
 * - 'id' - The regular expression fragment to use when matching IDs. By default, matches
 *    integer values and UUIDs.
 * - 'prefix' - URL prefix to use for the generated routes. Defaults to '/'.
 * - 'controller' - If an alias for the controller is being provided, then controller name(in plural and lowercase) should be specified 
 * - 'plugin' - If the controller specified is a plugin, then the name of the plugin should be provided.
 * 
 * @param string|array $controller A controller name or array of controller names (i.e. "Posts" or "ListItems")
 * @param array $options Options to use when generating REST routes
 * @return array Array of mapped resources
 */
public static function mapResources($controller, $options = array()) {
	$hasPrefix = isset($options['prefix']);
	$options += array(
		'connectOptions' => array(),
		'prefix' => '/',
		'id' => self::ID . '|' . self::UUID
	);

	$prefix = $options['prefix'];
	$connectOptions = $options['connectOptions'];
	unset($options['connectOptions']);
	if (strpos($prefix, '/') !== 0) {
		$prefix = '/' . $prefix;
	}
	if (substr($prefix, -1) !== '/') {
		$prefix .= '/';
	}

	foreach ((array)$controller as $name) {
		list($plugin, $name) = pluginSplit($name);
		$urlName = Inflector::underscore($name);
		$plugin = Inflector::underscore($plugin);
		if ($plugin && !$hasPrefix) {
			$prefix = '/' . $plugin . '/';
		}

		foreach (self::$_resourceMap as $params) {
			$url = $prefix . $urlName . (($params['id']) ? '/:id' : '');

			Router::connect($url,
				array(
              //Router hack to support alias
					'plugin' => !isset($options['plugin']) ? $plugin : $options['plugin'],
					'controller' => !isset($options['controller']) ? $urlName : $options['controller'],
              //Router hack to support alias

              /**
               * Original Routes
               * 'plugin' => $plugin,
               * 'controller' => $urlName,
               */

					'action' => $params['action'],
					'[method]' => $params['method']
				),
				array_merge(
					array('id' => $options['id'], 'pass' => array('id')),
					$connectOptions,
                                              $options['param']
				)
			);
		}
		self::$_resourceMapped[] = $urlName;
	}
	return self::$_resourceMapped;
}
```
	
To use the new *mapResources*, in your **router.php** file you can use the function as follows:
	
```php
Router::mapResources('Controllername', array('prefix' => '/api/:apiVersion/', 'param' => array('api' => 'api', 'apiVersion' => 'v1|v2|')));
```
  
Furthermore if you would like to give an alias to the controller as follows:
All request for sheets should be handled by the controller tables with default rest routes, you can write mapResources in router.php as follows:
  
```php
Router::mapResources('sheets', array('prefix' => '/api/:apiVersion/', 'param' => array('api' => 'api', 'apiVersion' => 'v1|v2|'), 'controller' => 'tables'));
```	 
