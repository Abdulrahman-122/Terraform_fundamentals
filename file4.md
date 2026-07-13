# understanding Terraform Providers + init command:
<img width="1326" height="568" alt="image" src="https://github.com/user-attachments/assets/09395481-cb61-4da9-a04d-7b81fc71010d" />

    
    - terraform init -> initialize your project
    - terraform plan -> checks your config against current state+ generates a plan
    - terraform apply -> apply your plan in order to create or update your infrastructure.
    - terraform destroy -> delete resouces.
    again;
      - terraform core -> provide engine for parsing (config,state files)
      - providers -> connect terraform with cloud services.

if you run ; terraform init  you directory will be as:
<img width="1200" height="1237" alt="image" src="https://github.com/user-attachments/assets/fc7935db-08b9-421b-8b55-2cce14950949" />

    - as you see these are generated modules that run when you start building


-----
Terraform State Management:


    -What is the state file?
      -file contain deployed resources on terraform.
      - it contain sensitive information.
      -so it must be encrypted or pretected.
  <img width="553" height="820" alt="image" src="https://github.com/user-attachments/assets/d4ffff63-8005-45cd-9ce4-579d3031d453" />

      - it's json format .
      -Types of storing this state file?
        1.Local storing -> it's by default stored on your direction.
        2.Remote storing -> using S3 instance or Terraform cloud...
        Advan,Disadvan of both ways:
          1.Local   
            pros:
              easy to set and use.
            cons;
              stores sensitive values in plain text.
              not good for collaboraion.
              need manual interaction.
        2.Remote:  
            pros;
            -contain sensitive data.
            -allow collaborations.
            - support automation through  CI/CD pipelines.
            cons:
              allow complexity.
-----
How commands Plan,Apply,Destroy work?

    1.plan Command:
      compare the states between the desired state and actual state.
      see the descrepancies
      output the differenc then reconcils the satates.
 <img width="600" height="287" alt="image" src="https://github.com/user-attachments/assets/fa384f7e-d0c9-4f47-878d-eb8f6a7c8dfb" />

     2.Apply commands:
       creates the actions in state file
       like resources +delete them to match desired sate
       update the terraform state file to reflect the changes
  <img width="1200" height="575" alt="image" src="https://github.com/user-attachments/assets/3a6745ce-ca8c-4f38-97e0-07c7a15371a8" />

    3.destroy commands:
      remove all resources that are using with terraform
      clean the resources in order to end the project.
  <img width="1200" height="577" alt="image" src="https://github.com/user-attachments/assets/36424b7a-4523-4282-9436-5cd214a9827c" />



  
     
  
