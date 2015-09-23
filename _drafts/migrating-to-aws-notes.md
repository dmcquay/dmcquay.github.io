RDS won't let you set timezone globally.

Load balancers don't have IPs like Rackspace does. This is a problem for root domains like steals.com b/c 
those cannot be CNAME records. They must point to an IP directly. Route 53 has a workaround for this 
specific situation, but that means you MUST move your DNS hosting to Route 53 as part of your migration.