# Evolution of Cloud Infrastructure:
  - before 2000s -> companies was writting the software + deploy it on their servers and manage all the infrastructure by themselfs where it was cumbersome process
  - after 2000s -> companies start builiding cloud and now it's easy for us and any company to run their software by a click button.
   
  Key advantage of Cloud Infrastructure:

      -  Available via Apis
      - scale up or down based on demand
      - now everything is not traditional , now no need for old data centers as the cloud provide the most new technology.
      - ....

Infrastructure as Code overview?

    three types you can access through the cloud;
      - cloud interface -> using the web console to provision your cloud
      - command line interface (cli) to interact with cloud (make it easy to manage it)
      - IAC (infrastructure as code) -> define your cloud through IAC (this is the best one)
Categories of IAC?

    -Configuration management tools -> like ansible
    - Ad-hoc=> basic API to make simple provisions in cloud using Api calls.
    - orchestration tools -> kubernetes -> deploying apps + managing containers.
    - server templating tools -> Amis(amazon machine images)
    - provisioning tools -> focus on provisioning the cloud using declarative approach.
    
What are the difference between declaritive,imperatiive tools?

    -declaritive ->define your end-state by using some cloud resourses(one server,5 load balancer) till get to this end state like terraform
    - Imparitive -> define the sequence of action to create the desired infrastructure.
What are the difference between specific ,cloud-agnostic tools?

    - specific tools-> like AWS,Azure these are using to provision the Infrastructure for you.
    -Agnostic tools -> like Terraform ,Pulumi -> can be used across any cloud provider in order to interact with these resources,control them.
    
