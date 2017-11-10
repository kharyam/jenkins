# SCAP Slave Image


This repository contains Dockerfiles for a Jenkins Slave Docker image intended for 
use with [OpenShift v3](https://github.com/openshift/origin). This slave can perform an OpenSCAP CVE scan of an image. See the sections below for details and examples.

## Configuration
Perform the following commands on an OpenShift node (as a cluster admin) to build and add this image to the OpenShift registry:

1. Build the docker image

       docker build -t scap-slave:latest -f Dockerfile.rhel7 .

2. Tag the image for the internal OpenShift Repository

       docker tag scap-slave:latest \
         docker-registry.default.svc.cluster.local:5000/openshift/scap-slave:latest

3. Login to the OpenShift repository
       
       docker login -u admin -p $(oc whoami -t) docker-registry.default.svc.cluster.local:5000

4. Push the image into the repository

       docker push docker-registry.default.svc.cluster.local:5000/openshift/scap-slave:latest

## Example Usage

1. Create a new project

       oc new-project scap-test

2. Create a *scap-bc.yml* file with the following contents. This pipeline will scan the base RHEL7 docker image from Red Hat and will ask for input if the scan fails.
        
       kind: "BuildConfig"
       apiVersion: "v1"
       metadata:
         name: "scap-pipeline"
       spec:
         strategy:
           jenkinsPipelineStrategy:
             jenkinsfile: |-
               podTemplate(name: 'scap', label: 'scap', cloud: 'openshift', containers: [
                   containerTemplate(
                       name: 'jnlp',
                       image: 'docker-registry.default.svc:5000/openshift/scap-slave',
                       alwaysPullImage: true,
                       args: '${computer.jnlpmac} ${computer.name}',
                       ttyEnabled: false,
                       privileged: true,
                       workingDir: '/tmp/jenkins')],
                   volumes: [
                       hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
                   ]
               ) {
                 node('scap') {
                   stage('SCAP CVE Scan') {
                     SCAN_RESULT = sh([returnStatus: true, script: 'cve-scan         registry.access.redhat.com/rhel7:latest'])
                     archiveArtifacts 'results.html'
                     if (SCAN_RESULT != 0) {
                       timeout(time: 7, unit: 'DAYS') {
                         echo "CVE(s) Detected!"
                         input message: 'One or more CVEs were detected.', submitter: 'admin,dmin-admin'
                       }
                     } else echo "Passed Scan"
                   }
                 }
               }
        


2. Create a pipeline buildconfig from the file:

       oc create -f scap-bc.yml 

3. As a cluster admin, grant the *anyuid* and *privileged* SCCs to the jenkins user:

       oc adm policy add-scc-to-user privileged -z jenkins
       oc adm policy add-scc-to-user anyuid  -z jenkins

4. Update the Jenkins *JAVA_OPTS* variable so that the HTML report is displayed properly in Jenkins

       oc set env dc/jenkins JAVA_OPTS=-Dhudson.model.DirectoryBrowserSupport.CSP=\"\"

5. Visit the jenkins url (*oc get routes*) and click the *scap-test/scap-pipeline* job, select **Build Now**

6. Once it completes, click the *results.html* link under *Last Successful Artifacts* to view the report