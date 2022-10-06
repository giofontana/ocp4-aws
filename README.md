
# How to install OpenShift using CCO in manual mode and STS

## Procedure

1. Create a policy using this roles json file: https://github.com/giofontana/ocp4-aws/blob/main/aws-iam-roles/aws-iam-roles.json
2. Create a user which uses this policy

        Username: ocp-admin
        AWS_ACCESS_KEY_ID: *********
        AWS_SECRET_ACCESS_KEY: *********

3. Create a policy using this roles json file: https://github.com/giofontana/ocp4-aws/blob/main/aws-iam-roles/aws-sts-manual-mode.json
4. Create a user which uses this policy

        Username: ocp-cco
        AWS_ACCESS_KEY_ID: *********
        AWS_SECRET_ACCESS_KEY: *********

4. Set AWS cli to use the **ocp-cco**

```
$ aws configure
```

5. Install CCO CLI

```
$ wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-install-linux.tar.gz
$ tar -xvzf openshift-install-linux.tar.gz

$ RELEASE_IMAGE=$(./openshift-install version | awk '/release image/ {print $3}')
$ CCO_IMAGE=$(oc adm release info --image-for='cloud-credential-operator' $RELEASE_IMAGE)

# Download pull-secret file to ~/.pull-secret
$ oc image extract $CCO_IMAGE --file="./ccoctl" -a ~/.pull-secret
$ sudo mv ./ccoctl /usr/bin/ccoctl
$ sudo chmod 775 ccoctl
$ ccoctl --help
```

6. Creating AWS resources

```
$ mkdir cco
$ cd cco

$ ccoctl aws create-identity-provider \
--name=ocp-sts-manual \
--region=us-east-2 \
--public-key-file=`pwd`/cco/serviceaccount-signer.public

$ oc adm release extract --credentials-requests \
--cloud=aws \
--to=`pwd`/cco/credrequests \
--from=$RELEASE_IMAGE

$ ccoctl aws create-iam-roles \
--name=ocp-sts-manual \
--region=us-east-2 \
--credentials-requests-dir=`pwd`/cco/credrequests \
--identity-provider-arn=arn:aws:iam::779980194408:oidc-provider/ocp-sts-manual-oidc.s3.us-east-2.amazonaws.com
```

7. Install OpenShift

- Set AWS cli to use the **ocp-admin** this time

```
$ aws configure
```

- Add `credentialsMode: Manual` snippet after the `baseDomain` line

```
$ cd ..
$ mkdir ocp
$ ./openshift-install create install-config --dir ocp/

# Add credentialsMode: Manual after the baseDomain line
$ vi ocp/install-config.yaml
```

- Create manifests

```
$ ./openshift-install create manifests --dir ocp/
```

- Copy the manifests and tls generated with ccoctl to install dir

```
$ cp `pwd`/cco/manifests/* ocp/manifests/
$ cp -a `pwd`/cco/tls ocp/
```

- Run the installer

```
$ ./openshift-install create cluster --dir ocp/
```

8. (Optional) Remove the users ocp-admin and ocp-cco, they are not required anymore.

9. Run a few checks to confirm that credentials are not stored and if cluster operators are using STS:

```
$ export KUBECONFIG=/home/gfontana/Temp/ocp-sts/ocp/auth/kubeconfig
$ oc get secrets -n kube-system aws-creds
Error from server (NotFound): secrets "aws-creds" not found

$ oc get secrets -n openshift-image-registry installer-cloud-credentials -o json | jq -r .data.credentials | base64 --decode
[default]
role_arn = arn:aws:iam::779980194408:role/ocp-sts-manual-openshift-image-registry-installer-cloud-credenti
web_identity_token_file = /var/run/secrets/openshift/serviceaccount/token

$ oc get secrets -n openshift-machine-api aws-cloud-credentials -o json | jq -r .data.credentials | base64 --decode
[default]
role_arn = arn:aws:iam::779980194408:role/ocp-sts-manual-openshift-machine-api-aws-cloud-credentials

# Run the following command and wait up to 5 minutes to see if the new nodes are properly spun up
$ oc get machineset -n openshift-machine-api --no-headers | awk '{print "oc -n openshift-machine-api scale --replicas=2 machineset "$1}' | sh
machineset.machine.openshift.io/ocp-qbvqn-worker-us-east-2a scaled
machineset.machine.openshift.io/ocp-qbvqn-worker-us-east-2b scaled
machineset.machine.openshift.io/ocp-qbvqn-worker-us-east-2c scaled

```

