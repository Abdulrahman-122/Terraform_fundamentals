There  are two  main approaches to  manage multiple environments  :

1. Terraform workspace
	- feature allows you  to manage multi environment each with separate config.
	- pros:
		- easy to start with.
		- minimize code duplication between environment.
		- you can use ; terraform workspace new env_name and it will be easy
	- cons:
		- can be dangerous if you apply wrong changes to different environment
		- state files are stored  in the  same backend so you will need to make permissions+access .
    - <img width="700" height="342" alt="image" src="https://github.com/user-attachments/assets/b753d11a-34c3-4662-ad0d-3d17676f9334" />
2. separate directories(the better one);
	- you divide the environments across multi directories.
	- Pros:
		- allow for  separate  back end configurations+improve  security.
		- decrease   potential error as environments are managed separately.
		
	- Cons:
		- requires separation between subdirectories in order to apply  changes to these environments.
		- more code dublication as main.tf is dublicated across envrionments .
  - <img width="700" height="441" alt="image" src="https://github.com/user-attachments/assets/84b1d80d-374d-42ac-9ad4-1ad73a9d456a" />
  - some good commands  to format your code in Terraform internally:
- static checks -> terraform fmt
	- terraform validate
	- terraform plan
- manual testing:
	- terraform apply --auto-approve
	- terraform init 
- Automatic testing;
	- use ;  bash script 
	- or use; some  robust methods in Go  like ;  Terratest or  whatever.
	
