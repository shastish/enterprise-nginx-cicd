# Enterprise Nginx CI/CD

An Nginx static site delivered through Jenkins, Docker, Trivy, Amazon ECR, Helm, and Amazon EKS.

## Pipeline

1. Check out the selected Git branch.
2. Validate and render the Helm chart.
3. Scan source files and the built container image with Trivy. Critical and high findings fail the build; unfixed vulnerabilities are ignored.
4. Build an unprivileged Nginx image tagged with the 12-character Git commit SHA.
5. On `main`, optionally push the scanned image to ECR.
6. On `main`, optionally deploy the immutable image with Helm and verify rollout and the Nginx health endpoint.

Publishing and deployment are opt-in Jenkins parameters. AWS credentials are never stored in this repository; Jenkins must use an instance profile or another short-lived AWS identity.

## Local checks

Install Docker, Helm, and Trivy, then run:

```sh
helm lint helm/nginx
helm template nginx-site helm/nginx --namespace nginx
docker build --tag enterprise-nginx-cicd:local .
trivy fs --scanners vuln,misconfig,secret --severity CRITICAL,HIGH --exit-code 1 --ignore-unfixed .
trivy image --severity CRITICAL,HIGH --exit-code 1 --ignore-unfixed enterprise-nginx-cicd:local
```

## Jenkins setup

Create a **Multibranch Pipeline** job for this repository and use `Jenkinsfile` as the script. The build agent needs Git, Docker, Helm, Trivy, AWS CLI v2, and kubectl. Docker daemon access should be limited to trusted build agents.

To publish or deploy, select `PUBLISH_TO_ECR` and/or `DEPLOY_TO_EKS` when building the `main` branch. Deployment requires:

- An ECR repository named `enterprise-nginx-cicd` with immutable tags enabled.
- An EKS cluster in the selected Region.
- A non-root Jenkins AWS identity with least-privilege ECR push permissions and EKS access restricted to the `nginx` namespace.
- Kubernetes access for the Jenkins identity, plus Helm and kubectl on the agent.

The pipeline intentionally does not create AWS resources. EKS, worker capacity, VPC/NAT, and public load balancers can incur ongoing charges. Provisioning is pending a valid non-root AWS identity and review of the target cluster configuration.

## GitHub webhook

Do not configure a webhook to the current Jenkins HTTP URL. First enable HTTPS with a trusted certificate and a webhook secret; then configure the GitHub webhook to the secured Jenkins endpoint. Until then, trigger a branch scan/build from Jenkins.
