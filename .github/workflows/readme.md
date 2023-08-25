## github action to build an image and publish to google artifact repository 

### authentication with google cloud
Authenticating with google cloud and exchanging github token for google cloud oauth access token is done as part of this github action.

At high level, it means:
1. A service account is created in google cloud (gc). It is assigned required roles. e.g. roles/artifactregistry.writer to publish to artifact repository.
1. Provide mapping of attributes in the github token to gc access token. these mappings are added to gc using workload pool identity pool and provider. 
1. workload identity based principal set (github identity) is assigned role ('roles/workloadIdentityUser'), to allow it to impersonate <serviceAccount> created in steps earlier.

Detailed steps are provided at [setting up workload identity federation](https://github.com/google-github-actions/auth#setting-up-workload-identity-federation)

steps are given along with example gcloud commands* below:
*commands will need to be adapted to ur environment (google project name, number, service account name, repository owner). 

1. set env variables. 
   
```bash
export P=my-gcp-project
export A=gkeops@$P.iam.gserviceaccount.com
export PN=$(gcloud projects describe $P --format='value(projectNumber)')
export O="<repository_owner>"
```

2. create service account. assign relevant roles.
```bash
gcloud iam service-accounts create gkeops
gcloud projects add-iam-policy-binding $P --member="serviceAccount:$A" --role="roles/artifactregistry.writer" 
```

3. create workload identity pool and provider 
```bash
gcloud iam workload-identity-pools create pool1 --project=$P --location=global --display-name=default
gcloud iam workload-identity-pools providers create-oidc "github-provider" --project="$P" --location="global" --workload-identity-pool=pool1 --display-name="github identity provider" --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" --issuer-uri="https://token.actions.githubusercontent.com"

```

4. Allow workload identity principal set to impersonate service account.
```bash
gcloud iam service-accounts add-iam-policy-binding $A --member="principalSet://iam.googleapis.com/projects/$PN/locations/global/workloadIdentityPools/pool1/attribute.repository_owner/$O" --role="roles/iam.workloadIdentityUser" --project=$P
```
### github action
github action is based on the template to build, publish and deploy to GKE. Deploy part is TBD. currently it does build and publishing to artifact repository. 

Few changes to note:
1. add `pull_request` trigger to test. it can be later removed once the action is well tested.
2. add `GKE_PROJECT` in repository settings as secret as [described in docs][2]
3. add `GKE_PROJECT_NUMBER` in repository settings as secret.
4. arguments for the google auth action namely workload identity provider and service account are updated.


[1]: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect
[2]: https://docs.github.com/en/actions/deployment/deploying-to-your-cloud-provider/deploying-to-google-kubernetes-engine
