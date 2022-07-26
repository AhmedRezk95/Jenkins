FROM jenkins/jenkins:lts-jdk11

# copy the list of plugins we want to install including slack, git , etc
# directory is /usr/share/jenkins/
COPY my-plug.txt /usr/share/jenkins/my-plug.txt

# install the my-plug using jenkins-plugin-cli -f command
RUN jenkins-plugin-cli -f /usr/share/jenkins/my-plug.txt

# VERY IMPORTANT
# setup wizard must be disabled because we are installing it jenkins as code
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false

# copy the config-as-code yaml file into the image
COPY casc.yaml /usr/local/casc.yaml

# VERY IMPORATANT
# set jenkins config-as-code plugin where to find the yaml file
ENV CASC_JENKINS_CONFIG /usr/local/casc.yaml