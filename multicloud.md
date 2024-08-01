# To multi cloud or not - it is always about the MONEY!!!!


### So why not go cloud agnostic?
Often as a Cloud Infra leader, the question asked is why not design the service to be cloud agnostic OR run on multiple clouds such as AWS or Azure or GCP or anything else. This may sound cool to do and also remove public cloud vendor lock in but that is NOT the reason to do this. The reason for saying this is that being cloud agnostic requires 

* Upskilling the team to learn multiple public clouds - human cost goes up 
* Differences for cost optimizationm, security & compliance
* Tool set choices shrink as not all support multiple clouds
* Adds a layer of complexity to operationalize

### Makes sense but surely there must be $$$ out there
YES - being cloud agnostic MUST be driven by real business aka $$$$$ use cases that justifies the cost to run your service on multiple public clouds. The most common business use cases would be:

* Your company has a go to market or sales partnership with 1 or more public cloud vendors that mandate using their own cloud
* Customers mandate use of a specific public cloud for competitive or other reasons i.e. most enterprise retail customers do not prefer to run on AWS
* The service uses best of breed technologies that are unique to specific public clouds such as BigQuery on GCP or Windows on Azure

### So can you still be cloud agnostic and scale?
In such cases, being cloud agnostic does make sense and to help make this leap, recommend using something like the list below, assuming you are cloud native:

* Infra as Code -> Terraform 
* CI -> Harness or Jenkins CI
* MicroServices -> Kubernetes (EKS, GKE, AKS etc)
* CD -> ArgoCD or Harness CD
* Relational DBs -> Postgres or MySQL (RDS, CloudSQL etc)
* NoSQL DBs -> Cassandra (Keyspaces, Azure Managed Cassandra etc)
* Observability -> Any vendor (Datadog, SumoLogic, Splunk, Honeycomb)

You may have noticed a pattern here - we are leveraging managed IaaS & PaaS for various technologies BUT the underlying technologies (MYSQL, Postgres, Cassandra, Redis, Kubernetes) are "portable" across the clouds with minor differences. Also 

Of course there maybe disagreements but this is "been there, done that" in a past life --> ended up running on multiple public clouds across 16 different cloud regions!!

### TL; DR
If you do not have a good business use case that is at least a $2M or higher annually then going multi cloud IMHO is overhead you can live without and the resources are better spent optimizing your current setup for scale, reliability & cost. The optimization on a specific public cloud will be the best investment for nimble and agile teams so they can focus on delivering value at the top or bottom line for the company.

