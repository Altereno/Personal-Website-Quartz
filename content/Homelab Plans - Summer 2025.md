# Update
I have done the bare minimum to get it working:
*(These are in order)*
[[HL2025 - Kubernetes on Talos (p1)]]
[[HL2025 - Kubernetes on Talos (p2)]]
[[HL2025 - Gitea Setup]]
[[HL2025 - Woodpecker CI Setup]]
[[HL2025 - Argo CD Setup]]
# Goals
- Set up a Kubernetes cluster and actually deploy services on it
- Set up a CI/CD pipeline to deploy my website automatically
- Automations (expanded in the plans)
# Plan
## Kubernetes Cluster
In my previous attempt with Kubernetes, I had set up everything manually. Also the school semester and recruiting took up most of my time so I didn't actually launch any services on the cluster. The idea this time is to have automations do most if not all the heavy lifting.
- [Talos Linux image](https://www.talos.dev/) for [Kubernetes](https://kubernetes.io/)
- [Terraform](https://developer.hashicorp.com/terraform) using the [Proxmox provider](https://github.com/bpg/terraform-provider-proxmox) to create all the nodes
- Running services on the cluster
*Some of the configuration details of Talos have not been figured out completely, so I'll just wing it. Most likely I want to have it automatically configure itself using some file that it'll grab on boot*
## CI/CD Pipeline
I had a previous attempt with installing [GitLab](https://about.gitlab.com/) and the [GitLab Runners](https://docs.gitlab.com/runner/) to automate builds but it was on a test VM and I also abandoned the project due to a lack of time. The idea this time is to build out the pipeline completely once the cluster is formed.
- Revisit [Gitea](https://about.gitea.com/) to use for git and a container registry
- Use [Woodpecker CI](https://woodpecker-ci.org/) for building and pushing images
- Use [Quartz](https://github.com/jackyzha0/quartz) to generate static files from my docs written using [Obsidian](https://obsidian.md/)
- Use [Argo CD](https://argo-cd.readthedocs.io/en/stable/) to deploy
## Automations
Aside from trying out Kubernetes and building out a CI/CD pipeline, I wanted to make sure that this entire setup is able to be destroyed and completely rebuilt from ground up with the least amount of work.