# Harness deployment to Kubernetes 

In this lab, you will install a Harness delegator in your Kubernetes cluster and deploy a sample guestbook application. 

## Prerequisites

### Create a GitHub account
Create a free GitHub account if you do not already have one.
https://github.com/join

### Create a GitHub Personal Access Token (PAT)
Generate Personal Access Token to authenticate from Ansible to GitHub.
1. In a browser visit: [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. Log in to GitHub if prompted
3. Click "Generate new token"
4. From the dropdown, choose "Generate new token (beta)"
  5. Name the token
  6. Select "All repositories"
7. Click "Repository permissions"
8. For "Contents", choose "Read and write"
9. At the bottom click "Generate token"
10. Copy the token and save it somewhere. You will not be able to access it again.

## Sign up for a Harness account 

Sign up for a new [Harness account](https://app.harness.io/auth/#/signup/)

I recommend using your Citi email to create the new account. Harness has policies to prevent 'non-corporate' accounts from using the Free tier. 

After signing up, sign in to Harness and follow the [CD guide](https://app.harness.io/ng/account/2cDL9wS3RFuJ1_ROM1bg0Q/cd/orgs/default/projects/default_project/cd-onboarding-wizard)



The instructor is available to answer any of your questions. 