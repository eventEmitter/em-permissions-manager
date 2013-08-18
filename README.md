# em-permissions-manager

*** not yet satble! ***

permissions middleware for ee-webservice. get, add & modify persmissions & roles for resources and users. requires a sessionmiddleware with the interface specified in em-sessions-manager.

permissisons are granted for roles ( which have inheritance ) on hierarchical resources. resources are organized hierarchical. 

if you set a persmission on the resource «com.eventemitter.sampleapp» all sub resources will inherit the permission. if a user with «get» permissions on that resource requests the resource «com.eventemitter.sampleapp.users.4395» get access is granted.

the permissions are stored on a key value base, inpendent of the storage backend used. if a permission for the «com.eventemitter.sampleapp.users.4395» object is laoded the storage is queried for every possible combination: «com», «com.eventemitter, ..., «com.eventemitter.sampleapp.users.4395». this may use many resources and cause significant cost. to avoid too much queries you shoud define not too deply nested resources. if the queries cause much cost ( which is the case if you are using the dynamo db backend ) you should us a caching layer like «em-permissions-cache-memcached» which caches the permissions once they are fetched from the storage ( permissions updates will remove the permission from the caching layer ).




	var   sessionsManager 		= new ( require( "em-sessions-manager" ) )
		, permissionsManager 	= new ( require( "em-permissions-manager" ) );


	// automatically load the permissions for every request. the function must return the hierarchical resource identifier
	// the autoloader will provide the «request.permission» object even if there isn't any permissions available for the
	// given resource ( deny all ).
	permissionsManager.autoload = function( request ){ 
		var resourceUri = request.pathname.split( "/" ).filter( function( item ){ return item.length > 0 } ).join( "." );
		return "com.eventemitter.sampleapp." + resourceUri; 
	}


	webservices.use( sessionsManager ); 	// provides the request.session object
	webservices.use( permissionsManager );	// provides the request.permissions object


	// your middleware
	webservices.use( function( request, response, next ){
		// the «request.permission» object is provided by the autolaoder. if you don't autoload ( which may a good thing
		// performance wise ) you have to load the permission using the «request.permissions.load» method. if the 
		// «request.session» object is missing everything is denied.
		if ( request.permission.allowed( request.method ) ){
			// ok
		}
		else response.send( "access_denied" );
	} );


	// you may always alter permissions using the permissionsManager instance or the reference of it, which is stored on each 
	// request. the laod method will return also inherited permissions. if you need the permissions for that exact resource
	// without inherited permissions you should use the «request.permissiosn.loadAtomic» method instead.
	request.permissions.load( "moderators", "com.eventemitter.sampleapp.messages", function( err, permission ){
		if ( err ) throw err;
		else if ( permission ){
			log.dir( permission.allowedActions );
		}
		else {
			var permission = new permissionsManager.permission();
			permission.role = "moderators";
			permission.resource = "com.eventemitter.sampleapp.messages";
			permission.allow( "get", "post", "put", "whatever" );
			permission.deny( "someRightGrantedByAnInheritedPermission" );
			permission.save();
		}
	} );


	// list all permissions for a given resource and or role. you should use the «permissionsManager.loadAtomic» method if you 
	// only want to get permissions fro the exact resource specified ( without inheritance ). you may provide null as role or 
	// resource if you want to get all permissions for a role or a resource. pay attention: this may cause to laod many  
	// objects depending on the number of permissions you created. this call will return a permissions object for each 
	// defined permission, also for inhertied permissions.
	permissionsManager.list( role, resource, function( err, permissions ){
		permissiosn.forEach( function( permission ){
			log.dir( permissions.allowedActions ) );
		} );
	} );