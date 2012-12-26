## What is this?

This is a simple Rakefile that will deploy code to a production Elastic Load balancer.   It enbles seamless deploys because it fires up new app servers for each deploy,
which are verifiable before the load balancers cut over to the new app servers.  See more details in the example below.

## Prerequisites


*   Ruby 1.x
*   An amazon AWS account, and your production setup uses Amazon ELBs
*   You utilize chef to bring up your application servers.  Application servers get set up 100%, and are ready and listenting for requests when chef is done running.

## Deployment workflow

#### Pre-deployment

1.  `rake create_new_app_servers` will fire off chef to create new application servers.  
2.  N app servers will be created.  They will then be attached to the `TestLoadBalancer`
2.  These new app servers will be tagged with the value of `NewAppServerTag`.
3.  You can do final verifications of the deploy on the test LB.

#### Cutting over to the new app servers

1.  When ready, you will run `rake deploy`.
2.  This will perform the following steps:
    1.  Register the app servers tagged with `NewAppServerTag` onto the production load balancers (`ProductionLoadBalancers`).
    2.  Verify instance health
    3.  If all instances are healthy, the app servers tagged as `CurrentAppServerTag` will be removed from the `ProductionLoadBalancers`.
    4.  Removes the new app servers (tagged `NewAppServerTag`) from the test load balancer (`TestLoadBalancer`).

#### If everything went swell (of course it did!)

After you are positive that you will not be reverting back to the previous app servers, run `rake cleanup`.  This will do the following:

1.  Terminate the old app servers that were previously in production
2.  Tag the current instances as `CurrentAppServerTag`.


#### Houston, we got problems

If there were unforeseen problems and you MUST bail out immediately, run `rake rollback`.  This will do the following:

1.  Re-registers the old app servers (`CurrentAppServerTag`) back into the production load balancers (`ProductionLoadBalancers`)
2.  Verifies instance health
3.  When all app servers are healthy, it will then take out the new app servers (`NewAppServerTag`) from the production load balancers 
4.  Re-registeres the new app servers back into the test load balancer (`TestLoadBalancer`), so you can figure out what went wrong.


## Hints

Sometimes you want to sneak in a few deploys to the new app servers, before cutting production over to them.  It is helpful to setup Capistrano (or whatever you use to
deploy) to deploy to your new app servers.  

An easy way to get the DNS names of your new app servers is something like this:

    require 'rubygems'
    require 'aws-sdk'
    client = AWS::EC2.new(:access_key_id => '<YOUR_ACCESS_KEY>', :secret_access_key => '<YOUR_SECRET_KEY>')
    ip_addresses = client.instances.tagged_values('new-app-server').map(&:dns_name)
