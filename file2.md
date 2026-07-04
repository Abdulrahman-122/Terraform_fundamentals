# Benefits of using this tool?

        - cloud agnostic approach which are compatible with any cloud providers.
        - interact with any online service through API
        - utilize version control to track changes over time
        - ...
    
Terraform with other tools:

     -Terraform with (configuration management tool) Ansible:

         - Terraform provision vm
         - ansible install + configures dependencies inside vm.
    -Terraform with (templating tool(Packer)):

          - terraform provisions servers.
          - packer builds the images.
    -Terraform with orchestration tool(kubernetes):
          -terraform provisions clusters.
          - kubernetes defines how the application is deployed and managed on the cloud.
          

- Architecture of Terraform:

    <img width="1326" height="568" alt="image" src="https://github.com/user-attachments/assets/8a563142-a092-49bc-a5ef-2d1c86fdf3ee" />

        - Terraform core;  
            - engine ->define configuration file+state file.
            - interact with cloud provider -> make sure the current state match the desired one.
        -Terraform provider
            - plugin for terraform to allow it to interact with cloud provider.
            - map the state+provide configuration to the appropriate calls.
            - over 100 providers are available for cloud services.
            

        
